---
title: c语言 - 多进程编程
date: 2017-08-11 11:08:19
categories:
- c
tags:
- c
keywords: c语言 多进程编程 fork()
---

> 
c语言 - 多进程编程

<!-- more -->

## 进程、线程
**进程是资源分配的最小单位，线程是CPU调度的最小单位**

**进程(Process)**
进程(Process)是计算机中的程序关于某数据集合上的一次运行活动，是系统进行资源分配和调度的基本单位，是操作系统结构的基础；
在早期面向进程设计的计算机结构中，进程是程序的基本执行实体；在当代面向线程设计的计算机结构中，进程是线程的容器；
程序是指令、数据及其组织形式的描述，进程是程序的实体；

**线程(Thread)**
线程(Thread)，有时被称为轻量级进程(Lightweight Process，LWP)，是程序执行流的最小单元；一个标准的线程由线程ID，当前指令指针(PC)，寄存器集合和堆栈组成；
另外，线程是进程中的一个实体，是被系统独立调度和分派的基本单位，线程自己不拥有系统资源，只拥有一点儿在运行中必不可少的资源，但它可与同属一个进程的其它线程共享进程所拥有的全部资源；
一个线程可以创建和撤消另一个线程，同一进程中的多个线程之间可以并发执行；
由于线程之间的相互制约，致使线程在运行中呈现出间断性；

线程也有就绪、阻塞和运行三种基本状态：
就绪状态是指线程具备运行的所有条件，逻辑上可以运行，在等待处理机；
运行状态是指线程占有处理机正在运行；
阻塞状态是指线程在等待一个事件(如某个信号量)，逻辑上不可执行；

每一个程序都至少有一个线程，若程序只有一个线程，那就是程序本身；
线程是程序中一个单一的顺序控制流程，进程内一个相对独立的、可调度的执行单元，是系统独立调度和分派CPU的基本单位指运行中的程序的调度单位；
在单个程序中同时运行多个线程完成不同的工作，称为多线程；

**进程和线程的区别**
- 地址空间和其它资源(如打开的文件)：进程间相互独立，同一进程的各线程间共享；某进程内的线程在其它进程不可见；
- 通信：进程间通信IPC，线程间可以直接读写进程数据段(如全局变量)来进行通信(需要进程同步和互斥手段的辅助，以保证数据的一致性)；
- 调度和切换：线程上下文切换比进程上下文切换要快得多；
- 在多线程OS中，进程不是一个可执行的实体；

## linux的进程
ps命令来查看当前系统中运行的进程：
<pre><code class="language-c line-numbers"><script type="text/plain"># root @ localhost in ~ [11:30:48]
$ ps -eo pid,ppid,comm,cmd | head
   PID   PPID COMMAND         CMD
     1      0 systemd         /usr/lib/systemd/systemd --switched-root --system --deserialize 21
     2      0 kthreadd        [kthreadd]
     3      2 ksoftirqd/0     [ksoftirqd/0]
     5      2 kworker/0:0H    [kworker/0:0H]
     7      2 migration/0     [migration/0]
     8      2 rcu_bh          [rcu_bh]
     9      2 rcu_sched       [rcu_sched]
    10      2 watchdog/0      [watchdog/0]
    11      2 watchdog/1      [watchdog/1]

# root @ localhost in ~ [11:31:20]
$ pstree
systemd─┬─NetworkManager───3*[{NetworkManager}]
        ├─agetty
        ├─auditd───{auditd}
        ├─chronyd
        ├─crond
        ├─dbus-daemon
        ├─irqbalance
        ├─lvmetad
        ├─polkitd───5*[{polkitd}]
        ├─rsyslogd───2*[{rsyslogd}]
        ├─sshd─┬─sshd───zsh───pstree
        │      └─sshd───zsh
        ├─systemd-journal
        ├─systemd-logind
        ├─systemd-udevd
        └─tuned───4*[{tuned}]
</script></code></pre>

ps命令中：
pid表示进程的进程ID，ppid表示进程的父进程ID，comm是进程的简称，cmd是进程对应的程序以及运行时的参数

