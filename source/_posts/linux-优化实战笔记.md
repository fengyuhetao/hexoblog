---
title: linux-优化实战笔记
tags: linux
abbrlink: 46756
date: 2018-12-04 10:24:03
---

​	这几年，写的代码基本都是本地跑，也没碰到什么性能问题。不过，实习的时候，网站三天两头就崩溃，看代码也似乎没啥问题，现在想想，我的解决方案无非重启，继续运行，崩溃了，继续重启，要是虚拟机有毛病，直接重装虚拟机，传说中的3R工程师（replace, reinstall, restart）。。。。。当时，这些问题基本都是技术老大处理，我也接触不到，怎么搞定的一无所知。

​	希望《linux性能优化》能给我带来一些新的知识面。

## 平均负载

### 概念

平均负载是指单位时间内，系统处于**可运行状态**和**不可中断状态**的平均进程数，也就是平均活跃进程数，和CPU使用率无直接关系。

* 可运行状态 指正在使用 CPU 或者正在等待 CPU 的进程，也就是我们ps命令中处于R（Runnning）状态的进程

* 不可中断状态 正处于内核态关键流程中的进程，并且这些流程是不可打断的，比如等待硬件设备的I/O相应，也就是ps命令中的D（Uninterruptible Sleep也叫做Disk Sleep）状态

### 查看

理想状态: 每个cpu运行一个进程。

主要就是top和uptime命令:

```
tiandiwuji@tiandiwuji:~/Desktop/linux_preformance$ uptime
 22:01:35 up 1 day, 19:23,  1 user,  load average: 0.00, 0.02, 0.00
```

load average: 分别是1分钟，5分钟，15分钟。考虑性能问题，应根据趋势来判断。

生产环境中，当平均负载高于CPU数量70%（不绝对），就需要开始关注分析。

最推荐的方法，还是把系统的平均负载监控起来，然后根据更多的历史数据，判断负载的变化趋势。

注意：平局负载只能表现出性能出现问题，具体哪出现问题无法提供。

### 平均负载与CPU使用率

CPU 使用率，是单位时间内 CPU 繁忙情况的统计，跟平均负载不完全对应。

* CPU 密集型进程，使用大量 CPU 会导致平均负载升高         -->  一致
* I/O 密集型进程，等待 I/O 也会导致平均负载升高，但 CPU使用率不一定高。       --> 不一致
* 大量等待 CPU 的进程调度也会导致平均负载升高，此时的 CPU使用率也会偏高。      --> 不一致

### 案例分析

使用工具: stress和sysstat

stree: 系统压力测试工具

sysstat:Linux性能工具。

* mpstat 常用多核CPU性能分析工具，实时查看每个CPU的性能指标以及所有CPU的平均指标
* pidstat 常用进程性能分析工具，用来实时查看进程的CPU、内存、I/O以及上下文切换等性能指标

过程:

略

## CPU上下文切换

### 概念

#### CPU上下文:

CPU 都需要知道任务从哪里加载、又从哪里开始运行，也就是**CPU寄存器和程序计数器(PC)**,CPU 寄存器，是 CPU 内置的容量小、但速度极快的内存。而程序计数器则是用来存储 CPU 正在执行的指令位置、或者即将执行的下一一条指令位置。它们都是 CPU 在运行任何任务前，必须的依赖环境，因此也被叫做 CPU 上下文。

**Note: x86 系统中自增的是 IP，用 CS:IP 组合表示正在执行的指令地址，此时 PC 只是一个概念上的说法。在 ARM 体系中 R15 就是 PC，当然 ARM 和 IA-32、x64 都支持高级内存管理，所以「PC」的内容未必是当前指令在内存中的绝对位置。**

#### 操作系统管理的任务

任务就是进程，或者说任务就是线程。是的，进程和线程正是最常见的任务。但是除此之外，还有没有其他的任务呢？硬件通过触发信号，会导致中断处理程序的调用，也是一种常见的任务。

#### CPU上下文切换

先把前一个任务的CPU上下文（也就是CPU寄存器和程序计数器）保存起来，然后加载新任务的上下文到这些寄存器和程序计数器，最后在跳转到程序计数器所指的新闻纸，运行新任务。*可参考SROP-study深入理解。*

根据操作系统管理的任务不同，CPU的上下文切换可以分布三个不同的场景，也就是**进程上下文切换、线程上下文切换、以及中断上下文切换**

### 进程上下文切换

#### 前置知识:

Linux 按照特权等级，把进程的运行空间分为内核空间和用户空间， CPU 特权等级的 Ring 0 和 Ring 3。Ring0权限最高，Ring3最小，只能访问受限资源，不能访问内存等硬件设备，必须通过系统调用陷入内核中，才能访问特权资源。

进程既会在用户空间运行，也会在内核空间运行。进程在用户空间运行时，被称为进程的用户态，陷入内核空间时，成为进程的内核态。

从用户态到内核态的转变，需要系统调用来完成，查看文件就需要多次系统调用，1. 调用open() 打开文件 2.调用read()读取文件，并调用write()将内容写入到标准输出 3.调用close()关闭文件。

系统调用时，会发生CPU上下文切换。系统调用开始，保存当前用户态上下文，然后更新为内核态指令的新位置，最后跳转到内核态运行内核任务，系统调用后，则反之。所以，一次系统调用，发生两次CPU上下文切换。

注意:

系统调用过程中，并不会涉及到虚拟内存等进程用户态的资源，也不会切换进程。这跟我们通常所说的进程上下文切换是不一样的：

* 进程上下文切换，从一个进程切换到另一个进程进行

* 系统调用过程中一直是同一个进程中进行。

所以系统调用过程通常称为特权模式切换，而不是上下文切换。但是，事实上，系统调用过程中，CPU的上下文切换还是无法避免。

系统调用指令:

* 快速系统调用指令。32位系统使用sysenter指令，64位系统使用syscall指令。
* 软中断“int 0x80”机制，早期2.6内核及其以前的版本，才用软中断机制进行系统调用。因为软中断机制性能较差

#### 进程上下文切换和系统调用的区别

进程由内核管理和调度，进程切换只能在内核态中进行，所以，进程上下文不仅包括虚拟内存、栈、全局变量等用户空间的资源，还包括内核堆栈、寄存器等内核空间的状态。

因此，进程的上下文切换就比系统调用时多了一步：在保存当前进程的内核状态和 CPU 寄存器之前，需要先把该进程的虚拟内存、栈等保存下来；而加载了下一进程的内核态后，还需要刷新进程的虚拟内存和用户栈。

![](/assets/linux/395666667d77e718da63261be478a96b.png)

根据Tsuna的测试报告，基本上每次上下文切换需要几十纳秒到数微妙的CPU时间。

另外，我们知道， Linux 通过 TLB（Translation Lookaside Buffer）来管理虚拟内存到物理内存的映射关系。当虚拟内存更新后，TLB 也需要刷新，内存的访问也会随之变慢。特别是在多处理器系统上，缓存是被多个处理器共享的，刷新缓存不仅会影响当前处理器的进程，还会影响共享缓存的其他处理器的进程。

进程什么时候会被调度到CPU上运行:

1. 进程执行完毕，从就绪队列里，调用其他进程
2. 为了保证所有进程可以得到公平调度，CPU 时间被划分为一段段的时间片，这些时间片再被轮流分配给各个进程。这样，当某个进程的时间片耗尽了，就会被系统挂起，切换到其它正在等待 CPU 的进程运行。
3. 进程在系统资源不足（比如内存不足）时，要等到资源满足后才可以运行，这个时候进程也会被挂起，并由系统调度其他进程运行。
4. 当进程通过睡眠函数  sleep 这样的方法将自己主动挂起时，自然也会重新调度。
5. 当有优先级更高的进程运行时，为了保证高优先级进程的运行，当前进程会被挂起，由高优先级进程来运行。
6. 发生硬件中断时，CPU 上的进程会被中断挂起，转而执行内核中的中断服务程序。

### 线程上下文切换

**线程是调度的基本单位，而进程则是资源拥有的基本单位。**。说白了，所谓内核中的任务调度，实际上的调度对象是线程；而进程只是给线程提供了虚拟内存、全局变量等资源。所以，对于线程和进程，我们可以这么理解：

* 当进程只有一个线程时，可以认为进程就等于线程。
* 当进程拥有多个线程时，这些线程会共享相同的虚拟内存和全局变量等资源。这些资源在上下文切换时是不需要修改的。
* 线程也有自己的私有数据，比如栈和寄存器等，这些在上下文切换时也是需要保存的。

线程上下文切换：

* 前后两个线程属于不同进程。此时，因为资源不共享，切换过程就跟进程上下文切换一样。
* 前后两个线程属于同一个进程。此时，因为虚拟内存是共享的，所以在切换时，虚拟内存这些资源就保持不动，只需要切换线程的私有数据、寄存器等不共享的数据。

