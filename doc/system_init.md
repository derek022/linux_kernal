## 1、操作系统的引导过程和启动程序

BIOS		BOOTLOADER

1. Linux操作系统的引导

   1.1 Linux 是如何从硬盘中读出的

   1.2 Linux 在启动的时候是如何拿到硬件参数的

   1.3 Linux 在初始运行中都做了什么

​			traps.c 		trap_init()

​			mktime.c	time_init()

​			sched.c		sched_init()



​			由PC机的BIOS（0xFFFF0是BIOS存储的总线地址）把bootsect从某个固定的地址拿到了内存中的某个固定地址（0x90000），并且进行了一系列的硬件初始化和参数设置



bootsect.s 文件

​	磁盘引导块程序，在磁盘的第一个扇区的程序（0磁道，0磁头，1扇区(固态硬盘)）

​	作用：首先将后续的setup.s代码从磁盘中加载到紧接着bootsect.s的地方

​				在显示屏上显示loading system再将system(操作系统)模块加载到0x10000的地方

​				最后跳转到setup.s中去运行

```asm
SYSSIZE = 0x3000
...
SETUPLEN = 4				! nr of setup-sectors
BOOTSEG  = 0x07c0			! original address of boot-sector
INITSEG  = 0x9000			! we move boot here - out of the way
SETUPSEG = 0x9020			! setup starts here
SYSSEG   = 0x1000			! system loaded at 0x10000 (65536).
ENDSEG   = SYSSEG + SYSSIZE		! where to stop loading
```

setup.s

​	解析BIOS/BOOTLOADER传递过来的参数

​	设置系统内核运行的LDT（局部描述符） IDT（中断描述符寄存器） 全局描述符(设置全局描述符寄存器)

​	设置中断控制芯片，进入保护模式运行（svc32保护模式  设置寄存器中的值）

​	跳转到system模块的最前面的代码运行（head.s）



```asm
mov	[0],dx		! 
mov	[2],ax		! 扩展内存大小
mov	[4],bx		! 显存大小和信息
mov	[6],ax		
mov	[8],ax
mov	[10],bx
mov	[12],cx
!2个硬盘参数表
!根文件系统
```



head.s

​	加载内核运行时的各数据段寄存器，重新设置中断描述符表

​	开启内核正常运行时的协处理器等资源

​	设置内核管理的分页机制

​	跳转到main.c开始运行



main.c 

```c
void main(void*);
```



```c
// 设置操作系统的根文件
ROOT_DEV = ORIG_ROOT_DEV;
//设置操作系统驱动参数
drive_info = DRIVE_INFO;
// 解析setup.s代码后获取系统内存参数
// 设置系统的内存大小，系统本身内存（1MB）+扩展内存大小（参数*KB）
memory_end = (1<<20) + (EXT_MEM_K<<10);
// 取整4K的内存大小
memory_end &= 0xfffff000;
// 控制操作系统的最大内存为16MB
if (memory_end > 16*1024*1024)
	memory_end = 16*1024*1024;
// 设置高速缓冲区的大小
if (memory_end > 12*1024*1024) 
	buffer_memory_end = 4*1024*1024;
else if (memory_end > 6*1024*1024)
	buffer_memory_end = 2*1024*1024;
else
	buffer_memory_end = 1*1024*1024;
// 主内存的开始，就是虚拟内存的结束
main_memory_start = buffer_memory_end;
// 如果有虚拟磁盘，主内存开始就再加上虚拟磁盘的偏移
#ifdef RAMDISK  
	main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
#endif
```





## 2、操作系统启动初始化程序 Init

1. 初始化代码

   ​	起点：磁盘引导程序，需要将内核等移入内存进行运行，并初始化多种模块和硬件

   ​	终点；运行第一个应用程序 系统的根文件系统



```c
void init(void)
{	
	int pid,i;

	setup((void *) &drive_info);
	(void) open("/dev/tty0",O_RDWR,0); // 打开标准输入控制台
	(void) dup(0);	// 打开标准输出控制台  通过复制
	(void) dup(0);	// 打开标准错误控制台  通过复制
	printf("%d buffers = %d bytes buffer space\n\r",NR_BUFFERS,
		NR_BUFFERS*BLOCK_SIZE);
	printf("Free mem: %d bytes\n\r",memory_end-main_memory_start);
    
    // 创建 了 1号进程 如果在0号父进程创建进程成功则 fork函数返回0，
    // 如果在子进程中fork则返回父进程PID。
    // 如果fork返回值为0，则在新进程中执行，
    // 如果返回值不为零，则在父进程中执行
	if (!(pid=fork())) {
        // 关闭0号进程创建的标准输入输出
		close(0);
		if (open("/etc/rc",O_RDONLY,0))
			_exit(1);
        // 挂接文件系统，执行shell程序
		execve("/bin/sh",argv_rc,envp_rc);
		_exit(2);
	}
    
    // 在0号进程中 等待子进程退出
    if (pid>0)
		while (pid != wait(&i))
            /* nothing */;
    
    
    while (1) {
		if ((pid=fork())<0) {
			printf("Fork failed in init\r\n");
			continue;
		}
		if (!pid) { // 在新创建的子进程中执行
			close(0);close(1);close(2);
			setsid();
			(void) open("/dev/tty0",O_RDWR,0);
			(void) dup(0);
			(void) dup(0);
			_exit(execve("/bin/sh",argv,envp));
		}
		while (1)
			if (pid == wait(&i))
				break;
		printf("\n\rchild %d died with code %04x\n\r",pid,i);
		sync();
	}
	_exit(0);
```