这里重点说一下`pid = 1`的进程`systemd`：
> 
我使用的Linux发行版是CentOS 7，而CentOS 7最大的变化就是用`systemd`取代了`sysvinit`；
在`sysvinit`中：`init`进程是所有进程的父进程；而在`systemd`中，`systemd`进程是所有进程的父进程；
为了叙述方便，我们还是用`init`这个名，指代pid为1的进程，在sysvinit中是init，在systemd中是systemd；

当Linux启动的时候，init是系统创建的第一个进程，这一进程会一直存在，直到我们关闭计算机；

实际上，在开机的时候，内核(kernel)只建立了一个init进程；
Linux内核并不提供直接建立新进程的系统调用，剩下的所有进程都是init进程通过fork机制建立的；
新的进程要通过老的进程复制自身得到，这就是fork，fork是一个系统调用；
进程存活于内存中，每个进程都在内存中分配有属于自己的一片空间(address space)；
当进程调用fork的时候，Linux在内存中开辟出一片新的内存空间给新的进程，并将老的进程空间中的内容复制到新的空间中，此后两个进程同时运行；

老进程成为新进程的父进程(parent process)，而相应的，新进程就是老进程的子进程(child process)；
一个进程除了有一个PID之外，还会有一个PPID(parent PID)来存储的父进程PID；
如果我们循着PPID不断向上追溯的话，总会发现其源头是init进程，所以说，所有的进程也构成一个`以init为根的树状结构`

pstree命令可以显示一个以init为根的进程树；

fork通常作为一个函数被调用，这个函数会有两次返回，将子进程的PID返回给父进程，0返回给子进程；
实际上，子进程总可以查询自己的PPID来知道自己的父进程是谁，这样，一对父进程和子进程就可以随时查询对方；

通常在调用fork函数之后，程序会设计一个if分支结构：
当PID等于0时，说明该进程为子进程，那么让它执行某些指令，比如说使用exec库函数(library function)来执行另一个程序；
而当PID为一个正整数时，说明为父进程，则执行另外一些指令；由此，就可以在子进程建立之后，让它执行与父进程不同的功能；

**孤儿进程和僵尸进程**
当子进程终结时，它会通知父进程，并清空自己所占据的内存，并在内核里留下自己的退出信息(exit code，如果顺利运行，为0；如果有错误或异常状况，为>0的整数)；在这个信息里，会解释该进程为什么退出；
父进程在得知子进程终结时，有责任对该子进程使用wait系统调用，这个wait函数能从内核中取出子进程的退出信息，并清空该信息在内核中所占据的空间；

但是，如果父进程早于子进程终结，子进程就会成为一个`孤儿(orphand)进程`；
孤儿进程会被过继给init进程，init进程也就成了该进程的父进程；init进程负责该子进程终结时调用wait函数；

当然，一个糟糕的程序也完全可能造成子进程的退出信息滞留在内核中的状况(父进程不对子进程调用wait函数)；
这样的情况下，子进程成为`僵尸(zombie)进程`；当大量僵尸进程积累时，内存空间会被挤占；

**僵尸进程的处理**
查看僵尸进程：`ps -ef | grep 'defunct' | grep -v 'grep'`
僵尸进程的状态为`Z`，一般我们很难直接用`kill -9`杀死僵尸进程，不过我们可以`kill -9`它们的父进程；
不过，如果父进程都已经不在了，那么只能重启系统了。。。

## 进程的基本操作
**fork**
`pid_t fork(void);`：进程的创建
- 头文件：`unistd.h`
- 返回值：执行成功：在父进程中返回子进程的pid，在子进程中返回0；执行失败：返回-1，并设置errno

头文件：`unistd.h`
`pid_t getpid(void);`：获取当前进程的进程ID
`pid_t getppid(void);`：获取当前进程的父进程ID
`uid_t getuid(void);`：获取进程的实际用户ID
`uid_t geteuid(void);`：获取进程的有效用户ID
`gid_t getgid(void);`：获取进程的实际用户组ID
`gid_t getegid(void);`：获取进程的有效用户组ID

新进程的创建，首先在内存中为新进程创建一个`task_struct`结构，然后将父进程的`task_struct`内容复制其中，再修改部分数据：分配新的内核堆栈、新的PID、再将这个task_struct添加到链表中，所谓创建，实际上是“复制”；

子进程刚开始，内核并没有为它分配物理内存，而是以只读的方式共享父进程内存，只有当子进程写时，才复制；即`copy-on-write`；fork都是由do_fork实现的；