**Note: 虽然同为上下文切换，但同进程内的线程切换，要比多进程间的切换消耗更少的资源，而这，也正是多线程代替多进程的一个优势。**

### 中断上下文切换

中断处理会打断进程的正常调度和执行,来快速响应硬件的事件，转而调用中断处理程序，响应设备事件。

跟进程上下文不同，中断上下文切换并不涉及到进程的用户态。所以，即便中断过程打断了一个正处在用户态的进程，也不需要保存和恢复这个进程的虚拟内存、全局变量等用户态资源。中断上下文，其实只包括内核态中断服务程序执行所必需的状态，包括 CPU 寄存器、内核堆栈、硬件中断参数   等。

对同一个 CPU 来说，中断处理比进程拥有更高的优先级，所以中断上下文切换并不会与进程上下文切换同时发生。同样道理，由于中断会打断正常进程的调度和执行，所以大部分中断处理程序都短小精悍，以便尽可能快的执行结束。

## 如何查看CPU上下文切换状况

### 工具: 

#### vmstat： 

用于分析系统内存的使用情况，也用来分析CPU上下文切换和终端次数

```
tiandiwuji@tiandiwuji:~$ vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 1736052 114604 1011184    0    0    46    23   34   48  0  0 98  1  0
```

含义:（具体可通过`man vmstat`查看）

* cs(context switch) 每秒上下文切换次数

* in(interrupt) 每秒中断次数
* r(Running or Runnable) 就绪队列长度，也就是正在运行和等待CPU的进程数
* b(Blocked) 处于不可中断睡眠状态的进程数

#### pidstat

通过-w选项，可以查看每个进程上下文切换的状况。

```
tiandiwuji@tiandiwuji:~$ pidstat -w 5
Linux 4.15.0-42-generic (tiandiwuji) 	12/10/2018 	_x86_64_	(4 CPU)

09:08:06 AM   UID       PID   cswch/s nvcswch/s  Command
09:08:11 AM     0         1     47.21      0.00  systemd
09:08:11 AM     0         8     45.02      0.00  rcu_sched
09:08:11 AM     0        11      0.40      0.00  watchdog/0
09:08:11 AM     0        14      0.40      0.00  watchdog/1
09:08:11 AM     0        20      0.40      0.00  watchdog/2
09:08:11 AM     0        26      0.40      0.00  watchdog/3
09:08:11 AM     0        28      0.20      0.00  ksoftirqd/3
09:08:11 AM     0        35      1.79      0.00  kworker/2:1
09:08:11 AM     0        42      0.20      0.00  khugepaged
```

重要参数:

* cswch: 每秒自愿上下文切换（voluntary context switches）的次数
* nvcswch: 每秒非自愿上下文切换（non voluntary context switches）的次数

**自愿上下文切换**，是指进程无法获取所需资源，导致的上下文切换。I/O、内存等系统资源不足时，就会发生自愿上下文切换。

**非自愿上下文切换**，则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换。大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。

#### sysbench

多线程的基准测试工具，一般用来评估不同系统参数下的数据库负载情况。

案例分析:

略。

## 某个CPU使用率达到100%

### CPU使用率

CPU 使用率是单位时间内 CPU 使用情况的统计，以百分比的形式显示。

CPU的时间会被分为很短的时间片，有调度器调度给各任务使用。

Linux定义节拍率，触发时间中断，并通过全局变量Jiffies记录开机以来的节拍数。

节拍率HZ是内核可配置选项。用户程序无法访问，内核还提供用户空间节拍率USER_HZ，固定位100.

```
root@tiandiwuji:/home/tiandiwuji# grep 'CONFIG_HZ=' /boot/config-$(uname -r)
CONFIG_HZ=250
```

/proc/stat 提供系统的CPU和任务的统计信息。

```
root@tiandiwuji:/home/tiandiwuji# cat /proc/stat | grep ^cpu
cpu  44665 3585 162408 2374608 26212 0 839 0 0 0
cpu0 11846 651 40882 592547 6989 0 204 0 0 0
cpu1 10418 1059 40370 594949 6362 0 87 0 0 0
cpu2 11638 551 40506 593987 6458 0 290 0 0 0
cpu3 10762 1323 40648 593124 6401 0 257 0 0 0
```

含义:

<ul>
<li>
<p>user（通常缩写为 us），代表用户态 CPU 时间。注意，它不包括下面的 nice 时间，但包括了 guest 时间。</p>
</li>
<li>
<p>nice（通常缩写为 ni），代表低优先级用户态 CPU 时间，也就是进程的 nice 值被调整为 1-19 之间时的 CPU 时间。这里注意，nice 可取值范围是 -20 到 19，数值越大，优先级反而越低。</p>
</li>
<li>
<p>system（通常缩写为 sys），代表内核态 CPU 时间。</p>
</li>
<li>
<p>idle（通常缩写为 id），代表空闲时间。注意，它不包括等待 I/O 的时间（iowait）。</p>
</li>
<li>
<p>iowait（通常缩写为 wa），代表等待 I/O 的 CPU 时间。</p>
</li>
<li>
<p>irq（通常缩写为 hi），代表处理硬中断的 CPU 时间。</p>
</li>
<li>
<p>softirq（通常缩写为 si），代表处理软中断的 CPU 时间。</p>
</li>
<li>
<p>steal（通常缩写为 st），代表当系统运行在虚拟机中的时候，被其他虚拟机占用的 CPU 时间。</p>
</li>
<li>
<p>guest（通常缩写为 guest），代表通过虚拟化运行其他操作系统的时间，也就是运行虚拟机的 CPU 时间。</p>
</li>
<li>
<p>guest_nice（通常缩写为 gnice），代表以低优先级运行虚拟机的时间。</p>
</li>
</ul>

**CPU使用率 = ( 1 - 空闲时间 / 总CPU时间)**

一般会间隔一段时间取两次值，计算平均CPU使用率

**平局CPU使用率 = 1 - ((新空闲时间 - 旧空闲时间) / (新总CPU时间 / 旧总CPU时间))**

### 查看CPU使用率

* top 显示了系统总体的 CPU 和内存使用情况，以及各个进程的资源使用情况。

* ps 则只显示了每个进程的资源使用情况。

top指令空白行之后是进程的实时信息，每一行的%cpu列，表示进程CPU使用率，包括用户态和内核态CPU使用率之和，包括进程用户空间使用的CPU、通过系统调用执行的内核CPU、以及在就绪队列等待运行的CPU。在虚拟化运行环境中，还包括运行虚拟机占用的CPU。

查看进程详细情况，包括用户态CPU和内核态CPU：

使用pidstat命令。`pidstat 1 5`

### CPU使用率过高

1. GDB 会中断程序运行，只适合性能分析后期，找到问题后，线下在调试
2. perf 内置性能分析工具，它以性能事件采样为基础，不仅可以分析系统的各种事件和内核性能，还可以用来分析指定应用程序的性能问题。

`perf top -g -p <pid>` :  对进程进行跟踪分析其调用

`perf top`，类似于 top，它能够实时显示占用 CPU，时钟最多的函数或者指令。

```
Samples: 1K of event 'cpu-clock', Event count (approx.): 276859353
Overhead  Shared Object                    Symbol
  38.00%  [kernel]                         [k] _raw_spin_unlock_irqrestore
  11.82%  [kernel]                         [k] vmw_cmdbuf_header_submit
   1.86%  [kernel]                         [k] __softirqentry_text_start
   1.11%  [kernel]                         [k] vsnprintf
   1.08%  libc-2.27.so                     [.] __strcmp_sse2_unaligned
   1.04%  [kernel]                         [k] format_decode
   1.01%  libc-2.27.so                     [.] _int_malloc
   0.79%  [kernel]                         [k] finish_task_switch
   0.79%  perf                             [.] 0x00000000001ea447
   0.69%  libgobject-2.0.so.0.5600.2       [.] g_type_check_instance_is_a
   0.67%  libc-2.27.so                     [.] malloc
   0.63%  [kernel]                         [k] exit_to_usermode_loop
   0.62%  [kernel]                         [k] kallsyms_expand_symbol.constprop.1
   0.60%  libmutter-clutter-2.so           [.] clutter_actor_is_visible
```

输出结果中，第一行包含三个数据，分别是采样数（Samples）、事件类型（event）和事件总数量（Event count）。比如这个例子中，perf 总共采集了 833 个 CPU 时钟事件，而总事件数则为 97742399。**采样数需要我们特别注意.**,如果采样数过少，那么排序和百分比就没什么参考价值。

`perf record`可以实时展示系统性能信息，并保存数据，`perf report`用于解析展示`perf record`保存的数据。参数`-g`表示开启调用关系的采样。

案例:

略。

## CPU使用率很高，找不到高CPU的应用

系统的 CPU 使用率，不仅包括进程用户态和内核态的运行，还包括中断处理、等待 I/O 以及内核线程等。所以当发现系统的 CPU 使用率很高的时候，不一定能找到相对应的高CPU使用率的进程。

