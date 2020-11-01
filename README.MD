# 操作系统

## 1. 综述

### 1.1 系统组成

![](./images/os-06.png)



### 1.2 命令行

#### 文件目录

```SHELL
# ls -l
drwxr-xr-x 6 root root    4096 Oct 20  2017 apt
-rw-r--r-- 1 root root     211 Oct 20  2017 hosts
```

第一个字段的第一个字符是文件类型。如果是“-”，表示普通文件；如果是 d，就表示目录。第一个字段剩下的 9 个字符是权限位（access permission bits）。3 个一组，**每一组 rwx 表示“读（read）”“写（write）”“执行（execute）”**。

第二个字段是硬链接（hard link）数目

#### 服务运行

Ubuntu 中，通过 apt-get install mysql-server 的方式安装 MySQL，**在 /lib/systemd/system 目录下会创建一个 [*].service 的配置文件**

CentOS 中，在 **/usr/lib/systemd/system** 目录下，创建一个 [*].service 的配置文件

+ 通过命令systemctl start mysql启动 MySQL
+ 通过systemctl enable mysql设置开机启动。

![](./images/os-07.png)



### 1.3 系统调用

#### 进程管理

创建一个新的进程，需要一个老的进程调用 fork 来实现，老的进程叫作父进程（Parent Process），新的进程叫作子进程（Child Process）。对于 fork 系统调用的返回值

+ 如果当前进程是子进程，就返回 0
+ 如果当前进程是父进程，就返回子进程的进程号

子进程，需要请求另一个系统调用**execve**来执行另一个程序

系统调用waitpid，父进程可以调用它，将子进程的进程号作为参数传给它，这样父进程就知道子进程是否运行完成和成功。



#### 内存管理

代码段是放程序代码的这部分。数据段是放进程运行中产生数据的。

当分配的内存数量比较小的时候，**使用 brk，会和原来的堆的数据连在一起。**

当分配的内存数量比较大的时候，**使用 mmap，会重新划分一块区域。**



#### 文件管理

文件的操作

+ 打开文件使用open，close关闭这个文件
+ 创建没有的文件可以使用create
+ 打开文件以后，可以使用lseek跳到文件的某个位置
+ 对文件的内容进行读写，读的系统调用是read，写是write

**Linux 里有一个特点，那就是一切皆文件**

+ 启动一个进程，需要一个程序文件，这是一个**二进制文件**
+ 加载一些配置文件，打印一些日志，这是**文本文件**
+ 日志打印到交互控制台上，是**标准输出 stdout 文件**
+ 这个进程的输出可以作为另一个进程的输入，这种方式称为管道，**管道**也是一个文件。
+ 进程可以通过网络和其他进程进行通信，**建立的 Socket，也是一个文件。**
+ 进程需要访问外部设备，**设备也是一个文件。**
+ 文件都被存储在文件夹里面，其实**文件夹也是一个文件。**
+ 进程运行起来，要想看到进程运行的情况，**会在 /proc 下面有对应的进程号，还是一系列文件。**

**Linux 都会分配一个文件描述符（File Descriptor）**，这是一个整数。有了这个文件描述符，我们就可以使用系统调用，查看或者干预进程运行的方方面面。



#### 信号处理

SIGKILL：用于终止一个进程的信号

SIGSTOP：用于中止一个进程的信号



#### 中介Glibc

Glibc 为程序员提供丰富的 API，最重要的是**封装了操作系统提供的系统服务，即系统调用的封装。**

![](./images/os-08.png)



## 2. 系统初始化

### 2.1 x86架构

核心是CPU，**CPU和其他设备通过总线连接**

![](./images/os-09.png)

#### CPU 架构

![](./images/os-10.png)

CPU包括三个部分，运算单元、数据单元和控制单元。

+ 运算单元只管算，例如做加法、做位移等等。
+ 数据单元根据数据的地址，从数据段里读到数据寄存器里。
+ 控制单元获得下一条指令，然后执行这条指令。这个指令会指导**运算单元取出数据单元中的某几个数据，计算出个结果，然后放在数据单元的某个地方。**
  + **指令指针寄存器**存放的是下一条指令在内存中的地址
  + **指令寄存器**拿进来的代码段的指令

总线分为两类

+ 地址总线：内存中哪个位置的数据
+ 数据总线：传输数据

x86实现了开放、统一、兼容规范



#### 8086 CPU

数据单元：8 个 16 位的通用寄存器。e.g. AX、BX、CX、DX、SP、BP、SI、DI

控制单元

+ IP 寄存器： 指令指针寄存器
+ CS 代码段寄存器（Code Segment Register）可以找到代码在内存中的位置
+ DS 数据段的寄存器（Data Segment Register），可以找到数据在内存中的位置
+ SS 栈寄存器（Stack Register）, 函数调用

IP寄存器和通用寄存器都是 16 位的。CS 和 DS 都是 16 位的，但是总线是20位，所以CS 和 DS 中的值**左移 4 位，**变成 20 位的，**加上 16 位的偏移量得到最终 20 位的数据地址**



#### 32位 CPU

通用寄存器: 扩展到 8 个 32 位的

CS/DS/SS/ES: 段的起始地址放在内存的某个地方。如同表格，表格中的一项一项是段描述符（Segment Descriptor）。**段描述符才是真正的段的起始地址。**段寄存器里面保存的是选择子（Selector），是指向这个表格中的某一项。

+ 实模式：段寄存器指向内存地址

+ 保护模式：段寄存器指向表格中的段描述符

CPU在启动时逐渐从实模式切换到保护模式。



### 2.2 内核加载

![](./images/os-11.png)

#### BIOS

ROM（Read Only Memory，只读存储器）保存BIOS（Basic Input and Output System，基本输入输出系统）。x86 系统中，将 1M 空间最上面的 0xF0000 到 0xFFFFF 这 64K 映射给 ROM， BIOS开始初始化工作。

+ 检查系统的硬件
+ 建立一个中断向量表和中断服务程序，e.g. 鼠标和键盘操作都是通过中断



#### bootloader

Linux 里面有Grub2，全称 Grand Unified Bootloader Version 2，用于系统启动的。