一般来说，fork之后父、子进程执行顺序是不确定的，这取决于内核调度算法；进程之间实现同步需要进行进程通信；

一个简单的fork演示程序：
<pre><code class="language-c line-numbers"><script type="text/plain">#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(void){
    int n = 0;
    printf("before fork: n = %d\n", n);

    pid_t fpid = fork();
    if(fpid < 0){
        perror("fork error");
        exit(EXIT_FAILURE);
    }else if(fpid == 0){
        n++;
        printf("child_proc(%d, ppid=%d): n = %d\n", getpid(), getppid(), n);
    }else{
        n--;
        printf("parent_proc(%d): n = %d\n", getpid(), n);
    }

    printf("quit_proc(%d) ... \n", getpid());

    return 0;
}
</script></code></pre>

<pre><code class="language-c line-numbers"><script type="text/plain"># root @ localhost in ~/tmp [13:17:34]
$ gcc a.c

# root @ localhost in ~/tmp [13:17:36]
$ ./a.out
before fork: n = 0
parent_proc(5159): n = -1
quit_proc(5159) ...
child_proc(5160, ppid=5159): n = 1
quit_proc(5160) ...
</script></code></pre>


**fork和vfork**
fork创建子进程，把父进程数据空间、堆和栈复制一份；
vfork创建子进程，与父进程内存数据共享；
vfork先保证子进程先执行，当子进程调用exit()或者exec后，父进程才往下执行；

为什么需要vfork？
因为用vfork时，一般都是紧接着调用exec，所以不会访问父进程数据空间，也就不需要在数据复制上花费时间了，因此vfork就是”为了exec而生“的；

但是后来的fork也学聪明了，不是一开始调用fork就复制数据，而是只有在子进程要修改数据的时候，才进行复制，即`copy-on-write`；
所以我们现在也很少去用vfork，因为vfork的优势已经不复存在了；

**wait、waitpid**
`pid_t wait(int *status);`：等待任意子进程退出，并捕获退出状态
- 头文件：`sys/wait.h`、`sys/types.h`
- `status`：输出参数，保存退出状态，可设置为NULL
- 返回值：返回捕获到的子进程ID，失败返回-1，并设置errno

`pid_t waitpid(pid_t pid, int *status, int options);`：等待子进程退出，并捕获退出状态
- 头文件：`sys/wait.h`、`sys/types.h`
- `pid`：输入参数，等待什么子进程：
`>0`：等待指定pid的子进程；
`0`：等待与当前进程同一组ID下的任意子进程；
`-1`：等待任意子进程；同wait()；
`<-1`：等待组ID为pid绝对值的任意子进程；
- `status`：输出参数，保存退出状态，可设置为NULL
- `options`：输入参数，等待子进程的选项：
`WNOHANG`：如果没有子进程终止就立即返回，返回值为0；
`WUNTRACED`：如果一个子进程stoped且没有被traced，那么立即返回；
`WCONTINUED`：如果stoped的子进程通过SIGCONT复苏，那么立即返回；
- 返回值：成功返回捕获到的子进程ID，WNOHANG选项下若没捕获到子进程退出则返回0，失败返回-1，并设置errno

`status`退出状态的相关宏：
- `WIFEXITED(status)`：如果子进程正常退出返回真
- `WEXITSTATUS(status)`：返回子进程的退出码，当且仅当WIFEXITED为真时有效
- `WIFSIGNALED(status)`：如果子进程被一个信号终止时返回真
- `WTERMSIG(status)`：返回终止子进程的信号编号，当且仅当WIFSIGNALED为真时有效
- `WCOREDUMP(status)`：如果子进程导致了"核心已转储"则返回真，当且仅当WIFSIGNALED为真时有效
- `WIFSTOPPED(status)`：如果子进程被一个信号暂停时返回真，当且仅当调用进程使用WUNTRACED或子进程正在traced时有效
- `WSTOPSIG(status)`：返回引起子进程暂停的信号编号，当且仅当WIFSTOPPED为真时有效
- `WIFCONTINUED(status)`：如果子进程收到SIGCONT而复苏时返回真

处理子进程的退出有以下两种方式：
第一种：通过信号处理函数`signal()`，如可以忽略子进程的`SIGCHLD`信号来防止僵尸进程的产生：`signal(SIGCHLD, SIG_IGN);`
第二种：通过调用`wait()`、`waitpid()`函数，来回收子进程