execsnoop 专为短时进程设计的工具。它通过 ftrace 实时监控进程的exec() 行为，并输出短时进程的基本信息，包括进程 PID、父进程 PID、命令行参数以及执行的结果。

https://github.com/brendangregg/perf-tools/blob/master/execsnoop

案例：略

## 系统中出现大量不可中断进程和僵尸进程

cpu使用类型除了用户CPU之外，还包括系统CPU（上下文切换），等待I/O的CPU（比如等待磁盘的响应）以及中断CPU（包括软中断和硬中断）等。

I/O的CPU（iowait）升高的问题：

当 iowait 升高时，进程很可能因为得不到硬件的响应，而长时间处于不可中断状态。从 ps 或者 top 命令的输出中，你可以发现它们都处于 D 状态，也就是不可中断状态（Uninterruptible Sleep）。

状态：

R： 是 Running 或 Runnable 的缩写，表示进程在 CPU 的就绪队列中，正在运行或者正在等待运行。

D：  是 Disk Sleep 的缩写，也就是不可中断状态睡眠（Uninterruptible Sleep），一般表示进程正在跟硬件交互，并且交互过程不允许被其他进程或中断打断。

Z：是 Zombie 的缩写，它表示僵尸进程，也就是进程实际上已经结束了，但是父进程还没有回收它的资源（比如进程的描述符、PID 等）。

S:  是 Interruptible Sleep 的缩写，也就是可中断状态睡眠，表示进程因为等待某个事件而被系统挂起。当进程等待的事件发生时，它会被唤醒并进入 R 状态。

I： 是 Idle 的缩写，也就是空闲状态，用在不可中断睡眠的内核线程上。前面说了，硬件交互导致的不可中断进程用 D 表示，但对某些内核线程来说，它们有可能实际上并没有任何负载，用 Idle 正是为了区分这种情况。要注意，D 状态的进程会导致平均负载升高， I 状态的进程却不会。

T或者t: ，也就是 Stopped 或 Traced 的缩写，表示进程处于暂停或者跟踪状态。

向一个进程发送 SIGSTOP 信号，它就会因响应这个信号变成暂停状态（Stopped）；再向它发送 SIGCONT 信号，进程又会恢复运行（如果进程是终端里直接启动的，则需要你用 fg 命令，恢复到前台运行）。

而当你用调试器（如 gdb）调试一个进程时，在使用断点中断进程后，进程就会变成跟踪状态，这其实也是一种特殊的暂停状态，只不过你可以用调试器来跟踪并按需要控制进程的运行。

X：也就是 Dead 的缩写，表示进程已经消亡，所以你不会在 top 或者 ps 命令中看到它。

一旦父进程没有处理子进程的终止，使得子进程一直保持运行状态，那么子进程就会陷入僵尸状态。大量的僵尸进程会用尽PID进程号，导致新进程没有办法创建。

pid最大值：

```
$ cat /proc/sys/kernel/pid_max 
131072
```

案例:

dstat: 是一个新的性能工具，它吸收了 vmstat、iostat、ifstat 等几种工具的优点，可以同时观察系统的 CPU、磁盘 I/O、网络以及内存使用情况。

1. O_DIRECT 选项打开磁盘，于是绕过了系统缓存，直接对磁盘进行读写。容易导致 iowait升高
2. 正确处理子进程

```
int status = 0;
for (;;) {
	for (int i = 0; i < 2; i++) {
		if(fork()== 0) {
			sub_process(disk);
		}
	}

	sleep(5);
}

// 应该将该行放到for循环里
while(wait(&status)>0);
======================》
for (;;) {
		for (int i = 0; i < 2; i++) {
			if(fork()== 0) {
				sub_process(disk);
			}
		}

		while(wait(&status)>0);
		sleep(5);
	}
```

iowait 高不一定代表 I/O 有性能瓶颈。当系统中只有 I/O 类型的进程在运行时，iowait 也会很高，但实际上，磁盘的读写远没有达到性能瓶颈的程度。

僵尸进程出现的原因是父进程没有回收子进程的资源出现的。解决办法是找到父进程，在父进程中处理，使用pstree查父进程，然后查看父进程的源码检查wait()/waitpid()的调用或SIGCHLD信号处理函数的注册。

略。

## 怎么理解Linux软中断

进程的不可中断状态是系统的一种保护机制，可以保证硬件的交互过程不被意外打断。短时间的不可中断正常，长时间的不可中断，就需要确认是不是磁盘I/O的问题。

除了iowait,软中断(sftirq)CPU使用率升高也是常见的性能问题。

中断是系统用来相应硬件设备请求的一种机制，打断进程的正常调度和执行，然后在内核中的中断处理程序来响应设备的请求。

**中断其实也是一种异步的事件处理机制，可以提高系统的并发处理能力**

由于中断处理程序会打断正常程序的运行，所以**为了减少正常进程运行调度的影响，中断处理程序需要尽可能快地运行。**

中断处理程序在响应中断的时候，会临时关闭中断，会导致上一次中断处理完成之前，其他中断无法响应，若此期间有其他中断，该中断可能会丢失。

### 软中断

为了解决中断处理程序执行过长和中断丢失的问题，Linux 将中断处理过程分成了两个阶段，也就是**上半部和下半部**

* 上半部用来快速处理中断，它在中断禁止模式下运行，主要处理跟硬件紧密相关的或时间敏感的工作。
* 下半部用来延迟处理上半部未完成的工作，通常以内核线程的方式运行

以网卡为例:

网卡接收到数据包后，会通过**硬件中断**的方式，通知内核有新的数据到了。这时，内核就应该调用中断处理程序来响应它。对上半部来说，既然是快速处理，其实就是要把网卡的数据读到内存中，然后更新一下硬件寄存器的状态（表示数据已经读好了），最后再发送一个**软中断**信号，通知下半部做进一步的处理。

1. 上半部直接处理硬件请求，也就是我们常说的硬中断，特点是快速执行；
2. 而下半部则是由内核触发，也就是我们常说的软中断，特点是延迟执行。

每个CPU对应一个软中断内核线程，名字"ksoftirqd/CPU编号"。通过ps，可以看到。

```
tiandiwuji@tiandiwuji:~$ ps -aux | grep softirq
root          7  0.0  0.0      0     0 ?        S    03:25   0:00 [ksoftirqd/0]
root         16  0.0  0.0      0     0 ?        S    03:25   0:00 [ksoftirqd/1]
root         22  0.0  0.0      0     0 ?        S    03:25   0:00 [ksoftirqd/2]
root         28  0.0  0.0      0     0 ?        S    03:25   0:00 [ksoftirqd/3]
```

除此以外，一些内核自定义的时间也属于软中断，比如内核调度和RCU锁(Read-Copy Update)的缩写，RCU是linux核最常用的锁之一。

### 查看软中断和内核线程

* /proc/softirqs 提供内核软中断运行情况
* /proc/interrupts 提供硬z中断的运行情况

```
tiandiwuji@tiandiwuji:~$ cat /proc/softirqs | awk '{print $1,$2,$3,$4,$5}'
          CPU0 CPU1 CPU2 CPU3 CPU4
HI:       0 0 1 0
TIMER:    113728 116472 129373 99897     # 定时
NET_TX:   1 2373 0 2                     # 网络接收中断
NET_RX:   150 345 235 36859              # 网络发送中断
BLOCK:    21437 52 13761 11537
IRQ_POLL: 0 0 0 0
TASKLET:  3 3642 36 60
SCHED:    108890 107750 120843 94838     # 调度
HRTIMER:  0 0 0 0
RCU:      122822 114064 107781 97828

```

第一列标识软中断的10个类别。正常情况下，每个cpu上的累计次数差不多。

TASKLET  在不同 CPU 上的分布并不均匀。TASKLET 是最常用的软中断实现机制，每个 TASKLET 只运行一次就会结束 ，并且只在调用它的函数所在的 CPU 上运行。

大量的网络小包会导致性能问题，可能是由于大量的小网络包会导致频繁的硬中断和软中断。

## 系统的软中断CPU使用率升高

sar: sar 是一个系统活动报告工具，既可以实时查看系统的当前活动，又可以配置保存和报告历史统计数据。

hping3: hping3 是一个可以构造 TCP/IP 协议数据包的工具，可以对系统进行安全审计、防火墙测试等。

tcpdump: tcpdump 是一个常用的网络抓包工具，常用来分析各种网络问题。

案例：

略。

## Linux内存是怎么工作的

### 虚拟内存

物理内存也成为主存，大多是动态随机访问内存（DRAM)。只有内核能够直接访问物理内存。

Linux为每个进程提供独立的虚拟地址空间，且连续。

虚拟地址空间被分为内核空间和用于空间，32位和64位系统地址空间范围不同。

![](/assets/linux/ed8824c7a2e4020e2fdd2a104c70ab7b.png)

具体版:

32位linux内核2.6.7以前默认布局：