+ boot.img 从硬盘加载到内存中的 0x7c00 来运行
+ grub2安装boot.img，由 boot.S 编译而成，一共 512 字节，正式安装到启动盘的第一个扇区。通常称为 MBR（Master Boot Record，主引导记录 / 扇区）。
+ boot.img 先加载的是 core.img 的第一个扇区， boot.img 将控制权交给 diskboot.img
+ diskboot.img将 core.img 的其他部分加载进来



#### 保护模式

+ 打开 Gate A20，第 21 根地址线的控制线。
+ **启用分段**，就是在内存里面建立段描述符表，将寄存器里面的段寄存器变成段选择子，指向某个段描述符，这样就能实现不同进程的切换了
+ **启动分页**，能够管理的内存变大了，就需要将内存分成相等大小的块
+ 解压lzma_decompress.img
+ 解压kernel.img，运行grub2 keneral
+ **Grub2 启动 linux内核**



### 2.3 内核初始化

![](./images/os-12.png)

#### 初始化管理子系统

+ 创建的0 号进程
+ 初始化系统调用模块，包括中断们用于输入输出
+ 初始化内存管理模块
+ 初始化基于内存的文件系统，文件系统为了兼容各种各样的文件系统，需要将文件的相关数据结构和操作抽象出来，形成一个抽象层对上提供统一的接口，这个抽象层就是 VFS（Virtual File System），虚拟文件系统。



#### 初始化1号/2号进程

rest_init 的第一工作是用 kernel_thread(kernel_init, NULL, CLONE_FS) 创建1号进程管理用户进程

rest_init 的第一工作是用 kernel_thread(kernel_init, NULL, CLONE_FS) 创建2号进程管理内核进程



#### 内核态/用户态切换

![](./images/os-13.png)

内核态（Kernel Mode）：能够访问关键资源的代码放在 Ring0

用户态（User Mode）: 普通的程序代码放在 Ring3

![](./images/os-14.png)

### 2.4 系统调用

#### 32位系统调用

+ 将请求参数放在寄存器里面
+ 根据系统调用名称，得到系统调用号，放在寄存器 eax 里面
+ 执行 ENTER_KERNEL
+ 从 eax 里面取出来系统调用号，然后根据系统调用号，在系统调用表sys_call_table中找到相应的函数进行调用，并将寄存器中保存的参数取出来，作为函数参数
+ iret 指令将原来用户态保存的现场恢复回来

![](./images/os-15.png)

#### 64位系统调用

+ 将请求参数放在寄存器里面
+ 根据系统调用名称，得到系统调用号，放在寄存器 eax 里面
+ 执行 syscall 进入内核， syscall是特殊模块寄存器，为CPU执行系统调用
+ 从rax 里面拿出系统调用号，然后根据系统调用号，在系统调用表 sys_call_table 中找到相应的函数进行调用，并将寄存器中保存的参数取出来，作为函数参数
+ sysretq 指令将原来用户态保存的现场恢复回来

![](./images/os-16.png)

#### 系统调用表

32 位的系统调用表定义在 arch/x86/entry/syscalls/syscall_32.tbl 文件里

64 位的系统调用定义在另一个文件 arch/x86/entry/syscalls/syscall_64.tbl 里

+ 第一列的数字是系统调用号
+ 第三列是系统调用的名字
+ 第四列是系统调用在内核的实现函数。

```assembly
2  common  open      sys_open
```



## 3. 进程管理

### 3.1 进程

#### ELF二进制格式

process.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
extern int create_process (char* program, char** arg_list);

int create_process (char* program, char** arg_list) {
    pid_t child_pid;
    child_pid = fork ();
    if (child_pid != 0)
        return child_pid;
    else {
        execvp (program, arg_list);
        abort ();
    }
}
```

createprocess.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
extern int create_process (char* program, char** arg_list);

int main () {
    char* arg_list[] = {
        "ls",
        "-l",
        "/etc/yum.repos.d/",
        NULL
    };
    create_process ("ls", arg_list);
    return 0;
}
```



#### 

源代码被编译成二进制文件，这个格式称为 ELF（Executeable and Linkable Format，可执行与可链接格式）。

![](./images/os-17.png)

##### 可重定位文件

![](./images/os-18.png)

ELF 文件的头是用于描述整个文件的

+ .text：放编译好的二进制可执行代码
+ .rodata：只读数据，例如字符串常量、const 的变量
+ .data：已经初始化好的全局变量
+ .bss：未初始化全局变量，运行时会置 0
+ .symtab：符号表，记录的则是函数和变量
+ .strtab：字符串表、字符串常量和变量名

编译好的代码和变量，将来加载到内存里面的时候，都是要加载到一定位置的。**.o 文件，不是一个可以直接运行的程序**，这里面只是部分代码片段。**create_process 函数不确定位置，需要可以重新定位代码片段。**.rel.text, .rel.data 就与重定位有关。



##### 可执行文件 - 静态链接

![](./images/os-19.png)

把process.o创建成静态链接库。**静态链接库会代码和变量的 section 都合并了**，因而程序运行的时候不依赖于这个库是否存在

```shell
ar cr libstaticprocess.a process.o
gcc -o staticcreateprocess createprocess.o -L. -libstaticprocess
```



##### 共享对象文件 - 动态链接

动态链接库（Shared Libraries）是多个对象文件的重新组合，可被多个程序共享。当运行这个程序的时候，首先寻找动态链接库，然后加载它。默认情况下，系统在 **/lib， /usr/lib 和 LD_LIBRARY_PATH 环境变量**

```SHELL
gcc -shared -fPIC -o libdynamicprocess.so process.o
gcc -o dynamiccreateprocess createprocess.o -L. -ldynamicprocess
```

+ 多了一个.interp 的 Segment，这里面是 **ld-linux.so，这是动态链接器**，运行时的链接动作都是它做的
+ ELF 文件中还多了两个 section
  + 一个是.plt，过程链接表（Procedure Linkage Table，PLT）
  + 一个是.got.plt，全局偏移量表（Global Offset Table，GOT）

**调用过程**