**wait实例**
<pre><code class="language-c line-numbers"><script type="text/plain">#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <wait.h>

int main(void){
    pid_t fpid = fork(), pid;

    if(fpid < 0){
        perror("fork error");
        exit(EXIT_FAILURE);
    }else if(fpid == 0){
        sleep(5);
        exit(5);
    }else{
        int stat;

        for(;;){
            pid = waitpid(fpid, &stat, WNOHANG);
            if(pid > 0){
                break;
            }else{
                printf("wait_child_proc ... \n");
                sleep(1);
            }
        }

        if(WIFEXITED(stat)){
            printf("child_proc(%d): exit_code: %d\n", pid, WEXITSTATUS(stat));
        }
    }
    return 0;
}
</script></code></pre>

<pre><code class="language-c line-numbers"><script type="text/plain"># root @ localhost in ~/tmp [14:36:03]
$ gcc a.c

# root @ localhost in ~/tmp [14:36:04]
$ ./a.out
wait_child_proc ...
wait_child_proc ...
wait_child_proc ...
wait_child_proc ...
wait_child_proc ...
child_proc(5716): exit_code: 5
</script></code></pre>


**exec系列函数**
exec系统调用是以新的进程空间替换现在的进程空间，但是pid不变，还是原来的pid，相当于换了个身体，但是名字不变；
调用exec后，系统会申请一块新的进程空间来存放被调用的程序，然后当前进程会携带pid跳转到新的进程空间，并从main函数开始执行，旧的进程空间被回收；

exec是一个函数族，由6个函数组成：
头文件：`unistd.h`
`int execl(const char *path, const char *arg, ...);`
`int execlp(const char *file, const char *arg, ...);`
`int execle(const char *path, const char *arg, ..., char *const envp[]);`
`int execv(const char *path, char *const argv[]);`
`int execvp(const char *file, char *const argv[]);`
`int execve(const char *path, char *const argv[], char *const envp[]);`

我们先来看一下我们熟悉的main函数：
`int main(int argc, char *argv[], char *envp[]);`
- `argc`：命令行参数的个数，一个整数；
- `argv`：命令行参数，一个字符串数组，数组最后一个元素为NULL；
- `envp`：环境变量，一个字符串数组，数组最后一个元素为NULL；

我们主要看`execve()`函数，其他的函数都是此函数的不同包装而已：
`int execve(const char *path, char *const argv[], char *const envp[]);`

有没有发现和main函数是如此的相似，第一个参数是目标程序的绝对路径，第二个参数是命令行参数，第三个参数是环境变量；
命令行参数和环境变量都是一个以NULL结尾的字符串数组，和main函数一样；

主要分为两大类：
- 以`execv`开头：argv参数是一个字符串数组的形式；
- 以`execl`开头：argv参数是一个可变参数列表的形式；

细分又有三类：
- 不带后缀`e`：表示新进程继承旧进程的环境变量；
- 带后缀`e`：表示新进程不继承旧进程的环境变量，环境变量由envp参数指定；
- 带后缀`p`：表示第一个参数可以是一个程序名，该程序名可以在环境变量PATH中找到；

命令行参数argv的第0个元素是执行的文件名；
环境变量envp的每个元素都是`KEY=VALUE`的形式，`KEY`是变量名，`VALUE`是变量的值；

返回值：执行成功没有返回值，因为原来的进程空间都没了，执行失败则返回-1，并设置errno

一个简单的例子：`execlp("echo", "echo", "www.zfl9.com", NULL);`将执行命令`echo www.zfl9.com`

**exit系列函数**
正常终止：
- 从`main()`返回
- 调用`exit()`、`_exit()`、`_Exit()`
- 最后一个线程从其启动例程返回
- 最后一个线程调用`pthread_exit()`

异常终止：
- 调用`abort()`
- 接到一个`信号`并终止
- 最后一个线程对取消请求作出响应

