## **3.2 Linux系统中的进程控制块**

&emsp;&emsp;操作系统为了对进程管理，就必须对每个进程在其生命周期内涉及的所有事情进行全面的描述。例如，进程当前处于什么状态，它的优先级是什么，它是正在CPU上运行还是因某些事件而被阻塞，给它分配了什么样的地址空间，允许它访问哪个文件等等。所有这些信息在内核中用一个结构体来描述——Linux中把对进程的描述结构叫做task_struct:

> 
> struct task_struct {
 
> …
> 
> … 

> ｝


&emsp;&emsp;传统上,这样的数据结构被叫做**进程控制块PCB**（process control blaock）。Linux中PCB是一个相当庞大的结构体，我们将它的所有域按其功能可分为一下几类：

（1）	状态信息－描述进程动态的变化，如就绪态，等待态，僵死态等。

（2）	链接信息－描述进程的亲属关系，如祖父进程，父进程，养父进程，子进程，兄进程，孙进程等。

（3）	各种标识符－用简单数字对进程进行标识，如进程标识符，用户标识符等

（4）	进程间通信信息－描述多个进程在同一任务上协作工作，如管道，消息队列，共享内存，套接字等

（5）	时间和定时器信息－描述进程在生存周期内使用CPU时间的统计、计费等信息等。

（6）	调度信息－描述进程优先级、调度策略等信息，如静态优先级，动态优先级，时间片轮转、高优先级优先以及多级反馈队列等调度策略。

（7）	文件系统信息－对进程使用文件情况进行记录，如文件描述符，系统打开文件表，用户打开文件表等。

（8）	虚拟内存信息－描述每个进程拥有的地址空间，也就是进程编译连接后形成的空间。

（9）	处理器环境信息－描述进程的执行环境(处理器的各种寄存器及堆栈等)，这是体现进程动态变化最主要的场景。

&emsp;&emsp;在进程的整个生命周期中，系统（也就是内核）总是通过PCB对进程进行控制的，也就是说，系统是根据进程的PCB而不是别的什么而感知进程的存在的。例如，当内核要调度某进程执行时，要从该进程的PCB中查出其运行状态和优先级；在某进程被选中投入运行时，要从其PCB中取出其处理机环境信息，恢复其运行现场；进程在执行过程中，当需要和与之合作的进程实现同步、通信或访问文件时，也要访问PCB。当进程因某种原因而暂停执行时，又需将其断点的处理机环境保存在PCB中。所以说，PCB是进程存在和运行的唯一标志。

&emsp;&emsp;当系统创建一个新的进程时，就为它建立一个PCB；进程结束时又收回其PCB，进程随之也消亡。PCB是内核中被频繁读写的数据结构，故应常驻内存。

&emsp;&emsp;进程的另外一个名字是任务（task）。Linux内核通常把进程也叫任务，在本书中我们交替使用这两个术语。另外，在Linux内核中，对进程和线程也不做明显的区别，也就是说，进程和线程的实现采取了同样的方式。

&emsp;&emsp;下面主要讨论PCB中进程的状态、标识符和进程间的父／子关系。

### **3.2.1 进程状态**

&emsp;&emsp;从操作系统的原理知道，进程一般有三种基本状态-执行，就绪和等待态，但是在具体操作系统的实现中，设计者根据具体需要可以设置不同的状态。在Linux的设计中，考虑到任一时刻在CPU上运行的进程最多只有一个，而准备运行的进程可能有若干个，为了管理上的方便，把就绪态和运行态合并为一个状态叫就绪态，这样系统把处于就绪态的进程放在一个队列中，调度程序从这个队列中选中一个进程投入运行。而等待态又被划分为两种，除此之外，还有暂停状态和僵死状态，这几个主要状态描述如下：

**就绪态（TASK_RUNNING）：**正在运行或准备运行，处于这个状态的所有进程组成就绪队列。

**睡眠（或等待）态：**分为浅度睡眠态和深度睡眠态

**浅度睡眠态（TASK_INTERRUPTIBLE）：**进程正在睡眠（被阻塞），等待资源有效时被唤醒，不仅如此，也可以由其他进程通过信号 或时钟中断唤醒。

**深度睡眠态（TASK_UNINTERRUPTIBLE）：** 与前一个状态类似，但其它进程发来的信号和时钟中断并不能打断它的熟睡。

**暂停状态（TASK_STOPPED）：**进程暂停执行，比如，当进程接收到如下信号后，进入暂停状态：

SIGSTOP-停止进程执行

SIGTSTP-从终端发来信号停止进程

SIGTTIN-来自键盘的中断

SIGTTOU-后台进程请求输出

**僵死状态（TASK_ZOMBIE）：**进程执行结束但尚未消亡的一种状态。此时，进程已经结束且释放大部分资源，但尚未释放其PCB。

图3.5 给出Linux进程状态的转换及其所调用的内核函数。

<div align=center>
<img src="3_5.png" />  
</div>


&emsp;&emsp;如图所示，通过fork（）创建的进程处于就绪状态，其PCB进入就绪队列。如果调度程序schedule()运行,则从就绪队列中选择一进程投入运行而占有CPU。在进程执行的过程中，因为输入输出等原因调用interruptible_sleep_on()或者sleep_on(),则进程进入浅度睡眠或者深度睡眠。因为进程进入睡眠状态放弃CPU，因此也调用了调度程序schedule()重新从就绪队列中调用一个进程运行。以此类推，读者可以自己理解图中调用其他函数的意义。

&emsp;&emsp;在taks_struct结构中（定义于shed.h），状态域定义为：

    Struct tast_struct{
       volatile long state;    /* -1 unrunnable, 0 runnable, >0 stopped */
    …
     ｝

&emsp;&emsp;其中volatile是一种类型修饰符，它告诉编译程序不必优化，从内存读取数据而不是寄存器，以确保状态的变化能及时地反映出来。

&emsp;&emsp;对每个具体的状态赋予一个常量,有些状态是在新的内核中增加的：

    #define TASK_RUNNING		0
    #define TASK_INTERRUPTIBLE	1
    #define TASK_UNINTERRUPTIBLE	2
    #define __TASK_STOPPED		4
    #define __TASK_TRACED		8/* 由调试程序暂停进程的执行 */
    /* in tsk->exit_state */
    #define EXIT_ZOMBIE		16
    #define EXIT_DEAD		32 /*最终状态，进程将被彻底删除，但需要父进程来回收*/
    /* in tsk->state again */
    #define TASK_DEAD		64  /*与EXIT_DEAD类似，但不需要父进程回收*/
    #define TASK_WAKEKILL		128/*接收到致命信号时唤醒进程，即使深度睡眠*/

&emsp;&emsp;也可以使用ps命令查看进程的状态

### **3.2.2 进程标识符**

&emsp;&emsp;每个进程有进程标识符、用户标识符、组标识符。

&emsp;&emsp;不管对内核还是普通用户来说，怎么用一种简单的方式识别不同的进程呢？这就引入了进程标识符（PID），每个进程都有一个唯一的标识符，内核通过这个标识符来识别不同的进程，同时，进程标识符PID也是内核提供给用户程序的接口，用户程序通过PID对进程发号施令。PID是32位的无符号整数，它被顺序编号：新创建进程的PID通常是前一个进程的PID加1。在Linux上允许的最大PID号是由变量pid_max来指定，可以在内核编译的配置界面里配置0x1000和0x8000两种值，即在4096以内或是32768以内。当内核在系统中创建进程的PID大于这个值时，就必须重新开始使用已闲置的PID号。

    #define PID_MAX_DEFAULT (CONFIG_BASE_SMALL ? 0x1000 : 0x8000)
    int pid_max = PID_MAX_DEFAULT;

&emsp;&emsp;这个最大值很重要，因为它实际上就是系统中允许同时存在的进程的最大数目。尽管最大值对于一般的桌面系统足够用了，但是大型服务器可能需要更多进程。这个值越小，转一圈就越快。如果确实需要的话，可以不考虑与老式系统的兼容，由系统管理员通过修改`/proc/sys/kernel/pid_max`来提高上限。可以通过cat命令查看系统pid_max的值。

    $ cat /proc/sys/kernel/pid_max 
    $32768

&emsp;&emsp;另外，每个进程都属于某个用户组。task_struct结构中定义有用户标识符UID（User Identifier）和组标识符GID（Group Identifier）。它们同样是简单的数字，这两种标识符用于系统的安全控制。系统通过这两种标识符控制进程对系统中文件和设备的访问。

### **3.2.3进程之间的亲属关系**

&emsp;&emsp;系统创建的进程具有父/子关系。因为一个进程能创建几个子进程，而子进程之间有兄弟关系。在PCB中引入几个域来表示这些关系。如前说述，进程1（init）是所有进程的祖先,系统中的进程形成一颗进程树。为了描述进程之间的父／子及兄弟关系，在进程的PCB中就要引入几个域。假设P表示一个进程，首先要有一个域描述它的父进程（parent）；其次，有一个域描述P的子进程，因为子进程不止一个，因此让这个域指向年龄最小的子进程（child）；最后，P可能有兄弟，于是用一个域描述P的长兄进程（old sibling）,一个域描述P的弟进程（younger sibling）
               										 
&emsp;&emsp;上面通过对进程状态、标识符及亲属关系的描述，我们可以把这些域描述如下：

    struct task_struct{
       volatile long state;   /*进程状态*/
       int pid,uid,gid； /*一些标识符*/
       struct task_struct *real_parent; /* 真正创建当前进程的进程，相当于亲生父亲*/
       struct task_struct *parent; /* 相当于养父*/
       struct list_head children;	/* 子进程链表 */
       struct list_head sibling;	/* 兄弟进程链表 */
       struct task_struct *group_leader;	/* 线程组的头进程 */
       …
       ｝

&emsp;&emsp;这里说明一点是，一个进程可能有两个父亲，一个为亲生父亲，一个为养父。因为父进程有可能在子进程之前销毁，就得给子进程重新找个养父，但大多数情况下，生父和养父是相同的，如图3.6

<div align=center>
<img src="3_6.png" />  
</div>

 
### **3.2.4进程控制块的存放**

&emsp;&emsp;当创建一个新的进程时，内核首先要为其分配一个PCB（task_struct结构）。那么，这个PCB存放在何处？怎样找到PCB？

&emsp;&emsp;每当进程从用户态进入内核态后都要使用栈，这个栈叫做进程的内核栈。当进程一进入内核态，CPU就自动设置该进程的内核栈，这个栈位于内核的数据段。为了节省空间，Linux把内核栈和一个紧挨近PCB的小数据结构thread_info放在一起，占用8KB的内存区，如图3.7所示：

<div align=center>
<img src="3_7.png" />  
</div>


&emsp;&emsp;在Intel系统中，栈起始于末端，并朝这个内存区开始的方向增长。从用户态刚切换到内核态以后，进程的内核栈总是空的，因此，堆栈寄存器esp直接指向这个内存区的顶端。 在图3.7中，从用户态切换到内核态后，只要把数据写进栈中，堆栈寄存器的值就超箭头方向递减，p表示thread_info的起始地址。而task是thread_info的第一个数据项，所以只要找到thread_info就很容易找到当前运行的task_struct了。

&emsp;&emsp;C语言使用下列的联合结构表示这样一个混合结构： 

    union thread_union {
	    struct thread_info thread_info;
	    unsigned long stack[THREAD_SIZE/sizeof(long)];/*大小一般是8KB，但也可以配置为4KB。本书以8KB叙述。*/
    };

&emsp;&emsp;从这个结构可以看出，内核栈占8KB的内存区。实际上，进程的PCB所占的内存是由内核动态分配的，更确切地说，内核根本不给PCB分配内存，而仅仅给内核栈分配8K的内存，并把其中的一部分让给PCB使用。
 
&emsp;&emsp;在x86上，其中 thread_info结构在文件<asm/thread_info.h>中定义如下：

    struct thread_info{
       struct task_sturct  *task;
     struct exec_domain  *exec_domain;
       …
    };

&emsp;&emsp;thread_info结构并不代表线程相关信息，而是和硬件关系更紧密的一些数据。thread_info 和task_struct结构中都有一个域指向对方，因此是一一对应的关系。之所以定义一个thread_info结构，原因之一可能是，进程控制块的所有成员中最被频繁引用的是thread_info。另一个原因可能是，随着Linux版本的变化，进程控制块的内容越来越多，所需空间越来越大，这样就使得留给内核堆栈的空间变小，因此把部分进程控制块的内容移出这个空间，只保留访问频繁的thread_info。

### **3.2.5 当前进程**

&emsp;&emsp;从效率的观点来看，刚才所讲的thread_info结构与内核态堆栈放在一起的最大好处是，内核很容易从esp寄存器的值获得当前在CPU上正在运行的thread_info结构的地址。事实上，如果thread_union 结构长度是8K，则内核屏蔽掉esp的低13位有效位就可以获得thread_info结构的基地址；这由 current_thread_info( )函数来完成，它产生如下一些汇编指令：

    movl $0xffffe000, %ecx 
    andl %esp, %ecx 
    movl %ecx, p 

&emsp;&emsp;这三条指令执行以后， p就指向进程的thread_info结构的指针。

&emsp;&emsp;进程最常用的是其task_struct结构的地址而不是thread_info结构的地址，为了获得当前在CPU上运行进程的PCB指针，内核要调用current宏，该宏本质上等价于`current_thread_info()->task`。

&emsp;&emsp;可以把current作为全局变量来使用，例如，current->pid返回当前正在执行的进程的标识符。对于当前进程，可以通过下面的代码获得其父进程的PCB

> struct  task_struct *my_parent=current->parent;


