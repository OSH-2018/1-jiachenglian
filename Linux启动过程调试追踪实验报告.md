

# Linux启动过程调试追踪实验报告

PB16110365 连家诚

## 一.实验目的

熟悉Linux命令行操作，大致了解Linux的启动过程

## 二.实验环境

主机系统：Ubuntu 16.04.4

Linux内核版本：4.15.14

模拟器：qemu 2.5.0

调试器：gdb 7.11.1

## 三.实验内容

用qemu启动Linux内核后用gdb追踪调试，并找出其中至少两个关键事件

## 四.实验过程

### 1.准备工作

#### （1）编译Linux内核代码

在官网上下载`linux-4.15.14.tar.xz`

用`tar zxvf linux-4.15.14.tar.xz`解压

设置内核，`make menuconfig`后，在出现的menu中取消Processor type and features -> Build a relocatable kernel，打开Kernel hacking -> Compile-time checks and compiler options -> Compile the kernel with debug info，完成设置

`make`后报错两次，第一次提示没有安装ncurses，用`sudo apt-get install libncurses5-dev`进行安装；第二次提示没有安装OpenSSL，用`sudo apt-get install libssl-dev`进行安装

再次`make`，结束后`sudo make install`，Linux内核编译完成

#### （2）下载qemu

用`sudo apt-get install qemu`下载qemu 2.5.0

#### （3）制作根文件系统

下载`busybox-1.20.2.tar.bz2`

用`tar -jxvf busybox-1.20.2.tar.bz2`解压

用`make menuconfig`修改配置，选择Busybox Settings -> Build Options -> Build BusyBox as a static binary (no shared libs)

`make`编译后`sudo make install`

#### （4）制作镜像

```shell
#!/bin/bash
sudo rm -rf rootfs
sudo rm -rf tmpfs
sudo rm -rf ramdisk*
sudo mkdir rootfs
sudo cp ../busybox-1.24.2/_install/*  rootfs/ -raf
sudo mkdir -p rootfs/proc/
sudo mkdir -p rootfs/sys/
sudo mkdir -p rootfs/tmp/
sudo mkdir -p rootfs/root/
sudo mkdir -p rootfs/var/
sudo mkdir -p rootfs/mnt/
sudo cp etc rootfs/ -arf
sudo mkdir -p rootfs/lib64
sudo cp -arf /lib/x86_64-linux-gnu/* rootfs/lib64/
sudo rm rootfs/lib/*.a
sudo strip rootfs/lib/*
sudo mkdir -p rootfs/dev/
sudo mknod rootfs/dev/tty1 c 4 1
sudo mknod rootfs/dev/tty2 c 4 2
sudo mknod rootfs/dev/tty3 c 4 3
sudo mknod rootfs/dev/tty4 c 4 4
sudo mknod rootfs/dev/console c 5 1
sudo mknod rootfs/dev/null c 1 3
sudo dd if=/dev/zero of=ramdisk bs=1M count=32
sudo mkfs.ext4 -F ramdisk
sudo mkdir -p tmpfs
sudo mount -t ext4 ramdisk ./tmpfs/  -o loop
sudo cp -raf rootfs/*  tmpfs/
sudo umount tmpfs
sudo gzip --best -c ramdisk > ramdisk.gz
```



### 2.用gdb调试追踪内核

用qemu启动内核，`qemu-system-x86_64 -kernel ~/linux-4.15.14/arch/x86/boot/bzImage -append "root=/dev/ram0 rw rootfstype=ext4 console=ttyS0 init=/linuxrc" -s -S`

此时系统停在起始状态

打开一个新shell，用`gdb`命令启动gdb

```shell
(gdb) file linux-4.15.14/vmlinux					 #在gdb界面加载符号表
Reading symbols from linux-4.15.14/vmlinux...done.     
(gdb) target remote :1234							#建立gdb与gdbserver之间的连接
Remote debugging using :1234
0x0000fff0 in ?? ()
(gdb) break start_kernel							#在start_kernel函数处设断点
Breakpoint 1 at 0xc1c427c6: file init/main.c, line 515.
(gdb) c											   #继续执行
Continuing.

Breakpoint 1, start_kernel () at init/main.c:515
515	{
```