退出状态`exit status`是我们传入到`exit()`、`_exit()`、`_Exit()`函数的参数；
进程正常终止的情况下，内核将退出状态转变为终止状态以供父进程使用`wait()`、`waitpid()`等函数获取；
终止状态`termination status`除了上述正常终止进程的情况外，还包括异常终止的情况；
如果进程异常终止，那么内核也会用一个指示其异常终止原因的终止状态来表示进程，当然，这种终止状态也可以由父进程的`wait()`、`waitpid()`进程捕获；

头文件：`stdlib.h`
`void exit(int status);`
`void _Exit(int status);`
头文件：`unistd.h`
`void _exit(int status);`

**`exit`和`_exit/_Exit`的区别**
exit是系统调用级别的，用于进程运行的过程中，随时结束进程；
return是语言级别的，用于调用堆栈的返回，返回上一层调用；
exit会调用终止处理程序和用户空间的标准I/O清理程序(如fclose)，而_exit和_Exit不调用而直接由内核接管进行清理；
在main函数中调用`exit(0)`等价于`return 0`；

`_exit()`函数的作用最为简单：直接使进程停止运行，清除其使用的内存空间，并销毁其在内核中的各种数据结构；
`exit()`函数则在这些基础上作了一些包装，在执行退出之前加了若干道工序；
`exit()`函数与`_exit()`函数最大的区别就在于`exit()`要检查文件的打开情况，把文件缓冲区中的内容写回文件，就是"清理I/O缓冲"；

按照ANSI C的规定，一个进程可以登记至多32个函数，这些函数将由exit自动调用；
我们称这些函数为`终止处理程序(exit handler)`，并用`atexit`函数来登记这些函数；

**atexit**
头文件：`stdlib.h`
`int atexit(void (*func)(void));`：登记终止处理函数
- `void (*func)(void)`：函数指针，该函数不接受任何参数，也不返回任何值；
- 返回值：成功返回0，失败返回-1，并设置errno

exit以登记这些函数的相反顺序调用它们，同一函数如若登记多次，则也被调用多次；

根据ANSI C和POSIX.1，exit首先调用各终止处理程序，然后按需多次调用fclose，关闭所有打开的流；

**on_exit**
头文件：`stdlib.h`
`int on_exit(void (*func)(int, void *), void *arg);`：登记带参数的终止处理函数
- `void (*func)(int, void *)`：函数指针，该函数接收两个参数，第一个参数为exit的参数，即退出状态码；第二个参数为一个指针，通过后面的arg参数传递；
- `arg`：传递给函数指针的第二个参数；
- 返回值：成功返回0，失败返回-1，并设置errno

exit以登记这些函数的相反顺序调用它们，同一函数如若登记多次，则也被调用多次；

**fork相关事项**
fork出的子进程会继承父进程的终止处理函数、信号处理设置；
但是在调用exec之后，终止处理函数和信号处理设置都会被清空；

**exit例子**
<pre><code class="language-c line-numbers"><script type="text/plain">#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <signal.h>

void func1(void){
    printf("<atexit> func1\n");
}

void func2(void){
    printf("<atexit> func2\n");
}

void func3(void){
    printf("<atexit> func3\n");
}

void func(int status, void *str){
    printf("<on_exit> exit_code: %d, arg: %s\n", status, (char *)str);
}

int main(void){
    signal(SIGCHLD, SIG_IGN);

    on_exit(func, "on_exit3");
    on_exit(func, "on_exit2");
    on_exit(func, "on_exit1");

    atexit(func3);
    atexit(func2);
    atexit(func1);

    pid_t pid;
    pid = fork();
    if(pid < 0){
        perror("fork error");
        exit(EXIT_FAILURE);
    }else if(pid == 0){
        exit(0);
    }else{
        sleep(3);
    }

    return 0;
}
</script></code></pre>

<pre><code class="language-c line-numbers"><script type="text/plain"># root @ localhost in ~/tmp [20:36:23]
$ gcc a.c

# root @ localhost in ~/tmp [20:36:44]
$ ./a.out
<atexit> func1
<atexit> func2
<atexit> func3
<on_exit> exit_code: 0, arg: on_exit1
<on_exit> exit_code: 0, arg: on_exit2
<on_exit> exit_code: 0, arg: on_exit3
<atexit> func1
<atexit> func2
<atexit> func3
<on_exit> exit_code: 0, arg: on_exit1
<on_exit> exit_code: 0, arg: on_exit2
<on_exit> exit_code: 0, arg: on_exit3
</script></code></pre>