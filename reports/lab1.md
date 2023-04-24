# lab 1
## 编程作业
**实现的功能**：查询当前正在执行的任务信息，任务信息包括任务控制块相关信息（任务状态）、任务使用的系统调用及调用次数、任务总运行时长（单位ms）。

具体是通过在`TaskControlBlock`结构体中增加了[`syscall_times: [u32; MAX_SYSCALL_NUM]`和`time: usize`](../os/src/task/task.rs)两项实现的。在任务初次调用时，在[`run_next_task`函数](../os/src/task/mod.rs)中启动计时，在[`sys_task_info`函数](../os/src/syscall/process.rs)中计算时长。
对系统调用类型的统计是在通过在[`trap_handler`函数](../os/src/trap/mod.rs)中捕捉`Exception::UserEnvCall`类的Trap并进行统计。

在进行BASE=2测试时发现无法通过，qemu会出现卡死的问题，最后通过修改[entry.asm](../os/src/entry.asm)中的`boot_stack_lower_bound`，将其`.space`从`4096 * 16`改到`4096 * 64`，通过测试。

在[h3_taskinfo.rs](../user/src/bin/ch3_taskinfo.rs)文件中有这样一个问题：
>想想为什么 write 调用是两次？
>>答：在使用`println`的时候，调用了`/user/src/console.rs`中的`print`函数，`print`函数调用了`ConsoleBuffer`所拥有的`Write trait`中的`write_fmt`方法，`write_fmt`又调用`core::fmt`中的`write`函数，`write`函数又调用了`ConsoleBuffer`实现的`Write trait`中的`write_str`方法，当`write_str`方法被调用时，如果缓冲区已满**或**遇到换行符，则会调用`flush`方法将缓冲区中的数据写入标准输出():
>>```
>>if (*c == b'\n' || self.0.len() == CONSOLE_BUFFER_SIZE) && -1 == self.flush() {...}
>>```
>>这里出现2次的原因是因为`println!`自带一个`\n`，同时字符串结尾又有一个`\n`，所以 write 调用是两次。
    
>虽然系统调用接口采用桶计数，但是内核采用相同的方法进行维护会遇到什么问题？是不是可以用其他结构计数？
>>如果内核采用相同的桶计数方法进行维护，可能会导致竞争条件和性能瓶颈的问题。在多核 CPU 上，多个进程或线程同时调用系统调用时，可能会导致对桶计数器进行竞争，从而可能成为性能瓶颈。
>>此外，不同系统调用的使用频率不一致，桶计数可能会出现有的系统调用的桶未被使用过而有的系统调用的桶已经溢出，且对桶进行扩容存在很大的时间开销，并进一步放大利用率低的问题。
>>桶计数可以看作是一种静态哈希，内核中可以考虑使用动态哈希，能够实现对桶的数量的动态调整，减少空间浪费。

## 简答作业
1. 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。 请同学们可以自行测试这些内容 (运行 Rust 三个 bad 测例 (ch2b_bad_*.rs) ， 注意在编译时至少需要指定 `LOG=ERROR` 才能观察到内核的报错信息) ， 描述程序出错行为，同时注意注明你使用的 sbi 及其版本。

    RustSBI 版本为 `0.3.0-alpha.2`。

    报错信息如下所示：
    ```
    [kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x80400414, kernel killed it.
    [kernel] IllegalInstruction in application, kernel killed it.
    [kernel] IllegalInstruction in application, kernel killed it.
    ```
    [`bad_address`](../user/src/bin/ch2b_bad_address.rs)程序首先通过一个裸指针将内存地址 0x0 强制转换为一个可变的 u8 类型的指针，并使用 write_volatile 函数在该地址写入了一个值为 0 的字节。由于内存地址 0x0 是一个无效的地址，因此这个操作会触发一个StorePageFault。

    [`bad_instructions`](../user/src/bin/ch2b_bad_instructions.rs)执行了sret指令，是一个s特权指令，只能在内核态下执行，这个指令作用是将处理器的状态恢复为中断前的状态，并从内核态返回到用户态。

    [`bad_register`](../user/src/bin/ch2b_bad_register.rs)执行了cssr指令，是一个s特权指令，只能在内核态下执行，这个指令作用是来读取处理器的状态寄存器 sstatus 的值，并将其存储到 sstatus 变量中去。

2. 深入理解 trap.S 中两个函数 `__alltraps` 和 `__restore` 的作用，并回答如下问题:
    1. L40：刚进入 `__restore` 时，`a0` 代表了什么值。请指出 `__restore` 的两种使用情景。

        调用__alltraps后进入trap_handler，trap_handler中a0为当前内陷进程的TaskContext地址，由__alltraps通过 mv a0, sp 传递而来，如果碰到中断/系统调用，trap_handler执行完成后再将a0原封不动地返回给__restore，如果碰到异常，则调用__switch对任务进行切换；调用__switch之后进入__restore，此时a0保存着上一个任务的TaskContext的地址。
    2. L43-L48：这几行汇编代码特殊处理了哪些寄存器？这些寄存器的的值对于进入用户态有何意义？请分别解释。
        ```
        ld t0, 32*8(sp)
        ld t1, 33*8(sp)
        ld t2, 2*8(sp)
        csrw sstatus, t0
        csrw sepc, t1
        csrw sscratch, t2
        ```
        借助t0、t1和t2寄存器，恢复了进程的sstatus、sepc和sscratch寄存器。

        sstatus寄存器保存进程的中断使能、特权级等信息，以将进程的状态从中断/异常处理的状态恢复到进程原来的状态。

        sepc寄存器保存进程在被中断/异常打断时的指令地址，以便进程能够从之前被打断的位置继续执行。

        sscratch寄存器保存S模式上下文的指针，以便重新进入内核态时，可以通过将sscratch寄存器与sp寄存器交换来快速恢复S模式上下文。

3. L50-L56：为何跳过了 `x2` 和 `x4`？
    ```
    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
    LOAD_GP %n
    .set n, n+1
    .endr
    ```
    x4又名tp，没有用到；x2又名sp，如果要ld x2, 2*8(sp)，则会把当前的sp破坏掉。
4. L60：该指令之后，`sp` 和 `sscratch` 中的值分别有什么意义？
    ```
    csrrw sp, sscratch, sp
    ```
    当前 sp 指向 user stack，sscratch 指向 kernel stack。
5. `__restore`：中发生状态切换在哪一条指令？为何该指令执行之后会进入用户态？

    sret，因为sstatus中的SPP等字段被修改为用户态。
6. L13：该指令之后，`sp` 和 `sscratch` 中的值分别有什么意义？
    ```
    csrrw sp, sscratch, sp
    ```
    当前 sp 指向 kernel stack，sscratch 指向 user stack。
7. 从 U 态进入 S 态是哪一条指令发生的？

    ecall。

## 荣誉准则
1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与以下各位就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：
无

2. 此外，我也参考了以下资料，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：
https://five-embeddev.com/quickref/instructions.html#sec_supervisor_sstatus

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。