![](/assets/linux/TIM截图20190103170730.png)

32位当前默认布局：

![](/assets/linux/TIM截图20190103170839.png)

64位默认布局：

![](/assets/linux/TIM截图20190103170947.png)

不同的进程都包含内核空间，但是这些内核空间均是相同的物理内存。进程切换到内核态之后，可以很方便的访问内核空间内存。

### 内存映射

将虚拟内存地址映射到物理地址内存：

![](/assets/linux/TIM截图20190103171140.png)

页表存储在CPU的内存管理单元MMU中，这样，正常情况下，处理器可以通过硬件找出要访问的内存。而当进程访问的虚拟地址在页表中查不到时，系统会产生一个**缺页异常**，，进入内核空间分配物理内存、更新进程页表，最后再返回用户空间，恢复进程的运行。

Linux 通过 TLB（Translation Lookaside Buffer，转译后备缓冲器）来管理虚拟内存到物理内存的映射关系。当虚拟内存更新后，TLB 也需要刷新，内存的访问也会随之变慢。特别是在多处理器系统上，缓存是被多个处理器共享的，刷新缓存不仅会影响当前处理器的进程，还会影响共享缓存的其他处理器的进程。原因如上。

TLB 其实就是 MMU 中页表的高速缓存。由于进程的虚拟地址空间是独立的，而 TLB 的访问速度又比 MMU 快得多，所以，通过减少进程的上下文切换，减少 TLB 的刷新次数，就可以提高 TLB 缓存的使用率，进而提高 CPU 的内存访问性能。不过要注意，MMU 并不以字节为单位来管理内存，而是规定了一个内存映射的最小单位，也就是页，通常是 4 KB 大小。这样，每一次内存映射，都需要关联 4 KB 或者 4KB 整数倍的内存空间。页的大小只有 4 KB ，导致的另一个问题就是，整个页表会变得非常大。比方说，仅 32 位系统就需要 100 多万个页表项（4GB/4KB），才可以实现整个地址空间的映射。为了解决页表项过多的问题，Linux 提供了两种机制，也就是多级页表和大页（HugePage）。

Linux 用的正是四级页表来管理内存页，如下图所示，虚拟地址被分为 5 个部分，前 4 个表项用于选择页，而最后一个索引表示页内偏移。

![](/assets/linux/b5c9179ac64eb5c7ca26448065728325.png)

### 内存分配与回收

#### 分配：

malloc是c标准库提供的内存分配函数，对应到系统调用上，有两种实现方式，`brk()`和`mmap()`。

对于小块内存(小于128k)，采用brk分配，通过移动堆顶位置来分配，释放后不会立即归还，会被缓存起来。易产生内存碎片 。释放:`free()`

对于大块内存(大于128k)，直接使用内存映射mmap()分配，在文件映射端找一段空闲内存分配出去。释放后，直接归还系统，每次mmap都会发生缺页异常。易导致内核管理负担增大。释放: `unmap`

通过以上两种方式调用之后，不会立即分配物理内存，只有在首次访问时，才会分配，通过缺页异常进入内核，由内核分配内存。

在内核空间，Linux采用slab分配器管理小内存。可以把slab看成构建在伙伴系统上的一个缓存，主要作用就是分配并释放内核中的小对象。

#### 回收

* 回收缓存，比如使用 LRU（Least Recently Used）算法，回收最近使用最少的内存页面；
* 回收不常访问的内存，把不常用的内存通过交换分区直接写到磁盘中；也就是swap（交换分区中）
* 杀死进程，内存紧张时系统还会通过 OOM（Out of Memory），直接杀掉占用大量内存的进程。

第三种方法是内核的一种保护机制，监控内存使用情况，并使用oom_score为每个进程的内存使用情况进行评分。

* 一个进程消耗的内存越大，oom_score 就越大；
* 一个进程运行占用的 CPU 越多，oom_score 就越小。

该值可通过`/proc`文件系统，手动设置进程的oom_adj，调整oom_score。oom_adj 的范围是 [-17, 15]，数值越大，表示进程越容易被 OOM 杀死；数值越小，表示进程越不容易被 OOM 杀死，其中 -17 表示禁止 OOM。

修改sshd进程的oom_score

```
echo -16 > /proc/$(pidof sshd)/oom_adj
```

### 查看内存使用情况

`free`

```
[root@iz2zehekqyjug6ar3u4omgz ~]# free
              total        used        free      shared  buff/cache   available
Mem:        1883724       79732      164524         356     1639468     1602056
Swap:             0           0           0
```

available 不仅包含未使用内存，还包括了可回收的缓存，所以一般会比未使用内存更大。不过，并不是所有缓存都可以回收，因为有些缓存可能正在使用中。

`top`,按`M`切换到内存内存排序

```
top - 17:47:49 up 46 days, 22:21,  2 users,  load average: 0.00, 0.01, 0.05
Tasks:  67 total,   2 running,  65 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.3 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  1883724 total,   162908 free,    79996 used,  1640820 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  1601808 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
  327 root      20   0   69684  30772  30460 S  0.0  1.6   0:15.87 systemd-journal
13420 root      20   0  293244  20100  16804 S  0.0  1.1   2:46.02 rsyslogd
13618 root      20   0  573752  17124   6108 S  0.0  0.9   6:49.66 tuned
  677 root      20   0  112876  12792    312 S  0.0  0.7   0:00.00 dhclient
 2148 root      20   0  129984  11608   9280 R  0.0  0.6 153:39.71 AliYunDun
13537 polkitd   20   0  538564  10364   4788 S  0.0  0.6   0:07.49 polkitd
16159 root      20   0  152540   5508   4212 S  0.0  0.3   0:00.01 sshd
16240 root      20   0  152540   5508   4212 S  0.0  0.3   0:00.03 sshd
 6159 root      20   0  112812   4312   3284 S  0.0  0.2   0:00.22 sshd
    1 root      20   0   43444   3664   2432 S  0.0  0.2   0:34.82 systemd
14443 nginx     20   0  121272   3576   1060 S  0.0  0.2   0:00.00 nginx
```

* VIRT:是进程虚拟内存的大小，只要是进程申请过的内存，即便还没有真正分配物理内存，也会计算在内。
* RES 是常驻内存的大小，也就是进程实际使用的物理内存大小，但不包括 Swap 和共享内存。
* SHR 是共享内存的大小，比如与其他进程共同使用的共享内存、加载的动态链接库以及程序的代码段等。
* %MEM 是进程使用物理内存占系统总内存的百分比。

注意:

1. 第一，虚拟内存通常并不会全部分配物理内存。从上面的输出，你可以发现每个进程的虚拟内存都比常驻内存大得多。
2. 共享内存 SHR 并不一定是共享的，比方说，程序的代码段、非共享的动态链接库，也都算在 SHR 里。当然，SHR 也包括了进程间真正共享的内存。所以在计算多个进程的内存使用时，不要把所有进程的 SHR 直接相加得出结果。

## 怎么理解内存中的Buffer和Cache

buffer: 缓冲区。cache: 缓存。两者都是数据在内存中的临时存储。

free命令数据来源

* Buffers 是内核缓冲区用到的内存，对应的是  /proc/meminfo 中的 Buffers 值。
* Cache 是内核页缓存和 Slab 用到的内存，对应的是  /proc/meminfo 中的 Cached 与 SReclaimable 之和。

查看proc

* Buffers 是对原始磁盘块的临时存储，也就是用来**缓存磁盘的数据**,，通常不会特别大（20MB 左右）。这样，内核就可以把分散的写集中起来，统一优化磁盘的写入，比如可以把多次小的写合并成单次大的写等等。

* Cached 是从磁盘读取文件的页缓存，也就是用来**缓存从文件读取的数据**,。这样，下次访问这些文件数据时，就可以直接从内存中快速获取，而不需要再次访问缓慢的磁盘。\
* SReclaimable 是 Slab 的一部分。Slab 包括两部分，其中的可回收部分，用 SReclaimable 记录；而不可回收部分，用 SUnreclaim 记录。

```
echo 3 >/proc/sys/vm/drop_caches
```

` /proc/sys/vm/drop_caches` ，这是通过 proc 文件系统修改内核行为的一个示例，写入 3 表示清理文件页、目录项、Inodes 等各种缓存。

写文件时会用到 Cache 缓存数据，而写磁盘则会用到 Buffer 来缓存数据。虽然文档上只提到，Cache 是文件读的缓存，但实际上，Cache 也会缓存写文件时的数据。同样，Buffer 既可以用作“将要写入磁盘数据的缓存”，也可以用作“从磁盘读取数据的缓存”。

**Buffer 是对磁盘数据的缓存，而 Cache 是文件数据的缓存，它们既会用在读请求中，也会用在写请求中**

注: linux中的块设备可以直接访问（比如数据库应用程序），也可以通过存储文件系统访问

## 利用系统缓存优化程序运行效率