+ PLT 里面建立一项 PLT[x]。本地的代理，在二进制程序里面，不直接调用 create_process 函数，而是调用 PLT[x]里面的代理代码，这个代理代码会在运行的时候找真正的 create_process 函数
+ GOT[y]一项是运行时 create_process 函数在内存中真正的地址
+ GOT[y]没有真正的地址，它又回调 PLT
+ PLT 这个时候会转而调用 PLT[0]，PLT[0]转而调用 GOT[2]，这里面是 ld-linux.so 的入口函数



##### 内核数据结构

在内核中，linux_binfmt数据结构定义加载二进制文件的方法

```c
struct linux_binfmt {
        struct list_head lh;
        struct module *module;
        int (*load_binary)(struct linux_binprm *);
        int (*load_shlib)(struct file *);
        int (*core_dump)(struct coredump_params *cprm);
        unsigned long min_coredump;     /* minimal dump size */
} __randomize_layout;
```

```c
static struct linux_binfmt elf_format = {
        .module         = THIS_MODULE,
        .load_binary    = load_elf_binary,
        .load_shlib     = load_elf_library,
        .core_dump      = elf_core_dump,
        .min_coredump   = ELF_EXEC_PAGESIZE,
};
```



#### 进程树

![](./images/os-20.png)

![](./images/os-21.png)

#### Fork

```c
long _do_fork(unsigned long clone_flags,
        unsigned long stack_start,
        unsigned long stack_size,
        int __user *parent_tidptr,
        int __user *child_tidptr,
        unsigned long tls)
{
  struct task_struct *p;
  int trace = 0;
  long nr;
  p = copy_process(clone_flags, stack_start, stack_size,
       child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
    wake_up_new_task(p);

  } 
```



##### 创建

###### dup_task_struct

+ 创建一个 task_struct 结构
+ 创建内核栈，分配一个THREAD_SIZE 的内存空间
+ 复制task_struct
+ 设置thread_info

###### copy_creds

+ 创建一个新的 struct cred 结构

###### 重设统计量

+ utime
+ Start_time

###### sched_fork

+ 初始化 sched_entity，将exec_start、sum_exec_runtime、prev_sum_exec_runtime、vruntime 都设为 0
+ 设置进程的状态 p->state = TASK_NEW
+ 初始化优先级
+ 设置调度类
+ 调用调度类的 task_fork 函数, 记当前运行的进程 TIF_NEED_RESCHED

###### copy_files/copy_fs

+ 创建 files_struct, 拷贝文件描述符数组 fdtable
+ 创建 fs_struct, 复制一个进程的目录信息

###### copy_singhand

+ 创建 sighand_struct， 维护信号处理函数

###### copy_mm

+ 创建mm_struct 结构

###### 分配 pid，设置 tid，group_leader

##### 唤醒

将进程的状态设置为 TASK_RUNNING

调用 enqueue_task

+ 更新运行的统计量
+ 将 sched_entity 加入到红黑树

调用check_preempt_wakeup， 在fork系统调用返回时抢占

![](./images/os-36.jpeg)

### 3.2 线程

#### 生命周期

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define NUM_OF_TASKS 5

void *downloadfile(void *filename)
{
   sleep(10);
   long downloadtime = rand()%100;
   printf("I finish downloading the file within %d minutes!\n", downloadtime);
   pthread_exit((void *)downloadtime);
}

int main(int argc, char *argv[])
{
   char files[NUM_OF_TASKS][20]={"file1.avi","file2.rmvb","file3.mp4","file4.wmv"};
   
   int rc;
   int t;
   int downloadtime;
 
   // 声明线程
   pthread_t threads[NUM_OF_TASKS];
   // 声明线程属性 
   pthread_attr_t thread_attr;
   // 初始化线程属性
   pthread_attr_init(&thread_attr);
   pthread_attr_setdetachstate(&thread_attr,PTHREAD_CREATE_JOINABLE);

   for(t=0;t<NUM_OF_TASKS;t++){
     printf("creating thread %d, please help me to download %s\n", t, files[t]);
     
     // 创建线程参数
     // 线程对象，线程属性，线程执行的方法， 线程执行的方法的参数
     rc = pthread_create(&threads[t], &thread_attr, downloadfile, (void *)files[t]);
     if (rc){
       printf("ERROR; return code from pthread_create() is %d\n", rc);
       exit(-1);
     }
   }

   pthread_attr_destroy(&thread_attr);
   for(t=0;t<NUM_OF_TASKS;t++){
     pthread_join(threads[t],(void**)&downloadtime);
     printf("Thread %d downloads the file %s in %d minutes.\n",t,files[t],downloadtime);
   }
   pthread_exit(NULL);
}
```

![](./images/os-22.png)

#### 线程数据

+ 线程栈上的本地数据
+ 整个进程里共享的全局数据
+ 线程私有数据（Thread Specific Data）: key 一旦被创建，所有线程都可以访问它，**但各线程可根据自己的需要往 key 中填入不同的值**，这就相当于提供了一个同名而不同值的全局变量。

```C
int pthread_key_create(pthread_key_t *key, void (*destructor)(void*))
int pthread_setspecific(pthread_key_t key, const void *value)
void *pthread_getspecific(pthread_key_t key)
```

#### 数据保护

操作共享资源的时候，还是需要抢互斥锁，因为可能很多人都受到了通知，都来访问了，所以条件变量和互斥锁是配合使用的。

+ Mutex，首先要使用 pthread_mutex_init 函数初始化这个 mutex
+ pthread_mutex_lock() 是去抢那把锁的函数，
  + 如果抢到了就可以执行下一行程序，对共享变量进行访问
  + 如果没抢到，就被阻塞在那里等待。
+ pthread_mutex_unlock 释放锁
+ 最终调用 pthread_mutex_destroy 销毁掉这把锁。

![](./images/os-23.png)

```C

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define NUM_OF_TASKS 3
#define MAX_TASK_QUEUE 11

char tasklist[MAX_TASK_QUEUE]="ABCDEFGHIJ";
int head = 0;
int tail = 0;

int quit = 0;

pthread_mutex_t g_task_lock;
pthread_cond_t g_task_cv;

