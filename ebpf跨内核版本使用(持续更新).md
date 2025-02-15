# 前言
最近在做`ebpf`相关的东西，然后有个最大的需求就是能够跨大版本使用，大到什么程度呢？
* `3.10` - `5.12`


这就需要在开发的时候就定好写法和技术栈之类的问题


# 技术栈
为了解决依赖问题，同时为了开发方便，最终选择使用的开发框架是
* [`cilium/ebpf`](https://github.com/cilium/ebpf)


这个框架的大概逻辑就是
1. 用`C`写`ebpf`内核侧的代码
2. 利用`bpf2go`将C代码编译成`*.o`文件同时生成了中间加载文件
3. 最终将`*.o`打包封装到`golang`编译的程序中去，不依赖任何动态链接库


简单来说`golang`负责了`ebpf`的加载和用户态处理，而`c`则负责了内核态的逻辑，这样的好处也很明显：
1. 只需要解决`C`的部分跨版本问题，并且可以自由选择开发方式
2. 纯`golang`程序解决了动态链接库的依赖，且用户态逻辑开发方便


`C`的部分需要考虑到`CO-RE`，因此使用了`bpf_core_read.h`来进行数据读取


# 问题记录
`ebpf`是一项和内核版本强相关的技术，因此绝大部分都是因为内核版本延伸的问题


## 内核支持
`ebpf`实际引入内核是`3.18`的时候，但是因为能力过于出众，在2019年`redhat`又将`ebpf`能力重新以`lkm`的方式移植到了`readhat 7/centos 7`上，但是注意的是并非全面支持，而是只移植了`probe`，`tracepoint`和`perv events`这几个能力，其余的例如`socket filters`和`xdp`等功能都是不支持的
```
eBPF in Red Hat Enterprise Linux 7.6 is enabled only for tracing purposes, which allows attaching eBPF programs to probes, tracepoints and perf events. Other use cases such as eBPF socket filters or eXpress DataPath (XDP) are not enabled at this stage.
```
因此实际上在`centos7`上我们也是可以使用`ebpf`的


## 内核信息读取
谨记一点，就是`ebpf`虽然具备`内核数据读取`的能力，但是所有的读取行为都需要经过`bpf`本身的支持函数来实现，因此在原生的`ebpf`中，是无法通过`指针`直接从内核中获取数据的


以读取进程信息为例子，在多数情况下，`ebpf`获取到的进程信息都是一个`指针`
```
struct task_struct *task = ctx->task;
```
在内核中`pid`信息是作为一个`pid_t`结构保存在`task_struct`中的
```
struct task_struct {
    ...
    pid_t pid;
    ...
}
```
然而在`ebpf`中是无法直接通过`pid_t p = task->pid`获取到`pid`的信息的，而是需要通过`bpf_helper`的函数来实现，这些函数都有安全检查来检测要访问的数据是否是在`当前可访问`的区域之类
> 当前可访问实际上就是禁用了缺页中断，同时有access_ok的判断，本质上还是存在内核任意地址读取的问题，例如在`/proc/kallsyms`中获取到的地址，就可以直接在`ebpf`中读取


通常来说有这几个函数能够实现我们的目的：
* `bpf_probe_read()`/`bpf_probe_read_str()` // 高版本以后不再推荐使用，因为没有区分内存空间
* `bpf_probe_read_kernel()`/`bpf_probe_read_kernel_str()`
* `bpf_probe_read_user()`/`bpf_probe_read_user_str()`
* `bpf_core_read()`/`bpf_core_read_str()`
* `BPF_CORE_READ()`
* `BPF_CORE_READ_INTO()`/`BPF_CORE_READ_STR_INTO`


在此明确一点，以上所有的函数逻辑都是从指定的地址上获取到数据，然后再存入到栈上的地址，操作的都是地址，因此要明确传入其中的数据都是一个数据地址才行


在高版本的编译器或者`BCC`框架中经常存在直接`pid_t p = task->pid`这样的引用的写法，这是因为框架/编译器本身做了`rewrite`，在编译的时候依然会翻译成多次的`bpf_probe_read/bpf_probe_read_kernel`的操作，而在实际的开发中还是推荐显式调用`bpf_read_*`，这样不容易出现问题


`bpf_*_str()`类型的函数主要用于读取内核中以NULL结尾的字符串信息，并且返回字符串长度，如果目标地址的字符串长度超过了要存入的地址长度，则会主动在`size - 1`的地方设置为NULL，这类函数在读取内核中的`char *`十分有效


`BPF_CORE_READ*`这种大写的函数主要作用是简化了针对复杂引用的处理，例如`t->mm->exe_file->fpath.dentry->d_name.name;`这样的多层引用，如果用`bpf_probe_read/bpf_probe_read_kernel/bpf_core_read`来读取的话，需要显式读取多次
```
struct task_struct *t = ...;
struct mm_struct *mm;
struct file *exe_file;
struct dentry *dentry;
const char *name;
bpf_core_read(&mm, 8, &t->mm);
bpf_core_read(&exe_file, 8, &mm->exe_file);
bpf_core_read(&dentry, 8, &exe_file->path.dentry);
bpf_core_read(&name, 8, &dentry->d_name.name);
```
而可以用`BPF_CORE_READ`简化写法
```
struct task_struct *t = ...;
const char *name;
name = BPF_CORE_READ(t, mm, exe_file, fpath.dentry, d_name.name);
```
这种写法十分高效快捷，但是相比于`bpf_core_read`直接针对内存地址做读取更依赖于结构体的数据偏移，因此需要注意内核版本变动带来的成员变量名变化的问题，而且此函数不会返回错误，即使中间某个指针出现问题，也只会最后返回`0/NULL`


`BPF_CORE_READ_INTO/BPF_CORE_READ_STR_INTO`，在解决了多次引用的问题的同时，还解决了`c`的返回值不能是一个数组的问题，对于多次引用后需要进行`* -> []`的读取场景十分好用


然而这儿也有内核问题，当内核版本低于`5.5`，就只有`bpf_probe_read`/`bpf_probe_read_str`这两种可用，而到了`5.5`以上以后内核粗暴的将`bpf_probe_read`取消了，而是改成了`bpf_probe_read_compat`，不过实际编译以后，发现都是使用的调用号来区分实际的调用，因此应该在低版本上也是通用的


然而更可惜的一点是，虽然`3.10`的内核移植了`eBPF`功能，然而却没有支持`BTF`，因此`CO-RE`的写法无法使用，此功能正常情况下直到`5.2`才支持
```
2023/07/31 15:35:10 [2023-07-31 15:35:10] field DumpTask: program dump_task: apply CO-RE relocations: load kernel spec: no BTF found for kernel version 3.10.0-1127.19.1.el7.x86_64: not supported
```


## 循环
`ebpf`的循环问题在`5.3`以前的`vertify`中是被限制的，因此只能使用`clang`本身的能力进行`有限循环展开`，在通用的`cpu`上一般是`8-16`次左右就会达到上限
```
#pragma clang loop unroll(full)
        for (i = 0; i < 4; i++) {
            /* Do stuff ... */
        }
```
`5.3`以后添加了`Bounded loops`的能力，但是适用对象主要还是针对`map`进行遍历，`5.17`增加了[`bpf_loop`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?h=e6f2dd0f80674e9d5960337b3e9c2a242441b326)能力，针对函数进行多次回调实现循环能力
```
long bpf_loop(u32 nr_loops, void *callback_fn, void *callback_ctx, u64 flags)
```
为了向低版本适配，最终`clang`的`有限循环展开`无疑是最好的选择，这个能力在获取`task_struct`的完整`cwd`上十分重要


## 用户态交互
`BPF_FUNC_seq_printf`到了`5.7`才支持，因此低版本无法使用


## 数值边界
经常出现写了半天的代码然后`verify`给你报错的情况，就比如我之前遇到的一次一直没有解决的问题[Clang compiles ebpf skips the judgment causing the program to fail to load](https://github.com/llvm/llvm-project/issues/62849)，这个我一直怀疑是`llvm`本身的问题


再比如另一个问题：
```
        unsigned long arg_start = (unsigned long)BPF_CORE_READ(task, mm, arg_start);
        unsigned long arg_end = (unsigned long)BPF_CORE_READ(task, mm, arg_end);
        unsigned int arg_len = arg_end - arg_start;
        bpf_copy_from_user_task(&info.argv, arg_len, (char *)arg_start, task, 0);
```
在经过`vertify`校验过程中`arg_len`的预测最小值却是一个负数，即使是已经设置了`unsigned int`，但是依然会被预测成是一个负数，这个可能和C的变量初始化有关吧，所对于一个没有显式初始化的变量，设置了`unsigned`大概也是无意义的，所以修改成如下就可以正确加载了
```
        unsigned long arg_start = (unsigned long)BPF_CORE_READ(task, mm, arg_start);
        unsigned long arg_end = (unsigned long)BPF_CORE_READ(task, mm, arg_end);
        unsigned int arg_len = 0; 
        arg_len = arg_end - arg_start;
        bpf_copy_from_user_task(&info.argv, arg_len, (char *)arg_start, task, 0);
```
> 据查这儿进行一次和`0xFFFFFFFF`的逻辑运算也可以，没试过


## 获取存量信息
从`5.8`版本开始，`ebpf`加入了迭代器的功能，能够将当前的数据视图一次性迭代查询出来，例如`iter/task`可以遍历内核中所有的进程信息，但是这儿要注意一个问题，我们是通过迭代的方式提取出内核中所有的相关数据，此刻的`cpu`中没有上下文信息的，因此当所取数据是在`用户态地址`的话，需要其余方式获取


`5.18`以后增加了`bpf_copy_from_user_task()`能力，可以直接从指定的`task`中获取到`虚拟地址`的数据，但是再次注意，此函数因为耗时会导致进程进入睡眠状态，因此需要将当前`ebpf`设置成可休眠的，这个能力需要`5.10`版本的内核


## `MAP`写法
忘了具体原因了，但是`map`的定义方式会影响带程序的编译结果
```
#if LINUX_VERSION_CODE < KERNEL_VERSION(5, 3, 0)
    struct bpf_map_def SEC("maps") MAP_NAME = {
        .type = BPF_MAP_TYPE_PERF_EVENT_ARRAY,
        .key_size = sizeof(int),
        .value_size = sizeof(u32),
    };
#else
    struct {
        __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
        __uint(key_size, sizeof(int));
        __uint(value_size, sizeof(u32));
    } egressmap SEC(".maps");
#endif
```
在常见情况下有如上两种定义方式，第一种方式会在`elf`中生成一个`map section`，而第二种则不会，因为是`BTF`写法，完全依靠`BTF map`，因此在某些情况下一个不需要`CO-RE`的程序会因为这个问题无法加载，即使是`CO-RE`，在低版本例如`3.10`版本中，也会爆出`map create: invalid argument (without BTF k/v)`的问题，因此建议都使用第一种写法来实现普通`map`


## 跨版本支持
`vmlinux.h`解决的问题是对于内核头文件的依赖，只能算是`CO-RE`的一部分而不是全部，实际上程序是否能正确跑起来是依赖于`BTF`提供的数据结构的偏移信息
`cilium/ebpf`对于`BTF`信息的寻找尊崇`libbpf`的逻辑，即优先加载`/sys/kernel/btf/vmlinux`，如果没有的话则寻找`vmlinux`的`elf`文件再从其中导出`BTF`信息
```
// btf.go
func loadKernelSpec() (_ *Spec, fallback bool, _ error) {
    fh, err := os.Open("/sys/kernel/btf/vmlinux")
    ......
    file, err := findVMLinux()
    spec, err := loadSpecFromELF(file)
    ......
}
func findVMLinux() (*internal.SafeELFFile, error) {
    ......
    locations := []string{
        "/boot/vmlinux-%s",
        "/lib/modules/%s/vmlinux-%[1]s",
        "/lib/modules/%s/build/vmlinux",
        "/usr/lib/modules/%s/kernel/vmlinux",
        "/usr/lib/debug/boot/vmlinux-%s",
        "/usr/lib/debug/boot/vmlinux-%s.debug",
        "/usr/lib/debug/lib/modules/%s/vmlinux",
    }
    ......
}
```
因此只要能够提供出对应内核版本的`BTF`并加载，就可以实现低版本内核的`CO-RE`，此能力可以借助[BTFHub](https://github.com/aquasecurity/btfhub)实现，需要在加载`ebpf prog`的时候指定`BTF`


`btf`解决的是不同内核版本的结构体成员偏移的问题，但是如果遇到成员缺失或者结构体变动，依然需要手动判断设置
```
#include <bpf/bpf_helpers.h>
extern int LINUX_KERNEL_VERSION __kconfig;
...
if (LINUX_KERNEL_VERSION > KERNEL_VERSION(5, 15, 0)) {
    /* we are on v5.15+ */
}
```
但是这时候又会因为引用的`vmlinux.h`在编译时候报错，所以需要再配合`struct flavor`来实现结构体的重定义，以官方文档来举例
```
/* latest kernel task_struct definition, which can also come from vmlinux.h */
struct task_struct {
    int __state;
} __attribute__((preserve_access_index));
struct task_struct___old {
    long state;
} __attribute__((preserve_access_index));
...
struct task_struct *t = (void *)bpf_get_current_task();
int state;
if (bpf_core_field_exists(t->__state)) {
    state = BPF_CORE_READ(t, __state);
} else {
    /* recast pointer to capture task_struct___old type for compiler */
    struct task_struct___old *t_old = (void *)t;
    /* now use old "state" name of the field */
    state = BPF_CORE_READ(t_old, state);
}
...
```
## `read/write only map`
```
resolve .kconfig: creating map: map create: read- and write-only maps not supported (requires >= v5.2)
```
代码里使用`extern int LINUX_KERNEL_VERSION __kconfig;`直接导致编译出来的`ebpf`无法在低版本的内核上使用了，原因是我的`cilium/ebpf`在加载程序的时候，会将`.kconfig`设置成`unix.BPF_F_RDONLY_PROG | unix.BPF_F_MMAPABLE`，然而`BPF_F_MMAPABLE`能力是到了`5.5`才被支持到的，至于`BPF_F_RDONLY`是[[PATCH bpf-next v6 03/16] bpf: add program side {rd, wr}only support for maps](https://lore.kernel.org/bpf/20190409210910.32048-4-daniel@iogearbox.net/)之后被支持的，依然是低版本不可用


在`cilium/ebpf`中`__kconfig`的能力是[Reference kconfig variables in bpf using __kconfig#698](https://github.com/cilium/ebpf/issues/698)这个`issue`中引入的，自动处理了`__kconfig`和`LINUX_KERNEL_CODE`，所以在用户态目前没办法绕过这个问题，起码我不知道


但是仔细看一下逻辑，就是创建了一个`.kconfig`的虚拟`section`并且和`LINUX_KERNEL_VERSION`关联上，然后在加载的时候提前去修改了这个变量，既然如此的话那就可以自己想办法去实现这个逻辑，就是借助了`RewriteConstants`的方式，在加载时重写变量
```
// in *.c
volatile const uint32_t arg;


// in *.go
err := spec.RewriteConstants(map[string]interface{}{
    "arg":  uint32(1),
})
```
然而这还是一个问题，就是这个变量会被放置在`.rodata`，而这个`section`会被设置成`unix.BPF_F_RDONLY_PROG`，那就还是老问题了，最后只能通过新建一个`map`，然后在加载时写入`version`的方式来解决这个问题


然后在C代码中硬编码的字符串也会导致`.rodata`，例如：
```
char *block = "block\x00";
```
最终编译出来的`ebpf`文件的`section`如下
```
4 .rodata       00000007  0000000000000000  0000000000000000  000005e8  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
```


## 协议栈大小
`ebpf`程序的协议栈好像只有512字节，所以不能在栈上存放太多信息，但是如果想要把众多信息传入到用户态中，可以使用`map`的方式实现，在程序启动时候就新建一个`map`出来，然后再从`map`中获取到一块区域，这时候直接写入数据即可


## 指针问题
这是一个巨大的坑点，就是所有的指针最好都在初始化的时候就设置成空指针，然后之前获取过的指针，如果要重新赋值的话就重新设置指针，简单来说就是把指针当做是一次性用品，用过了就重新设置！


## 字符串比对
这其实是一个经常需要用到的东西，但是直接使用是无法通过检测的，不过可以曲线救国，通过明确的循环来挨个校验字符串
```
    char comm_str[TASK_COMM_LEN] = {0};
    char comm[] = "bash";
    bpf_get_current_comm(&comm_str, sizeof(comm_str));


    int len = 0;
#pragma clang loop unroll(full)
    for (int i = 0; i < 4; i++) {
        if (comm_str[i] == comm[i]) {
            len ++;
        }
    }


    if (len == 4)
        return -1;
```


# 参考资料
* [Introduction to eBPF in Red Hat Enterprise Linux 7](https://www.redhat.com/en/blog/introduction-ebpf-red-hat-enterprise-linux-7)
* [Are loops allowed in Linux's BPF programs?](https://yanhang.me/post/2021-ebpf-loop/)
* [TaskIterator.cc](https://github.com/iovisor/bcc/blob/18b00a903955950f796db85d1f24815f7a6f2177/examples/cpp/TaskIterator.cc)
* [BPF CO-RE reference guide](https://nakryiko.com/posts/bpf-core-reference-guide/)
* [BPF 迭代器：以灵活和高效的方式检索内](https://www.ebpf.top/post/bpf-iterator-retrieving-kernel-data-with-flexibility-and-efficiency/)
* [Sleepable BPF programs](https://lwn.net/Articles/825415/)
* [How to make eBPF program sleepable](https://stackoverflow.com/questions/75869746/how-to-make-ebpf-program-sleepable)
* [Is it possible to use CO-RE with kernel 4.9? #1033](https://github.com/cilium/ebpf/discussions/1033)
* [Why does my eBPF program contain an .rodata map? #592](https://github.com/cilium/ebpf/discussions/592)
* [BPF 进阶笔记（四）：调试 BPF 程序](http://arthurchiao.art/blog/bpf-advanced-notes-4-zh/)
* [How do I rewrite global variables before loading a BPF object?#795](https://github.com/cilium/ebpf/discussions/795)