**存的命中率**：是指直接通过缓存获取数据的请求次数，占所有数据请求次数的百分比。

安装cachestat, cachetop，这两工具均为bcc软件包的一部分。基于Linux内核的eBPF(extended Berkeley Packet Filters)机制，跟踪内核中管理的缓存，并输出缓存的使用和命中情况。

```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 4052245BD4284CDD
echo "deb https://repo.iovisor.org/apt/xenial xenial main" | sudo tee /etc/apt/sources.list.d/iovisor.list
sudo apt-get update
sudo apt-get install -y bcc-tools libbcc-examples linux-headers-$(uname -r)
```

```
export PATH=$PATH:/usr/share/bcc/tools
```

安装pcstat:

```
$ export GOPATH=~/go
$ export PATH=~/go/bin:$PATH
$ go get golang.org/x/sys/unix
$ go get github.com/tobert/pcstat/pcstat
```

`go get golang.org/x/sys/unix` 报错:

```
package golang.org/x/sys/unix: unrecognized import path "golang.org/x/sys/unix" (https fetch: Get https://golang.org/x/sys/unix?go-get=1: dial tcp 216.239.37.1:443: getsockopt: connection refused)
```

解决方案:

```
$ mkdir -p $GOPATH/src/golang.org/x/
$ git clone https://github.com/golang/sys.git sys
$ go install golang.org/x/sys/unix
```

报错:

`undefined runtime.KeepAlive`,原因：

```
Sorry, Go 1.6 is no longer supported.

If you were using Go 1.6 for App Engine, note that GAE standard supports Go 1.9 (by default) now.
```

升级golang即可。

首先卸载go:

```
sudo apt-get remove golang-1.6
sudo apt-get install golang-1.9
```

注意，原来卸载之后并没有把这里的文件卸载干净，这里的可执行文件也没有变。需要先清空原先的go文件夹。 

```
cd /usr/lib
rm -rf go-1.6
rm -rf go
mkdir go
cp -r ./go-1.9/* ./go
```

配置环境变量:

```
在/etc/profile文件末尾添加，然后source /etc/profile
export GOPATH=/go
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```

### centos安装bcc-tools

```
[root@centos-80 ~]# yum update
[root@centos-80 ~]# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org && rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
[root@centos-80 ~]# uname -r ## 
3.10.0-862.el7.x86_64
[root@centos-80 ~]# yum remove kernel-headers kernel-tools kernel-tools-libs
[root@centos-80 ~]# yum --disablerepo="*" --enablerepo="elrepo-kernel" install kernel-ml kernel-ml-devel kernel-ml-headers kernel-ml-tools kernel-ml-tools-libs kernel-ml-tools-libs-devel 
[root@centos-80 ~]# sed -i '/GRUB_DEFAULT/s/=.*/=0/' /etc/default/grub
[root@centos-80 ~]# grub2-mkconfig -o /boot/grub2/grub.cfg
[root@centos-80 ~]# reboot
[root@centos-80 ~]# uname -r ## 升级成功
4.20.0-1.el7.elrepo.x86_64
[root@centos-80 ~]# yum install -y bcc-tools
[root@centos-80 ~]# echo 'export PATH=$PATH:/usr/share/bcc/tools' > /etc/profile.d/bcc-tools.sh
[root@centos-80 ~]# . /etc/profile.d/bcc-tools.sh
[root@centos-80 ~]# cachestat 1 1 ## 测试安装是否成功
TOTAL MISSES HITS DIRTIES BUFFERS_MB CACHED_MB
0 0 0 0 2 287
```

测试案例:

略。

## 内存分配

对普通进程来说，能看到的其实是内核提供的虚拟内存，这些虚拟内存还需要通过页表，由系统映射为物理内存。

当进程通过 malloc() 申请虚拟内存后，系统并不会立即为其分配物理内存，而是在首次访问时，才通过缺页异常陷入内核中分配内存。

为了协调 CPU 与磁盘间的性能差异，Linux 还会使用 Cache 和 Buffer ，分别把文件和磁盘读写的数据缓存到内存中。

内存泄漏:

* 没正确回收分配后的内存，导致了泄漏。
* 访问的是已分配内存边界外的地址，导致程序异常退出，等等。

用户空间内存包括多个不同的内存段，比如只读段、数据段、堆、栈以及文件映射段等。这些内存段正是应用程序使用内存的基本方式。

* 栈内存由系统自动分配和管理。一旦程序运行超出了这个局部变量的作用域，栈内存就会被系统自动回收，所以不会产生内存泄漏的问题。
* 使用标准库函数 malloc()在程序中动态分配内存。这时候，系统就会从内存空间的堆中分配内存。堆内存由应用程序自己来分配和管理。除非程序退出，这些堆内存并不会被系统自动释放，而是需要应用程序明确调用库函数 来释放它们。如果应用程序没有正确释放堆内存，就会造成内存泄漏。
* 只读段，包括程序的代码和常量，由于是只读的，不会再去分配新的内存，所以也不会产生内存泄漏。
* 数据段，包括全局变量和静态变量，这些变量在定义时就已经确定了大小，所以也不会产生内存泄漏。
* 最后一个内存映射段，包括动态链接库和共享内存，其中共享内存由程序动态分配和管理。所以，如果程序在分配后忘了回收，就会导致跟堆内存类似的泄漏问题。

虽然，系统最终可以通过 OOM （Out of Memory）机制杀死进程，但进程在 OOM 前，可能已经引发了一连串的反应，导致严重的性能问题。比如，其他需要内存的进程，可能无法分配新的内存；内存不足，又会触发系统的缓存回收以及 SWAP 机制，从而进一步导致 I/O 的性能问题等等。

案例实战:

检测内存泄漏的工具，memleak。memleak 可以跟踪系统或指定进程的内存分配、释放请求，然后定期输出一个未释放内存和相应调用栈的汇总情况（默认 5 秒）。

略

## 系统的Swap增高

内存回收，也就是系统释放掉可以回收的内存，比如缓存和缓冲区，就属于可回收内存。它们在内存管理中，通常被叫做**文件页**。

大部分文件页，都可以直接回收，以后有需要时，再从磁盘重新读取就可以了。而那些被应用程序修改过，并且暂时还没写入磁盘的数据（也就是脏页），就得先写入磁盘，然后才能进行内存释放。

这些脏页，一般可以通过两种方式写入磁盘。

* 可以在应用程序中，通过系统调用 fsync  ，把脏页同步到磁盘中；
* 也可以交给系统，由内核线程 pdflush 负责这些脏页的刷新。

应用程序动态分配的堆内存，也就是我们在内存管理中说到的匿名页，这些内存可能还要被访问，但是，如果这些内存在分配后很少被访问，也是一种资源浪费。Linux的Swap机制可以把它们暂时先存在磁盘里，释放内存给其他更需要的进程。

### Swap原理

Swap 说白了就是把一块磁盘空间或者一个本地文件（以下讲解以磁盘为例），当成内存来使用。它包括换出和换入两个过程。

* 换出，就是把进程暂时不用的内存数据存储到磁盘中，并释放这些数据占用的内存。
* 换入，则是在进程再次访问这些内存的时候，把它们从磁盘读到内存中来。

使用场景，常见的笔记本电脑的休眠和快速开机的功能，也基于 Swap 。休眠时，把系统的内存存入磁盘，这样等到再次开机时，从磁盘中加载内存就可以。就省去了很多应用程序的初始化过程，加快开机速度。

**直接内存回收**:有新的大块内存分配请求，但是剩余内存不足。这个时候系统就需要回收一部分内存（比如前面提到的缓存），进而尽可能地满足新内存请求。

除了直接内存回收，还有一个专门的内核线程用来定期回收内存，也就是**kswapd0**。

```
tiandiwuji@tiandiwuji:~$ sudo su
[sudo] password for tiandiwuji: 
root@tiandiwuji:/home/tiandiwuji# ps -aux | grep swap
root         55  0.0  0.0      0     0 ?        S    07:33   0:00 [kswapd0]
root        308  0.0  0.0      0     0 ?        I<   07:33   0:00 [ttm_swap]
```

为了衡量内存的使用情况，kswapd0 定义了三个内存阈值（watermark，也称为水位），分别是

* 页最小阈值（pages_min）
* 页低阈值（pages_low）
* 页高阈值（pages_high）。

剩余内存，则使用 pages_free 表示。

![](/assets/linux/c1054f1e71037795c6f290e670b29120.png)

kswapd0 定期扫描内存的使用情况，并根据剩余内存落在这三个阈值的空间位置，进行内存的回收操作。

* 剩余内存小于**页最小阈值**,，说明进程可用内存都耗尽了，只有内核才可以分配内存。
* 剩余内存落在**页最小阈值**和**页低阈值** 中间，说明内存压力比较大，剩余内存不多了。这时 kswapd0 会执行内存回收，直到剩余内存大于高阈值为止。

* 剩余内存落在**页低阈值**和**页高阈值**中间，说明内存有一定压力，但还可以满足新内存请求。

