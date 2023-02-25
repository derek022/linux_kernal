syscall_

do_xxx    基本都是中断调用的函数

# 进程的销毁流程

1.1 exit是销毁函数      一个系统调用  do_exit

​		首先改函数会释放进程的代码段和数据段占用的内存   



1.2 关闭进程打开的所有文件，对当前的目录和节点进行同步（文件操作）

1.3 如果当前要销毁的进程有子进程，那么就让1号进程作为新的父进程（init进程）

1.4 如果当前进程是另一个会话头进程，则会终止会话中的所有进程

1.5 改变当前进程的运行状态，变成TASK_ZOMBIE僵死状态，并且向其父进程发送SIGCHLD信号

1.6 重新调度



2.1 父进程在运行子进程的手机号 一般都会运行wait waitpid这两个函数（父进程等待某个子进程终止的）

​    当父进程收到SIGCHLD信号时父进程会终止僵死状态的子进程

2.2 首先父进程会吧子进程的运行时间累加到自己的进程变量中

2.3 把对应的子进程的进程描述结构体进行释放，置空任务数组中的空槽



exit.c文件

```c
void release(struct task_struct* p);
```

完成清空了任务描述表中的对应进程表项，释放对应的内存叶（代码段和数据段）



```c
static inline int send_sig(long sig,struct task_struct * p,int priv)
```

给指定的p进程发送对应的sig信号



```c
static void kill_session(void)
```

终止会话，终止当前进程的会话，给其发送SIGHUP信号



```c
int sys_kill(int pid,int sig)
```

kill - 不是杀死的意思，向对应的进程号或者进程组号发送任何信号

​	pid  (

​				pid>0   给进程的pid的发送sig

​				pid=0   给当前进程的进程组发送sig

​				pid=-1  给所有进程发送sig

​				pid<-1  给进程组号为-pid的进程组发送信号

​	)



```c
static void tell_father(int pid)
```

子进程向父进程发送SIGCHLD信号



```c
int do_exit(long code)
```

current->pid就是当前关闭的进程