// 消费者代码
void *coder(void *notused)
{
  pthread_t tid = pthread_self();

  while(!quit){
		// 获取mutex lock
    pthread_mutex_lock(&g_task_lock);
    while(tail == head){
      if(quit){
        pthread_mutex_unlock(&g_task_lock);
        pthread_exit((void *)0);
      }
      printf("No task now! Thread %u is waiting!\n", (unsigned int)tid);
      // 持续检查任务队列是否为空，如果为空就继续等待，等待的时候会释放锁
      pthread_cond_wait(&g_task_cv, &g_task_lock);
      printf("Have task now! Thread %u is grabing the task !\n", (unsigned int)tid);
    }
    
    // 执行一下代码如果任务队列不为空，操作共享变量
    char task = tasklist[head++];
    // 释放锁
    pthread_mutex_unlock(&g_task_lock);
    printf("Thread %u has a task %c now!\n", (unsigned int)tid, task);
    sleep(5);
    printf("Thread %u finish the task %c!\n", (unsigned int)tid, task);
  }

  pthread_exit((void *)0);
}

int main(int argc, char *argv[])
{
  pthread_t threads[NUM_OF_TASKS];
  int rc;
  int t;

  pthread_mutex_init(&g_task_lock, NULL);
  pthread_cond_init(&g_task_cv, NULL);

  for(t=0;t<NUM_OF_TASKS;t++){
    rc = pthread_create(&threads[t], NULL, coder, NULL);
    if (rc){
      printf("ERROR; return code from pthread_create() is %d\n", rc);
      exit(-1);
    }
  }

  sleep(5);

  for(t=1;t<=4;t++){
    pthread_mutex_lock(&g_task_lock);
    tail+=t;
    printf("I am Boss, I assigned %d tasks, I notify all coders!\n", t);
    //唤醒所有线程
    pthread_cond_broadcast(&g_task_cv);
    pthread_mutex_unlock(&g_task_lock);
    sleep(20);
  }

  pthread_mutex_lock(&g_task_lock);
  quit = 1;
  pthread_cond_broadcast(&g_task_cv);
  pthread_mutex_unlock(&g_task_lock);

  pthread_mutex_destroy(&g_task_lock);
  pthread_cond_destroy(&g_task_cv);
  pthread_exit(NULL);
}
```

![](./images/os-24.png)

![](./images/os-25.png)



#### 创建

##### 用户态创建

创建pthread 结构， 调用ALLOCATE_STACK

+ 线程栈是在进程的堆里面创建的， 申请一块内存空间，如果没有，就调用__mmap 创建一块新的
+ 线程栈也是自顶向下生长的，还记得每个线程要有一个 pthread 结构
+ 填充pthread 这个结构里面的成员变量 stackblock、stackblock_size、guardsize、specific等全局变量
+ 线程栈放到 stack_used 链表中，其实管理线程栈总共有两个链表，一个是 stack_used，也就是这个栈正被使用；另一个是 stack_cache，缓存起来暂时不适用的线程

##### 内核态创建

###### 标志位设定

+ copy_files： files_struct 引用计数加一
+ copy_fs：fs_struct 的用户数加一
+ copy_sighand： sighand_struct 引用计数加一
+ copy_signal：直接返回
+ copy_mm：直接指向了原来的 mm_struc

###### 设置亲缘关系

+ 设置group_leader 和 tgid

###### 信号的处理

+ 整个进程里的所有线程共享一个 shared_pending

##### 用户态执行

回到用户态执行start_thread

```C
#define START_THREAD_DEFN \
  static int __attribute__ ((noreturn)) start_thread (void *arg)

START_THREAD_DEFN
{
    struct pthread *pd = START_THREAD_SELF;
    /* Run the code the user provided.  */
    THREAD_SETMEM (pd, result, pd->start_routine (pd->arg));
    /* Call destructors for the thread_local TLS variables.  */
    /* Run the destructor for the thread-local data.  */
    __nptl_deallocate_tsd ();
    if (__glibc_unlikely (atomic_decrement_and_test (&__nptl_nthreads)))
        /* This was the last thread.  */
        exit (0);
    __free_tcb (pd);
    __exit_thread ();
}
```

线程执行完后调用 __deallocate_stack 来释放整个线程栈，这个线程栈要从当前使用线程栈的列表 stack_used 中拿下来，放到缓存的线程栈列表 stack_cache 中。

![](./images/os-37.png)



### 3.3 任务

Linux 里面，无论是进程，还是线程，到了内核里面，我们统一都叫任务（Task），由一个统一的结构 task_struct 进行管理。

**进程**，那 pid 是自己，tgid 是自己，group_leader 指向的还是自己

**线程**，那 pid 是自己，tgid 就是进程的主线程的 pid，group_leader 指向的就是进程的主线程。

#### 信号处理

```C
/* Signal handlers: */
struct signal_struct    *signal;
struct sighand_struct    *sighand;
sigset_t      blocked;
sigset_t      real_blocked;
sigset_t      saved_sigmask;
struct sigpending    pending;
unsigned long      sas_ss_sp;
size_t        sas_ss_size;
unsigned int      sas_ss_flags;
```

+ blocked: 信号被阻塞暂不处理
+ pending: 信号尚等待处理
+ sighand: 正在通过信号处理函数进行处理



#### 任务状态

![](./images/os-26.png)

+ **TASK_RUNNING**： 并不是说进程正在运行，而是表示进程在时刻准备运行的状态
+ **TASK_INTERRUPTIBLE**：可中断的**睡眠状态**，信号可以唤醒进程
+ **TASK_UNINTERRUPTIBLE**：不可中断的**睡眠状态**，线程不能被唤醒
+ **TASK_KILLABLE**：可以终止的新**睡眠状态**，可以响应致命信号
+ **TASK_STOPPED**：进程接收到 SIGSTOP、SIGTTIN、SIGTSTP 或者 SIGTTOU 信号之后进入该状态
+ **TASK_TRACED**：进程被 debugger 等进程监视，进程执行被调试程序所停止
+ **EXIT_ZOMBIE**：进程要结束
+ **EXIT_DEAD**：最终状态



#### 其他状态标志

```C
#define PF_EXITING    0x00000004
#define PF_VCPU      0x00000010
#define PF_FORKNOEXEC    0x00000040
```

+ PF_EXITING 表示正在退出
+ PF_VCPU 表示进程运行在虚拟 CPU 上
+ PF_FORKNOEXEC 表示 fork 完了，还没有 exec。



#### 运行统计信息

```C
u64        utime;//用户态消耗的CPU时间
u64        stime;//内核态消耗的CPU时间
unsigned long      nvcsw;//自愿(voluntary)上下文切换计数
unsigned long      nivcsw;//非自愿(involuntary)上下文切换计数
u64        start_time;//进程启动时间，不包含睡眠时间
u64        real_start_time;//进程启动时间，包含睡眠时间
```



#### 亲缘关系

```C
struct task_struct __rcu *real_parent; /* real parent process */
struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */
struct list_head children;      /* list of my children */
struct list_head sibling;       /* linkage in my parent's children list */
```

+ parent 指向其父进程。当它终止时，必须向它的父进程发送信号
+ children 表示链表的头部。链表中的所有元素都是它的子进程
+ sibling 用于把当前进程插入到兄弟链表中

![](./images/os-27.png)

#### 权限控制

```C
/* Objective and real subjective task credentials (COW): */
const struct cred __rcu         *real_cred;
/* Effective (overridable) subjective task credentials (COW): */
const struct cred __rcu         *cred;
```

+ real_cred 就是说明谁能操作我这个进程
+ cred 就是说明我这个进程能够操作谁

```C

