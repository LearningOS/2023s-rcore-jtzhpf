# lab 1
## 编程作业
**实现的功能**：查询当前正在执行的任务信息，任务信息包括任务控制块相关信息（任务状态）、任务使用的系统调用及调用次数、任务总运行时长（单位ms）。

具体是通过在`TaskControlBlock`结构体中增加了[`syscall_times: [u32; MAX_SYSCALL_NUM]`和`time: usize`](../os/src/task/task.rs)两项实现的。在任务初次调用时，在[`run_next_task`函数](../os/src/task/mod.rs)中启动计时，在[`sys_task_info`函数](../os/src/syscall/process.rs)中计算时长。
对系统调用类型的统计是在通过在[`trap_handler`函数](../os/src/trap/mod.rs)中捕捉`Exception::UserEnvCall`类的Trap并进行统计。

在[h3_taskinfo.rs](../user/src/bin/ch3_taskinfo.rs)文件中有这样一个问题：
>想想为什么 write 调用是两次？
>>答：在使用`println`的时候，调用了`/user/src/console.rs`中的`print`函数，`print`函数调用了`ConsoleBuffer`所拥有的`Write trait`中的`write_fmt`方法，`write_fmt`又调用`core::fmt`中的`write`函数，`write`函数又调用了`ConsoleBuffer`实现的`Write trait`中的`write_str`方法，当`write_str`方法被调用时，如果缓冲区已满**或**遇到换行符，则会调用`flush`方法将缓冲区中的数据写入标准输出():
>>```
>>if (*c == b'\n' || self.0.len() == CONSOLE_BUFFER_SIZE) && -1 == self.flush() {...}
>>```
>>这里出现2次的原因是因为`println!`自带一个`\n`，同时字符串结尾又有一个`\n`，所以 write 调用是两次。
    
>虽然系统调用接口采用桶计数，但是内核采用相同的方法进行维护会遇到什么问题？是不是可以用其他结构计数？
>>桶的体积太大，且
## 简答作业