* 剩余内存大于**页高阈值**，，说明剩余内存比较多，没有内存压力。

一旦剩余内存小于页低阈值，就会触发内存的回收。这个页低阈值，其实可以通过内核选项 /proc/sys/vm/min_free_kbytes 来间接设置。min_free_kbytes 设置了页最小阈值，而其他两个阈值，都是根据页最小阈值计算生成的，计算方法如下 ：

```
pages_low = pages_min*5/4
pages_high = pages_min*3/2
```

### NUMA和Swap(NUMA不太懂？)

为什么剩余内存很多的情况下，也会发生 Swap 呢？

这正是处理器的 NUMA （Non-Uniform Memory Access）架构导致的。在 NUMA 架构下，多个处理器被划分到不同 Node 上，且每个 Node 都拥有自己的本地内存空间。而同一个 Node 内部的内存空间，实际上又可以进一步分为不同的内存域（Zone），比如直接内存访问区（DMA）、普通内存区（NORMAL）、伪内存区（MOVABLE）等，如下图所示： 

![](/assets/linux/be6cabdecc2ec98893f67ebd5b9aead9.png)

既然 NUMA 架构下的每个 Node 都有自己的本地内存空间，那么，在分析内存的使用时，我们也应该针对每个 Node 单独分析。

numactl 命令，来查看处理器在 Node 的分布情况，以及每个 Node 的内存使用情况。

```
root@tiandiwuji:/home/tiandiwuji# numactl --hardware
available: 1 nodes (0)
node 0 cpus: 0 1 2 3
node 0 size: 3921 MB
node 0 free: 2096 MB
node distances:
node   0 
  0:  10 
```

前面提到的三个内存阈值（页最小阈值、页低阈值和页高阈值），都可以通过内存域在 proc 文件系统中的接口 /proc/zoneinfo 来查看。

```
root@tiandiwuji:/home/tiandiwuji# cat /proc/zoneinfo  | more
..................................
pages free     3968
        min      67
        low      83
        high     99
        spanned  4095
        present  3997
        managed  3976
        protection: (0, 2938, 3846, 3846, 3846)
      nr_free_pages 3968
      nr_zone_inactive_anon 0
      nr_zone_active_anon 0
      nr_zone_inactive_file 0
      nr_zone_active_file 0
.................................
```

* pages 处的 min、low、high，就是上面提到的三个内存阈值，而 free 是剩余内存页数，它跟后面的 nr_free_pages 相同。
* nr_zone_active_anon 和 nr_zone_inactive_anon，分别是活跃和非活跃的匿名页数。
* nr_zone_active_file 和 nr_zone_inactive_file，分别是活跃和非活跃的文件页数。

当然，某个 Node 内存不足时，系统可以从其他 Node 寻找空闲内存，也可以从本地内存中回收内存。具体选哪种模式，你可以通过 /proc/sys/vm/zone_reclaim_mode 来调整。它支持以下几个选项：

* 默认的 0 ，也就是刚刚提到的模式，表示既可以从其他 Node 寻找空闲内存，也可以从本地回收内存。
* 1、2、4 都表示只回收本地内存，2 表示可以回写脏数据回收内存，4 表示可以用 Swap 方式回收内存。

### swappiness

* 对文件页的回收，当然就是直接回收缓存，或者把脏页写回磁盘后再回收。

* 对匿名页的回收，其实就是通过 Swap 机制，把它们写入磁盘后再释放内存。

两种不同的回收机制，实际回收时，采用哪种方式:

Linux 提供了一个  /proc/sys/vm/swappiness 选项，用来调整使用 Swap 的积极程度。

swappiness 的范围是 0-100，数值越大，越积极使用 Swap，也就是更倾向于回收匿名页；数值越小，越消极使用 Swap，也就是更倾向于回收文件页。虽然 swappiness 的范围是 0-100，不过要注意，这并不是内存的百分比，而是调整 Swap 积极程度的权重，即使你把它设置成 0，当[剩余内存 + 文件页小于页高阈值](https://www.kernel.org/doc/Documentation/sysctl/vm.txt),还是会发生swap.

，当剩余内存 + 文件页小于页高阈值

案例：

查看每个进程swap使用量

`for file in /proc/*/status ; do awk '/VmSwap|Name|^Pid/{printf $2 " " $3}END{ print ""}' $file; done | sort -k 3 -n -r | head`

计算物理内存使用总量:

```
# 使用 grep 查找 Pss 指标后，再用 awk 计算累加值
$ grep Pss /proc/[1-9]*/smaps | awk '{total+=$2}; END {printf "%d kB\n", total }'
391266 kB
```

略。

## Linux文件系统如何工作

* 磁盘为系统提供了最基本的持久化存储
* 文件系统在磁盘的基础上，提供了一个用来管理文件的树状结构

### 工作原理

#### 索引节点和目录项

Linux一切皆文件，不仅普通的文件和目录，块设备、套接字、管道等都需要通过统一的文件系统管理。

为了方便管理，Linux文件系统为每个文件都分配了两个数据结构，索引接点（index node)和目录项（directory entry)。它们主要用来记录文件的元信息和目录结构。

* 索引节点   记录文件的元数据，比如inode编号、文件大小、访问权限、修改日期、数据的位置。索引节点与文件一一对应，和文件内容一样会被持久化存储到磁盘中。**索引节点也会占用磁盘空间**
* 目录项 detry，记录文件的名字、索引节点指针以及其他目录项的关联关系。多个关联的目录项，就构成了文件系统的目录结构。不过，不同于索引节点，目录项是有内核维护的**一个内存数据结构，通常也被叫做目录项缓存**

索引节点是每个文件的唯一标志，目录项的维护正是文件系统的树状结构。目录项和索引节点的关系是多对一，可以理解为: 一个文件可以有多个别名。

案例: 通过硬链接为文件创建的别名，对应不同的目录项，不过这些目录项本质上还是连接同一个文件，所以，它们的索引节点相同。

#### 文件数据如何存储:

磁盘的最小读取单位是扇区，只有512B大小，为了提高效率，文件系统将连续扇区组合成1块，每次以逻辑块为最小单元管理数据。常见逻辑快大小为4KB,由连续8个扇区组成。

![img](https://static001.geekbang.org/resource/image/32/47/328d942a38230a973f11bae67307be47.png)

注意:

1. 目录项本身就是内存缓存，索引节点则是存储在磁盘中的数据。为了协调慢速磁盘与快速CPU的性能差异，文件内容会缓存到页缓存Cache中。所以节点也会缓存到内存中。
2. 磁盘在执行文件系统格式化时，会分为3个存储区域，超级块、索引节点区和数据块区。
   * 超级块          存储整个文件系统的状态
   * 索引节点区   存储索引节点
   * 数据块区       存储文件数据

### 虚拟文件系统

目录项、索引节点、逻辑块以及超级块，构成Linux文件系统的四大基本要素。为了支持不同的文件系统，Linux内核在用户进程和文件系统中间，引入一个抽象层，也就是虚拟文件系统VFS（Virtual File System)。

VFS定义了一组所有文件系统都支持的数据结构和标准接口。这样，用户进程和内核中的其他子系统，只需要跟VFS提供的统一接口进行交互即可，不需要关心底层的各种文件系统实现细节。