struct cred {
......
        kuid_t          uid;            /* real UID of the task */
        kgid_t          gid;            /* real GID of the task */
        kuid_t          suid;           /* saved UID of the task */
        kgid_t          sgid;           /* saved GID of the task */
        kuid_t          euid;           /* effective UID of the task */
        kgid_t          egid;           /* effective GID of the task */
        kuid_t          fsuid;          /* UID for VFS ops */
        kgid_t          fsgid;          /* GID for VFS ops */
......
        kernel_cap_t    cap_inheritable; /* caps our children can inherit */
        kernel_cap_t    cap_permitted;  /* caps we're permitted */
        kernel_cap_t    cap_effective;  /* caps we can actually use */
        kernel_cap_t    cap_bset;       /* capability bounding set */
        kernel_cap_t    cap_ambient;    /* Ambient capability set */
......
} __randomize_layout;
```



##### 用户和用户组权限

+ euid/egid，是 effective user/group id。当这个进程要操作消息队列、共享内存、信号量等对象的时候，其实就是在比较这个用户和组是否有权限。
+ fsuid/fsgid，是 filesystem user/group id。这个是对文件操作会审核的权限。



可以通过 chmod u+s program 命令，给这个游戏程序设置 set-user-ID 的标识位，把游戏的权限变成 rwsr-xr-x。这个时候，用户 A 再启动这个游戏的时候，创建的进程 uid 当然还是用户 A，但是 euid 和 fsuid 就不是用户 A 了，因为看到了 set-user-id 标识，**就改为文件的所有者的 ID，也就是说，euid 和 fsuid 都改成用户 B 了**。

![](./images/os-28.png)

##### capabilities

```C
#define CAP_CHOWN            0
#define CAP_KILL             5
#define CAP_NET_BIND_SERVICE 10
#define CAP_NET_RAW          13
#define CAP_SYS_MODULE       16
#define CAP_SYS_RAWIO        17
#define CAP_SYS_BOOT         22
#define CAP_SYS_TIME         25
#define CAP_AUDIT_READ          37
#define CAP_LAST_CAP         CAP_AUDIT_READ
```

+ **cap_permitted** 表示进程能够使用的权限。
+ **cap_effective** 表示进程真正可以使用的权限
+ **cap_bset** 是 capability bounding set，是系统中所有进程允许保留的权限。如果这个集合中不存在某个权限，那么系统中的所有进程都没有这个权限
+ **cap_ambient** 非 root 用户进程使用 exec 执行一个程序的时候，如何保留权限的问题。当执行 exec 的时候，cap_ambient 会被添加到 cap_permitted 中，同时设置到 cap_effective 中



#### 用户态函数栈

##### 32 位操作系统

+ ESP（Extended Stack Pointer）是栈顶指针寄存器，入栈操作 Push 和出栈操作 Pop 指令，会自动调整 ESP
+ EBP（Extended Base Pointer），是栈基地址指针寄存器
+ 参数压入栈顶
+ 返回值会保存在 EAX 寄存器中

![](./images/os-29.png)

##### 64 位操作系统

+ RSP 是栈顶指针寄存器，入栈操作 Push 和出栈操作 Pop 指令，会自动调整 ESP
+ RBP是栈基地址指针寄存器
+ rdi、rsi、rdx、rcx、r8、r9 这 6 个寄存器用于传递存储函数调用时的 6 个参数。如果超过 6 的时候，还是需要放到栈里面。

![](./images/os-30.png)

#### 内核态函数栈

这段空间的最低位置，是一个 thread_info 结构。这个结构是对 task_struct 结构的补充。因为 task_struct 结构庞大但是通用，不同的体系结构就需要保存不同的东西，所以往往与体系结构有关的，都放在 thread_info 里面。

![](./images/os-31.png)

```C
/*
 * TOP_OF_KERNEL_STACK_PADDING reserves 8 bytes on top of the ring0 stack.
 * This is necessary to guarantee that the entire "struct pt_regs"
 * is accessible even if the CPU haven't stored the SS/ESP registers
 * on the stack (interrupt gate does not save these registers
 * when switching to the same priv ring).
 * Therefore beware: accessing the ss/esp fields of the
 * "struct pt_regs" is possible, but they may contain the
 * completely wrong values.
 */