```shell
(gdb) i r											#查看寄存器状态
eax            0xfffffffd	-3
ecx            0x0	0
edx            0x0	0
ebx            0x800	2048
esp            0xc1b65fa0	0xc1b65fa0 <init_thread_union+8096>
ebp            0xc1b65fac	0xc1b65fac <init_thread_union+8108>
esi            0x20800	133120
edi            0x1c9b0a0	29995168
eip            0xc1c427c6	0xc1c427c6 <start_kernel>
eflags         0x200086	[ PF SF ID ]
cs             0x60	96
ss             0x68	104
ds             0x7b	123
es             0x7b	123
fs             0xd8	216
gs             0x0	0
(gdb) list											#查看上下文
510		/* Should be run after espfix64 is set up. */
511		pti_init();
512	}
513	
514	asmlinkage __visible void __init start_kernel(void)
515	{
516		char *command_line;
517		char *after_dashes;
518	
519		set_task_stack_end_magic(&init_task);
```

在跳转到内核入口之前，完成了一系列初始化操作，start_kernel初始化时，调用的第一个函数为`set_task_stack_end_magic`

```shell
(gdb) step											#单步调试
519		set_task_stack_end_magic(&init_task);
(gdb) list											#看上下文
503	
504	void set_task_stack_end_magic(struct task_struct *tsk)
505	{
506		unsigned long *stackend;
507	
508		stackend = end_of_stack(tsk);
509		*stackend = STACK_END_MAGIC;	/* for overflow detection */
510	}
```

`init_task`的pid为0，下面语句创建了0号进程

```c
struct task_struct init_task = INIT_TASK(init_task);
```

`set_task_stack_end_magic`函数的作用是先通过`end_of_stack`函数获取堆栈并赋给 `task_struct`

```shell
(gdb) l
520		smp_setup_processor_id();					   #此函数在x86_64架构下为空
521		debug_objects_early_init();
522	
523		cgroup_init_early();
524	
525		local_irq_disable();
526		early_boot_irqs_disabled = true;
```

接下来调用`boot_cpu_init`函数初始化CPU

```c
(gdb) l
2007	void __init boot_cpu_init(void)
2008	{
2009		int cpu = smp_processor_id();				//获取当前处理器的ID
2010	
2011		/* Mark the boot cpu "present", "online" etc for SMP and UP case */
2012		set_cpu_online(cpu, true);					//将该处理器设为在线
2013		set_cpu_active(cpu, true);
2014		set_cpu_present(cpu, true);
2015		set_cpu_possible(cpu, true);
2016	
2017	#ifdef CONFIG_SMP
2018		__boot_cpu_id = cpu;
2019	#endif
2020	}
```

然后调用`pr_notice`打印

```shell
(gdb) break rest_init									#在rest_init处加断点
Breakpoint 2 at 0xc18d85a0: file init/main.c, line 392.
(gdb) c												   #继续执行
Continuing.

Breakpoint 2, rest_init () at init/main.c:392
392	{
(gdb) l
387	 */
388	
389	static __initdata DECLARE_COMPLETION(kthreadd_done);
390	
391	static noinline void __ref rest_init(void)
392	{
393		struct task_struct *tsk;
394		int pid;
395	
396		rcu_scheduler_starting();						#激活RCU调度程序
402		pid = kernel_thread(kernel_init, NULL, CLONE_FS); #创建kernel_init内核线程，即1号进程	

408		rcu_read_lock();								#标记读端临界区起始位置
409		tsk = find_task_by_pid_ns(pid, &init_pid_ns);
410		set_cpus_allowed_ptr(tsk, cpumask_of(smp_processor_id()));
411		rcu_read_unlock();								#标记读端临界区终止位置
```

执行完`cpu_startup_entry(CPUHP_ONLINE);`后rest_init结束，start_kernel也结束，此时kernel初始化完成。

## 五.实验结果

完成了追踪调试的任务，找到两个关键函数start_kernel和rest_init