![](https://static001.geekbang.org/resource/image/72/12/728b7b39252a1e23a7a223cdf4aa1612.png)

按照存储文件位置不同，将文件系统分为3类:

* 基于磁盘的文件系统，将数据直接存储在计算机本地挂载的磁盘中。常见的EXT4，XFS，OverlayFS等。
* 基于内存文件系统，也就是常说的虚拟文件系统。这类文件系统，不需要任何磁盘分配存储空间，需要占用内存。比如/proc文件系统，就是常见的虚拟文件系统。/sys文件系统也属于该类，主要想用户空间导出层次化的内核对象。
* 网络文件系统，用来访问其他计算机数据的文件系统，比如NFS、SMB、ISCSI等。

## 文件系统I/O

把文件系统挂载到挂载点之后，通过挂载点，访问它管理的文件。VFS提供了一组标准的文件访问接口。这些接口以系统调用的方式，提供给应用程序使用。

案例: cat命令，首先调用open打开文件，在调用read()读取文件的内容，最后调用write将对融输出。

文件读写方式的各种差异导致I/O分类多种多样，缓冲与非缓冲I/O，直接和非直接I/O，阻塞和非阻塞I/O，同步和异步I/O。

* 根据是否利用标准库缓存，把文件I/O分为缓冲I/O和非缓冲I/O。

  * 缓冲I/O，利用标准库缓存加速文件的访问，而标准库内部通过系统调用访问文件。
  * 非缓冲I/O，直接通过系统调用访问文件，不在经过标准库缓存。

* 根据是否使用操作系统的页缓存，分为直接I/O和非直接I/O

  * 直接I/O，跳过操作系统的页缓存，直接根文件系统交互来访问文件
  * 非直接I/O，文件读写，先经过系统的页缓存，然后由内核或额外的系统调用，真正写入磁盘。

  想要实现直接I/O，需要在系统调用中，指定O_DIRECT标志，如果没有设置，默认非直接I/O。

  直接I/O，非直接I/O，本质上还是和文件系统交互，如果实在数据库等场景中，会出现跳过文件系统读写磁盘的情况，也就是所谓的裸I/O。

* 根据程序是否阻塞自身运行，可以把文件I/O分为阻塞I/O和非阻塞I/O

  * 阻塞I/O 应用程序执行I/O操作后，如果没有或得相应，就会阻塞当前线程，自然不能执行其他任务。
  * 非阻塞I/O 应用程序执行I/O操作后，不会阻塞当前线程，可以继续执行其他任务，然后通过轮询或事件通知的形式，获取调用结果。

  访问管道或者网络套接字时，设置O_NONBLOCK标志，表示使用非阻塞方式访问，如果不做任何设置，默认阻塞访问。

* 根据是否等待响应结果，可以把文件I/O分为同步和异步I/O。

  * 所谓同步I/O，应用程序执行I/O操作后，等到整个I/O完成后，才能或得I/O响应
  * 异步I/O，应用程序执行I/O操作后，不用等待完成后的响应，而是继续执行即可。等到这次I/O完成后，响应会用事件通知的方式，告诉应用程序。

  操作文件时，如果设置O_SYNC或者O_DSYNC标志，代表同步I/O，如果设置O_DSYNC，等文件数据写入磁盘后，才能返回，而O_SYNC则是在O_DSYNC基础上，要求文件元数据也要写入磁盘后，才能返回。

  访问管道或者网络套接字时，设置了O_ASYNC对应的就是异步I/O。这样，内核会通过SIGIO或者SIGPOLL，通知进程文件是否可读写。

  ### 性能观测:

  #### 容量:

  ```
  /dev df -h
  Filesystem      Size   Used  Avail Capacity iused               ifree %iused  Mounted on
  /dev/disk1s1   113Gi   69Gi   42Gi    63%  957481 9223372036853818326    0%   /
  devfs          330Ki  330Ki    0Bi   100%    1140                   0  100%   /dev
  /dev/disk1s4   113Gi  1.0Gi   42Gi     3%       1 9223372036854775806    0%   /private/var/vm
  map -hosts       0Bi    0Bi    0Bi   100%       0                   0  100%   /net
  map auto_home    0Bi    0Bi    0Bi   100%       0                   0  100%   /home
  ```

  查看索引节点情况,索引节点不足，也会导致出现空间不足的问题。

  `df -i`

  #### 缓存:

  通过free或者vmstat，观察也缓存的大小。free输出的Cache是页缓存和Slab缓存的和，可以从/proc/meminfo，直接得到他们的大小。

  ```
  cat /proc/meminfo | grep -E "SReclaimable|Cached"
  ```

  文件系统中目录项和索引节点缓存，如何观察:

  内核使用Slab机制，管理目录项和索引节点缓存。/proc/meminfo只给出Slab的整体大小，具体到每一种Slab缓存，还要查看/proc/slabinfo这个文件。

  查看目录项和各种文件系统索引节点的缓存情况:

  ```
  cat /proc/slabinfo | grep -E "^#|detry|inode"
  ```

  dentry 行表示目录项缓存，inode_cache 行，表示 VFS 索引节点缓存，其余的则是各种文件系统的索引节点缓存。

  /proc/slabinfo 的列比较多，具体含义可以查询  man slabinfo。在实际性能分析中，我们更常使用 slabtop  ，来找到占用内存最多的缓存类型。

  题目:

  `find / -name file-name`会导致什么缓存升高。

  ```
  这个命令，会不会导致系统的缓存升高呢？
  --> 会的
  如果有影响，又会导致哪种类型的缓存升高呢？
  --> /xfs_inode/ proc_inode_cache/dentry/inode_cache
  
  实验步骤：
  1. 清空缓存：echo 3 > /proc/sys/vm/drop_caches ; sync
  2. 执行find ： find / -name test
  3. 发现更新top 4 项是：
    OBJS ACTIVE USE OBJ SIZE SLABS OBJ/SLAB CACHE SIZE NAME
   37400 37400 100% 0.94K 2200 17 35200K xfs_inode
   36588 36113 98% 0.64K 3049 12 24392K proc_inode_cache
  104979 104979 100% 0.19K 4999 21 19996K dentry
   18057 18057 100% 0.58K 1389 13 11112K inode_cache
  
  find / -name 这个命令是全盘扫描（既包括内存文件系统又包含本地的xfs【我的环境没有mount 网络文件系统】），所以 inode_cache & dentry & proc_inode_cache 会升高。
  
  另外，执行过了一次后再次执行find 就机会没有变化了，执行速度也快了很多，也就是下次的find大部分是依赖cache的结果。
  ```

  ## Linux磁盘I/O是怎么工作的

  #### 磁盘

  * 机械磁盘  HDD 有盘头和读写磁头组成，数据存储在盘片的环状磁道中。读写数据前，需要移动读写磁头，定位到数据所在磁道。
  * 固态磁盘 SSD，连续I/O和非连续I/O 均比HDD好得多。

  机械磁盘最小读写单位扇区，大小为512字节

  固态磁盘最小读取单位页，大小为4kb，8kb等

  按照接口分类: IDE(Integrated Drive Electronics) 前缀hd, SCSI 前缀sd,SAS 前缀sd,SATA,FC等。

  Linux中， 磁盘实际上是作为一个块设备来管理的，也就是以块为单位读写数据，并且支持随机读写。每个块设备都会被赋予两个设备号，分别是主、次设备号。主设备号用在驱动程序中，用来区分设备类型；而次设备号则是用来给多个同类设备编号。

  #### 通用块层

  跟虚拟文件系统 VFS 类似，为了减小不同块设备的差异带来的影响，Linux 通过一个统一的通用块层，来管理各种不同的块设备。

  通用块层，其实是处在文件系统和磁盘驱动中间的一个块设备抽象层。它主要有两个功能 。

  * 第一个功能跟虚拟文件系统的功能类似。向上，为文件系统和应用程序，提供访问块设备的标准接口；向下，把各种异构的磁盘设备抽象为统一的块设备，并提供统一框架来管理这些设备的驱动程序。
  * 第二个功能，通用块层还会给文件系统和应用程序发来的 I/O 请求排队，并通过重新排序、请求合并等方式，提高磁盘读写的效率。

  对 I/O 请求排序的过程，也就是我们熟悉的 I/O 调度。事实上，Linux 内核支持四种 I/O 调度算法，分别是 NONE、NOOP、CFQ 以及 DeadLine。这里我也分别介绍一下。

  * NONE

    不能算 I/O 调度算法。因为它完全不使用任何 I/O 调度器，对文件系统和应用程序的 I/O 其实不做任何处理，常用在虚拟机中（此时磁盘 I/O 调度完全由物理机负责）。

  * NOOP 

    最简单的一种 I/O 调度算法。它实际上是一个先入先出的队列，只做一些最基本的请求合并，常用于 SSD 磁盘。

  * CFQ（Completely Fair Scheduler）

    也被称为完全公平调度器，是现在很多发行版的默认 I/O 调度器，它为每个进程维护了一个 I/O 调度队列，并按照时间片来均匀分布每个进程的 I/O 请求。

    类似于进程 CPU 调度，CFQ 还支持进程 I/O 的优先级调度，所以它适用于运行大量进程的系统，像是桌面环境、多媒体应用等。

  * DeadLine

    调度算法，分别为读、写请求创建了不同的 I/O 队列，可以提高机械磁盘的吞吐量，并确保达到最终期限（deadline）的请求被优先处理。DeadLine 调度算法，多用在 I/O 压力比较重的场景，比如数据库等。

  #### I/O栈

  LInux 存储系统的I/O栈，从上到下分为三个层次，文件系统层、通用块层和设备层。

  ![](https://static001.geekbang.org/resource/image/14/b1/14bc3d26efe093d3eada173f869146b1.png)

  * 文件系统层，包括虚拟文件系统和其他各种文件系统的具体实现。它为上层的应用程序，提供标准的文件访问接口；对下会通过通用块层，来存储和管理磁盘数据。
  * **通用块层，包括块设备 I/O 队列和 I/O 调度器。它会对文件系统的 I/O 请求进行排队，再通过重新排序和请求合并，然后才要发送给下一级的设备层。**
  * 设备层，包括存储设备和相应的驱动程序，负责最终物理设备的 I/O 操作。

  存储系统的 I/O ，通常是整个系统中最慢的一环。所以， Linux 通过多种缓存机制来优化 I/O 效率。比方说，为了优化文件访问的性能，会使用页缓存、索引节点缓存、目录项缓存等多种缓存机制，以减少对下层块设备的直接调用。 同样，为了优化块设备的访问效率，会使用缓冲区，来缓存块设备的数据。

  ### 磁盘性能指标

  * 使用率，是指磁盘处理 I/O 的时间百分比。过高的使用率（比如超过 80%），通常意味着磁盘 I/O 存在性能瓶颈。这里要注意的是，使用率只考虑有没有 I/O，而不考虑 I/O 的大小。换句话说，当使用率是 100% 的时候，磁盘依然有可能接受新的 I/O 请求。
  * 饱和度，是指磁盘处理 I/O 的繁忙程度。过高的饱和度，意味着磁盘存在严重的性能瓶颈。当饱和度为 100% 时，磁盘无法接受新的 I/O 请求。
  * IOPS（Input/Output Per Second），是指每秒的 I/O 请求数。
  * 吞吐量，是指每秒的 I/O 请求大小。
  * 响应时间，是指 I/O 请求从发出到收到响应的间隔时间。

  一般来说，我们在为应用程序的服务器选型时，要先对磁盘的 I/O 性能进行基准测试，以便可以准确评估，磁盘性能是否可以满足应用程序的需求。

  #### 性能测试工具

  磁盘I/O观测:

  iostat,指标来自`/proc/diskstats`

  ```
  tiandiwuji@tiandiwuji:~$ iostat -d -x 1
  Linux 4.15.0-43-generic (tiandiwuji) 	02/24/2019 	_x86_64_	(4 CPU)
  
  Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
  loop0            0.21    0.00      1.98      0.00     0.00     0.00   0.00   0.00   45.60    0.00   0.00     9.43     0.00   0.00   0.00
  loop1            0.21    0.00      0.66      0.00     0.00     0.00   0.00   0.00   52.34    0.00   0.00     3.14     0.00   0.00   0.00
  loop2            0.23    0.00      1.98      0.00     0.00     0.00   0.00   0.00   48.21    0.00   0.00     8.68     0.00   0.00   0.00
  loop3            0.22    0.00      1.99      0.00     0.00     0.00   0.00   0.00   50.56    0.00   0.00     9.19     0.00   0.00   0.00
  loop4            0.23    0.00      0.68      0.00     0.00     0.00   0.00   0.00   44.21    0.00   0.00     2.97     0.00   0.00   0.00
  loop5            0.26    0.00      2.01      0.00     0.00     0.00   0.00   0.00   17.12    0.00   0.00     7.79     0.00   0.00   0.00
  loop6            0.23    0.00      2.00      0.00     0.00     0.00   0.00   0.00   42.67    0.00   0.00     8.56     0.00   0.21   0.00
  loop7            0.23    0.00      0.68      0.00     0.00     0.00   0.00   0.00   22.15    0.00   0.00     2.92     0.00   0.00   0.00
  sda            117.60   20.69   4532.55   1940.82    25.48    60.22  17.81  74.43   34.83  323.08  10.78    38.54    93.79   5.27  72.92
  loop8            0.22    0.00      1.99      0.00     0.00     0.00   0.00   0.00   43.89    0.00   0.00     8.97     0.00   1.73   0.04
  loop9            0.20    0.00      0.65      0.00     0.00     0.00   0.00   0.00   22.47    0.00   0.00     3.21     0.00   0.00   0.00
  loop10           0.28    0.00      6.40      0.00     0.00     0.00   0.00   0.00   24.77    0.00   0.00    22.70     0.00   0.00   0.00
  loop11           0.32    0.00      6.37      0.00     0.00     0.00   0.00   0.00   52.07    0.00   0.00    19.67     0.00   7.85   0.25
  loop12           0.21    0.00      0.66      0.00     0.00     0.00   0.00   0.00   45.26    0.00   0.00     3.14     0.00   0.00   0.00
  loop13          50.28    0.00     56.41      0.00     0.00     0.00   0.00   0.00   27.08    0.00   1.08     1.12     0.00   0.74   3.74
  loop14           0.26    0.00      2.03      0.00     0.00     0.00   0.00   0.00   28.36    0.00   0.00     7.70     0.00   0.27   0.01
  loop15           0.23    0.00      2.00      0.00     0.00     0.00   0.00   0.00   33.26    0.00   0.00     8.76     0.00   0.00   0.00
  loop16           0.37    0.00      6.42      0.00     0.00     0.00   0.00   0.00   47.16    0.00   0.00    17.26     0.00   4.52   0.17
  loop17           0.28    0.00      6.41      0.00     0.00     0.00   0.00   0.00   28.26    0.00   0.00    23.22     0.00   0.17   0.00
  loop18           0.03    0.00      0.05      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     1.60     0.00   0.00   0.00
  ```

  指标含义:

  ![](https://static001.geekbang.org/resource/image/cf/8d/cff31e715af51c9cb8085ce1bb48318d.png)

  * %util  使用率
  * r/s+w/s IOPS
  * rKB/s + wKB/s  吞吐量
  * r_await + w_await 相应时间

  饱和度测试：饱和度通常没有简单的观测方法，不过可以将观测到的，平均请求队列长度或者读写请求完成的等待时间，和基准测试的结果（通过fio)进行对比，进行综合评估。

  #### 进程I/O观测

  iostat提供整体I/O性能数据，通过pidstat和 iotop可以观察进程I/O情况。

  ```
  tiandiwuji@tiandiwuji:~$ pidstat -d 1
  Linux 4.15.0-43-generic (tiandiwuji) 	02/24/2019 	_x86_64_	(4 CPU)
  02:32:07 AM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
  02:32:08 AM     0       335     -1.00     -1.00     -1.00       2  jbd2/sda1-8
  ```

  iotop:

  ```
  Total DISK READ :       0.00 B/s | Total DISK WRITE :      42.37 K/s
  Actual DISK READ:       0.00 B/s | Actual DISK WRITE:      61.63 K/s
     TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND                                                                                                                                     
     335 be/3 root        0.00 B/s   42.37 K/s  0.00 %  0.06 % [jbd2/sda1-8]
       1 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % init splash
       2 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [kthreadd]
       4 be/0 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [kworker/0:0H]
       6 be/0 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [mm_percpu_wq]
       7 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [ksoftirqd/0]
       8 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [rcu_sched]
       9 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [rcu_bh]
      10 rt/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [migration/0]
      11 rt/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [watchdog/0]
  ```

  从这个输出，前两行分别表示，进程的磁盘读写大小总数和磁盘真实的读写大小总数。因为缓存、缓冲区、I/O 合并等因素的影响，它们可能并不相等。

  剩下的部分，则是从各个角度来分别表示进程的 I/O 情况，包括线程 ID、I/O 优先级、每秒读磁盘的大小、每秒写磁盘的大小、换入和等待 I/O 的时钟百分比等。

  ## 如何找出狂打日志的“内鬼”

  1. top观察cpu和内存的使用情况
  2. iostat 查看I/O使用情况
  3. pidstat -d 1 查看每个进程的I/O使用情况
  4. strace -p pid 查看系统调用
  5. lsof -p pid 查看进程打开了那些文件，包括目录，块设备，动态库，网络套接字等

  ```
  import string
  import signal
  import time
  
  from logging.handlers import RotatingFileHandler
  
  logger = logging.getLogger(__name__)
  logger.setLevel(level=logging.INFO)
  rHandler = RotatingFileHandler(
      "/tmp/logtest.txt", maxBytes=1024 * 1024 * 1024, backupCount=1)
  rHandler.setLevel(logging.INFO)
  formatter = logging.Formatter(
      '%(asctime)s - %(name)s - %(levelname)s - %(message)s')
  rHandler.setFormatter(formatter)
  logger.addHandler(rHandler)
  
  
  def set_logging_info(signal_num, frame):
      '''Set loging level to INFO when receives SIGUSR1'''
      logger.setLevel(logging.INFO)
  
  
  def set_logging_warning(signal_num, frame):
      '''Set loging level to WARNING when receives SIGUSR2'''
      logger.setLevel(logging.WARNING)
  
  
  def get_message(N):
      '''Get message for logging'''
      return N * ''.join(
          random.choices(string.ascii_uppercase + string.digits, k=1))
  
  
  def write_log(size):
      '''Write logs to file'''
      message = get_message(size)
      while True:
          logger.info(message)
          time.sleep(0.1)
  
  
  signal.signal(signal.SIGUSR1, set_logging_info)
  signal.signal(signal.SIGUSR2, set_logging_warning)
  
  if __name__ == '__main__':
      msg_size = 300 * 1024 * 1024
      write_log(msg_size)
  ```

  该程序可以调整日志级别，发送-SIGUSR2信号即可。

  `kill -SIGUSR2 pid`

  ## 为什么磁盘I/O延迟很高

  filetop： 跟踪内核中文件的读写情况

  ```
  cd /usr/share/bcc/tools
  ./filetop -C
  ```

  opensnoop: 动态跟踪内核中的open系统调用

  ## 一个SQL查询需要15秒？



  ## Redis响应严重延迟