#define task_pt_regs(task) \
({                  \
  unsigned long __ptr = (unsigned long)task_stack_page(task);  \
  __ptr += THREAD_SIZE - TOP_OF_KERNEL_STACK_PADDING;    \
  ((struct pt_regs *)__ptr) - 1;          \
})
```

先从 task_struct 找到内核栈的开始位置。然后这个位置加上 THREAD_SIZE 就到了最后的位置，然后转换为 struct pt_regs，再减一，就相当于减少了一个 pt_regs 的位置，就到了这个结构的首地址

```C
#ifdef CONFIG_X86_32
# ifdef CONFIG_VM86
#  define TOP_OF_KERNEL_STACK_PADDING 16
# else
#  define TOP_OF_KERNEL_STACK_PADDING 8
# endif
#else
# define TOP_OF_KERNEL_STACK_PADDING 0
#endif
```

##### 单核CPU

thread_info中有成员变量 task 指向 task_struct， 常用 current_thread_info()->task 来获取 task_struct。

```C
struct thread_info {
  struct task_struct  *task;    /* main task structure */
  __u32      flags;    /* low level flags */
  __u32      status;    /* thread synchronous flags */
  __u32      cpu;    /* current CPU */
  mm_segment_t    addr_limit;
  unsigned int    sig_on_uaccess_error:1;
  unsigned int    uaccess_err:1;  /* uaccess failed */
};

static inline struct thread_info *current_thread_info(void)
{
  return (struct thread_info *)(current_top_of_stack() - THREAD_SIZE);
}
```

##### 多核CPU

每个 CPU 运行的 task_struct 直接放在 Per CPU 变量里面了。多核情况下，CPU 是同时运行的，但是它们共同使用其他的硬件资源的时候。Per CPU 变量是内核中一种重要的同步机制，Per CPU 变量就是为每个 CPU 构造一个变量的副本，这样多个 CPU 各自操作自己的副本，互不干涉。

```C
#include <asm/current.h>
#define current_thread_info() ((struct thread_info *)current)
#endif

struct task_struct;

DECLARE_PER_CPU(struct task_struct *, current_task);
static __always_inline struct task_struct *get_current(void)
{
  return this_cpu_read_stable(current_task);
}
#define current get_current
```



### 3.4 调度

#### 进程优先级

在 Linux 里面，进程大概可以分成两种

+ 实时进程，也就是需要尽快执行返回结果的那种, 优先级的范围是 0～99
+ 普通进程，大部分的进程其实都是这种, 优先级的范围是 100～139

```C
int prio, static_prio, normal_prio;
unsigned int rt_priority;
```

#### 调度策略

##### 实时调度

+ **SCHED_FIFO**：高优先级的进程可以抢占低优先级的进程，而相同优先级的进程，我们遵循先来先得。
+ **SCHED_RR**：采用时间片，相同优先级的任务当用完时间片会被放到队列尾部，以保证公平性，而高优先级的任务也是可以抢占低优先级的任务。
+ **SCHED_DEADLINE**: 按照任务的 deadline 进行调度的。DL 调度器总是选择其 deadline 距离当前时间点最近的那个任务，并调度它执行。

##### 普通调度

+ **SCHED_NORMAL** 是普通的进程
+ **SCHED_BATCH** 是后台进程，几乎不需要和前端进行交互
+ **SCHED_IDLE** 是特别空闲的时候才跑的进程

##### Completely Fair Scheduling - 普通调度的一种实现

首先，你需要记录下进程的运行时间。CPU 会提供一个时钟，过一段时间就触发一个时钟中断，这个我们叫 Tick。**CFS 会为每一个进程安排一个虚拟运行时间 vruntime**。如果一个进程在运行，随着时间的增长，也就是一个个 tick 的到来，进程的 vruntime 将不断增大。没有得到执行的进程 vruntime 不变。

```c

/*
 * Update the current task's runtime statistics.
 */
static void update_curr(struct cfs_rq *cfs_rq)
{
  struct sched_entity *curr = cfs_rq->curr;
  u64 now = rq_clock_task(rq_of(cfs_rq));
  u64 delta_exec;
......
  delta_exec = now - curr->exec_start;
......
  curr->exec_start = now;
......
  curr->sum_exec_runtime += delta_exec;
......
  curr->vruntime += calc_delta_fair(delta_exec, curr);
  update_min_vruntime(cfs_rq);
......
}


/*
 * delta /= w
 */
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
  if (unlikely(se->load.weight != NICE_0_LOAD))
        /* delta_exec * weight / lw.weight */
    delta = __calc_delta(delta, NICE_0_LOAD, &se->load);
  return delta;
}
```

+ 得到当前的时间，以及这次的时间片开始的时间，两者相减就是这次运行的时间 delta_exec。
+ 做一定的转化才作为虚拟运行时间 vruntime。转化方法如下：虚拟运行时间 **vruntime += 实际运行时间 delta_exec * NICE_0_LOAD/ 权重**

同样的实际运行时间，给高权重的算少了，低权重的算多了



#### 调度队列/调度实体

看来 CFS 需要一个数据结构来对 vruntime 进行排序。能够平衡查询和更新速度的是树，在这里使用的是红黑树。红黑树的的节点是应该包括 vruntime 的，称为调度实体。

```C
struct sched_entity {
  struct load_weight    load;
  struct rb_node      run_node;
  struct list_head    group_node;
  unsigned int      on_rq;
  u64        exec_start;
  u64        sum_exec_runtime;
  u64        vruntime;
  u64        prev_sum_exec_runtime;
  u64        nr_migrations;
  struct sched_statistics    statistics;
......
};
```

![](./images/os-32.png)

CPU 也是这样的，每个 CPU 都有自己的 struct rq 结构，其包括

+ 一个实时进程队列 rt_rq 
+ 一个 CFS 运行队列 cfs_rq

在调度时，调度器首先会先去实时进程队列找是否有实时进程需要运行，如果没有才会去 CFS 运行队列找是否有进程需要运行。

```C
struct rq {
  /* runqueue lock: */
  raw_spinlock_t lock;
  unsigned int nr_running;
  unsigned long cpu_load[CPU_LOAD_IDX_MAX];
......
  struct load_weight load;
  unsigned long nr_load_updates;
  u64 nr_switches;

  struct cfs_rq cfs;
  struct rt_rq rt;
  struct dl_rq dl;
......
  struct task_struct *curr, *idle, *stop;
......
};
```

![](./images/os-33.png)

#### 调度类

```C
struct sched_class {
  const struct sched_class *next;


  void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
  void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
  void (*yield_task) (struct rq *rq);
  bool (*yield_to_task) (struct rq *rq, struct task_struct *p, bool preempt);


  void (*check_preempt_curr) (struct rq *rq, struct task_struct *p, int flags);


  struct task_struct * (*pick_next_task) (struct rq *rq,
            struct task_struct *prev,
            struct rq_flags *rf);
  void (*put_prev_task) (struct rq *rq, struct task_struct *p);


  void (*set_curr_task) (struct rq *rq);
  void (*task_tick) (struct rq *rq, struct task_struct *p, int queued);
  void (*task_fork) (struct task_struct *p);
  void (*task_dead) (struct task_struct *p);


  void (*switched_from) (struct rq *this_rq, struct task_struct *task);
  void (*switched_to) (struct rq *this_rq, struct task_struct *task);
  void (*prio_changed) (struct rq *this_rq, struct task_struct *task, int oldprio);
  unsigned int (*get_rr_interval) (struct rq *rq,
           struct task_struct *task);
  void (*update_curr) (struct rq *rq)
```

**所有的调度类是放在一个链表上的**。调度的时候是从优先级最高的调度类到优先级低的调度类，依次执行。而对于每种调度类，有自己的实现

![](./images/os-34.png)

#### 主动调度

##### 上下文切换

计算主要是 CPU 和内存的合作；网络和存储则多是和外部设备的合作；在操作外部设备的时候，往往需要让出 CPU，**选择调用 schedule() 函数。**

```c
asmlinkage __visible void __sched schedule(void)
{
  struct task_struct *tsk = current;


  sched_submit_work(tsk);
  do {
    preempt_disable();
    __schedule(false);
    sched_preempt_enable_no_resched();
  } while (need_resched());
}t
```

上下文切换主要干两件事情

+ switch_mm_irqs_off 一是切换进程空间，也即虚拟内存
+ switch_to 切换寄存器和 CPU 上下文

```C

/*
 * context_switch - switch to the new MM and the new thread's register state.
 */
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
         struct task_struct *next, struct rq_flags *rf)
{
  struct mm_struct *mm, *oldmm;
......
  mm = next->mm;
  oldmm = prev->active_mm;
......
  switch_mm_irqs_off(oldmm, mm, next);
......
  /* Here we just switch the register state and the stack. */
  switch_to(prev, next, prev);
  barrier();
  return finish_task_switch(prev);
}
```

switch_to。它就是寄存器和栈的切换，它调用到了 __switch_to_asm。这是一段汇编代码，主要用于栈的切换。

+ 对于 32 位操作系统来讲，切换的是栈顶指针 esp。
+ 对于 64 位操作系统来讲，切换的是栈顶指针 rsp。

```assembly
/*
 * %rdi: prev task
 * %rsi: next task
 */
ENTRY(__switch_to_asm)
......
  /* switch stack */
  movq  %rsp, TASK_threadsp(%rdi)
  movq  TASK_threadsp(%rsi), %rsp
......
  jmp  __switch_to
END(__switch_to_asm)
```

**在 x86 体系结构中，提供了一种以硬件的方式进行进程切换的模式，对于每个进程，x86 希望在内存里面维护一个 TSS（Task State Segment，任务状态段）结构。**有一个特殊的寄存器 TR（Task Register，任务寄存器），指向某个进程的 TSS。

更改 TR 的值，将会触发硬件保存 CPU 所有寄存器的值到当前进程的 TSS 中，然后从新进程的 TSS 中读出所有寄存器值，加载到 CPU 对应的寄存器中

````c

__visible __notrace_funcgraph struct task_struct *
__switch_to(struct task_struct *prev_p, struct task_struct *next_p)
{
  struct thread_struct *prev = &prev_p->thread;
  struct thread_struct *next = &next_p->thread;
......
  int cpu = smp_processor_id();
  struct tss_struct *tss = &per_cpu(cpu_tss, cpu);
......
  load_TLS(next, cpu);
......
  this_cpu_write(current_task, next_p);


  /* Reload esp0 and ss1.  This changes current_thread_info(). */
  load_sp0(tss, next);
......
  return prev_p;
}
````

**Linux 操作系统想了一个办法。还记得在系统初始化的时候，会调用 cpu_init。这里面会给每一个 CPU 关联一个 TSS，然后将 TR 指向这个 TSS**。在 Linux 中，真的参与进程切换的寄存器很少，主要的就是栈顶寄存器。**所谓的进程切换，就是将某个进程的 thread_struct 里面的寄存器的值，写入到 CPU 的 TR 指向的 tss_struct**

```C
void cpu_init(void)
{
  int cpu = smp_processor_id();
  struct task_struct *curr = current;
  struct tss_struct *t = &per_cpu(cpu_tss, cpu);
    ......
    load_sp0(t, thread);
  set_tss_desc(cpu, t);
  load_TR_desc();
    ......
}


struct tss_struct {
  /*
   * The hardware state:
   */
  struct x86_hw_tss  x86_tss;
  unsigned long    io_bitmap[IO_BITMAP_LONGS + 1];
} 
```

###### 用户态切换

+ 用户栈切换：内存切换
+ 用户栈顶指针切换：pt_regs 恢复
+ 用户指令指针切换：pt_regs 恢复

###### 内核态切换

+ 内核栈切换：**current_task指向新的task_struct**
+ 内核栈顶指针切换：sp/rp指向新的TSS
+ 内核指令指针切换：_switch_to_asm(prev, next) 

##### 切换过程

一个进程 A 在用户态是要写一个文件的，写文件的操作用户态没办法完成，就要通过系统调用到达内核态。在这个切换的过程中，用户态的指令指针寄存器是保存在 pt_regs 里面的，到了内核态，就开始沿着写文件的逻辑一步一步执行，结果发现需要等待，于是就调用 __schedule 函数

```c
switch_to(prev, next, prev);
barrier();
return finish_task_switch(prev);
```

当进程 A 在内核里面执行 switch_to 的时候，内核态的指令指针也是指向这一行的。**但是在 switch_to 里面，将寄存器和栈都切换到成了进程 B 的，唯一没有变的就是指令指针寄存器。**当 switch_to 返回的时候，指令指针寄存器指向了下一条语句 finish_task_switch。

当年 B 进程被别人切换走的时候，也是调用 **\__schedule，也是调用到 switch_to，被切换成为 C 进程的，所以，B 进程当年的下一个指令也是 finish_task_switch**，这就说明指令指针指到这里是没有错的。假设B 就是最前面例子里面调用 tap_do_read 读网卡的进程。它当年调用 **\__schedule 的时候，是从 tap_do_read 这个函数调用进去的。B 进程的内核栈里面放的是 tap_do_read。于是，从 __schedule 返回之后，当然是接着 tap_do_read 运行，**然后在内核运行完毕后，返回用户态。

```C
#define switch_to(prev, next, last)          \
do {                  \
  prepare_switch_to(prev, next);          \
                  \
  ((last) = __switch_to_asm((prev), (next)));      \
} while (0)
```

A 切换到 B 的时候，运行到 __switch_to_asm 这一行的时候，是在 A 的内核栈上运行的，prev 是 A，next 是 B。但是，A 执行完 __switch_to_asm 之后就被切换走了，当 C 再次切换到 A 的时候，运行到 __switch_to_asm，是从 C 的内核栈运行的。这个时候，prev 是 C，next 是 A，但是 __switch_to_asm 里面切换成为了 A 当时的内核栈。



#### 抢占调度

一个进程执行时间太长了，是时候切换到另一个进程了

##### 抢占准备

###### 时钟中断抢占

计算机里面有一个时钟，会过一段时间触发一次时钟中断，通知操作系统，时间又过去一个时钟周期，这是个很好的方式，可以查看是否是需要抢占的时间点。时钟中断处理函数会调用 scheduler_tick()

```c
void scheduler_tick(void)
{
  int cpu = smp_processor_id();
  struct rq *rq = cpu_rq(cpu);
  struct task_struct *curr = rq->curr;
......
  curr->sched_class->task_tick(rq, curr, 0);
  cpu_load_update_active(rq);
  calc_global_load_tick(rq);
......
}
```

scheduler_tick 调用task_tick， task_tick调用entity_tick

在 entity_tick 里面，update_curr会更新当前进程的 vruntime，然后调用 check_preempt_tick，检查是否是时候被抢占了。

```C
static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
  unsigned long ideal_runtime, delta_exec;
  struct sched_entity *se;
  s64 delta;


  ideal_runtime = sched_slice(cfs_rq, curr);
  delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
  if (delta_exec > ideal_runtime) {
    resched_curr(rq_of(cfs_rq));
    return;
  }
......
  se = __pick_first_entity(cfs_rq);
  delta = curr->vruntime - se->vruntime;
  if (delta < 0)
    return;
  if (delta > ideal_runtime)
    resched_curr(rq_of(cfs_rq));
}
```

如下情况会被抢占

+ sum_exec_runtime 指进程总共执行的实际时间，prev_sum_exec_runtime 指上次该进程被调度时已经占用的实际时间。**sum_exec_runtime-prev_sum_exec_runtime 就是这次调度占用实际时间。如果这个时间大于 ideal_runtime，**则应该被抢占了。
+ 通过 __pick_first_entity 取出红黑树中最小的进程。**如果当前进程的 vruntime 大于红黑树中最小的进程的 vruntime，且差值大于 ideal_runtime**

标记一个进程应该被抢占，都是调用 resched_curr，它会调用 set_tsk_need_resched，标记进程应该被抢占，并不真的抢占，而是打上一个标签 TIF_NEED_RESCHED。

###### 唤醒抢占

当一个进程在等待一个 I/O 的时候，会主动放弃 CPU。但是当 I/O 到来的时候，进程往往会被唤醒。这个时候是一个时机。当被唤醒的进程优先级高于 CPU 上的当前进程，就会触发抢占。

```C
static void ttwu_do_wakeup(struct rq *rq, struct task_struct *p, int wake_flags,
         struct rq_flags *rf)
{
  check_preempt_curr(rq, p, wake_flags);
  p->state = TASK_RUNNING;
  trace_sched_wakeup(p);
```

##### 抢占时机

###### 用户态的抢占时机

64 位的系统调用的链路位 do_syscall_64->syscall_return_slowpath->prepare_exit_to_usermode->exit_to_usermode_loop

```C
static void exit_to_usermode_loop(struct pt_regs *regs, u32 cached_flags)
{
  while (true) {
    /* We have work to do. */
    local_irq_enable();


    if (cached_flags & _TIF_NEED_RESCHED)
      schedule();
......
  }
}
```

exit_to_usermode_loop 函数中，上面打的标记起了作用，如果被打了 _TIF_NEED_RESCHED，调用 schedule 进行调度。**对于用户态的进程来讲，从中断中返回的那个时刻，也是一个被抢占的时机。**

###### 内核态的抢占时机

在内核态的执行中，有的操作是不能被中断的，所以在进行这些操作之前，总是先调用 preempt_disable() 关闭抢占，当再次打开的时候，就是一次内核态代码被抢占的机会。

**preempt_enable()** 会调用 preempt_count_dec_and_test()，**判断 preempt_count 和 TIF_NEED_RESCHED 是否可以被抢占**。如果可以，就调用 preempt_schedule->preempt_schedule_common->__schedule 进行调度。

```C

#define preempt_enable() \
do { \
  if (unlikely(preempt_count_dec_and_test())) \
    __preempt_schedule(); \
} while (0)


#define preempt_count_dec_and_test() \
  ({ preempt_count_sub(1); should_resched(0); })

static __always_inline bool should_resched(int preempt_offset)
{
  return unlikely(preempt_count() == preempt_offset &&
      tif_need_resched());
}

#define tif_need_resched() test_thread_flag(TIF_NEED_RESCHED)

static void __sched notrace preempt_schedule_common(void)
{
  do {
......
    __schedule(true);
......
  } while (need_resched())
```

![](./images/os-35.png)







