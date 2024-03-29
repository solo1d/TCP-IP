**进程间通信(Interprocess Communication,  IPC),  IPC的基本设计目标是高性能**

**Posix是 可移植操作系统接口 `(Portable Operating System Interface)` 首字母的缩写**

- [UNIX进程间共享信息的三种方式](#UNIX进程间共享信息的三种方式)
- [IPC对象的持续性](#IPC对象的持续性)
- [名字空间](#名字空间)
- [fork与exec和exit对IPC对象的影响](#fork与exec和exit对IPC对象的影响)
- [出错处理的包裹函数](#出错处理的包裹函数)
- [UNIXerrno值](#UNIXerrno值)
- 





> **shell管道 属于IPC机制, 线程间通信也属于IPC通信**
>
> - **四种IPC 形式:**
>   - 消息传递 ,  管道, FIFO和消息队列
>   - 同步  ,  排斥量,条件变量, 读写锁, 文件和记录锁,  信号量
>   - 共享内存  ,  匿名的 和 具名的
>   - 远程过程调用  , Solaris门 和 Sun RPC
> - **sock通信也算是一种 IPC,  但速度比 单主机IPC 要慢得多**
> - **共享内存, 同步 等方法通常也只能用于当主机, 跨网络时可能无法使用.**
> - **同步用的 Posix 函数**: 互斥锁, 条件变量  以及读写锁.  它们可用于线程或进程的同步,而且往往在访问共享内存时使用.
> - pthreads 就是 Posix线程环境
> - **对空管道进行 read读取, 会阻塞当前线程  或进程**
> - **FIFO指先进先出（first in，first out），它是一个单向（半双工）数据流。不同于管道的是，每个FIFO有一个路径名与之关联，从而允许无亲缘关系的进程访问同一个FIFO。FIFO也称为有名管道（named pipe）。**



## UNIX进程间共享信息的三种方式

1. **两个进程共享存留于文件系统中的某个文件上的某些信息.  为访问这些信息,每个进程都得穿越内核(read等).**
2. **两个进程共享驻留于内核中的某些信息.  管道是这种共享类型的一个例子, System V 消息队列和 System V信号量也是,  每次访问共享信息都涉及对内核的一次系统调用**
3. **两个进程有一个双方都能访问的共享内存区.  每个进程一旦设置好该共享内存区, 就能根本不涉及内核而访问其中的数据.  共享内存区的进程需要某种形式的同步.**





## IPC对象的持续性

- **可以将 任意类型的IPC持续性 (persistence) 定义成该烈性的一个对象 一直存在多长时间.**
  - **随进程持续的`(process-presistent)`  IPC对象 一直存在到打开着该对象的最后一个进程关闭该对象为止.  **
    - **例如 FIFO和管道**
        - **FIFO指先进先出（first in，first out），它是一个单向（半双工）数据流。不同于管道的是，每个FIFO有一个路径名与之关联，从而允许无亲缘关系的进程访问同一个FIFO。FIFO也称为有名管道（named pipe）。**
  - **随内核持续的 `(kernel-persistent)` IPC对象 移植存在到内核重新自举或显示删除该对象为止.**
    - 例如 System V 消息队列,  信号量, 共享内存区.
    - Posix消息队列, 信号量,和共享内存区必须至少是随内核持续的, 但也可以是随文件系统持续的,具体取决于实现
  - **随文件系统持续的 `(filesystem-persistent)` IPC对象一直存在到显示删除该对象为止.即使内核重新自举可, 该对象还是保持其值.**
    - Posix消息队列, 信号量, 共享内存区 如果是使用映射文件实现的 (不是必须条件) , 那么它们就是随文件系统持续的

|        IPC类型        | 持续性 |
| :-------------------: | :----: |
|         管道          | 随进程 |
|         FIFO          | 随进程 |
|      Posix互斥锁      | 随进程 |
|     Posix条件变量     | 随进程 |
|      Posix读写锁      | 随进程 |
|    fcntl 记录上锁     | 随进程 |
|     Posix消息队列     | 随内核 |
|    Posix有名信号量    | 随内核 |
| Posix基于内存的信号量 | 随内核 |
|    Posix共享内存区    | 随内核 |
|   System V消息队列    | 随内核 |
|    System V信号量     | 随内核 |
|  System V共享内存区   | 随内核 |
|       TCP套接字       | 随进程 |
|       UDP套接字       | 随进程 |
|     Unix域套接字      | 随进程 |



## 名字空间

**当两个或多个 无亲缘关系 的进程使用某种类型的 IPC对象来彼此交换信息时,  该IPC 对象必须有一个某种形式的 ==名字或标识符== ,   这样其中一个进程(往往是服务器)  可以创建该 IPC 对象, 其余进程则可以指定同一个 IPC对象**

- **对于一种给定的 IPC 类型, 其可能的名字的集合称为它的 ==名字空间==**
  - **名字空间特别重要, 对于除普通管道之外所有形式的IPC来说, 名字是客户端与服务器彼此连接以交换消息的手段**
- 管道没有名字, 因此不能用于无亲缘关系的进程间



**无名同步变量:  互斥锁, 条件变量, 读写锁, 基于内存的信号量**



**下面的 定义和标准都写在  <unistd.h> 这个头文件中**

|                   IPC类型                   | 用于打开或创建IPC的名字空间 |  IPC打开后的标识符   |                    Posix.1 1996                    |     Unix 98     |
| :-----------------------------------------: | :-------------------------: | :------------------: | :------------------------------------------------: | :-------------: |
|                    管道                     |         (没有名字)          |        描述符        |                        强制                        |      强制       |
|                    FIFO                     |           路径名            |        描述符        |                        强制                        |      强制       |
|                 Posix互斥锁                 |         (没有名字)          | pthread_mutex_t指针  |                   _POSIX_THREADS                   |      强制       |
|                Posix条件变量                |         (没有名字)          |  pthread_cond_t指针  |                   _POSIX_THREADS                   |      强制       |
| Posix条件变量  进程间共享的 互斥锁/条件变量 |                             |                      |           _POSIX_THREADS_PROCESS_SHARED            |      强制       |
|                 Posix读写锁                 |         (没有名字)          | pthread_rwlock_t指针 |                      (未定义)                      |      强制       |
|               fcntl 记录上锁                |           路径名            |        描述符        |                        强制                        |      强制       |
|                Posix消息队列                |        Posix IPC名字        |       mqd_t值        |               _POSIX_MESSAGE_PASSING               | _XOPEN_REALTIME |
|               Posix有名信号量               |        Posix IPC名字        |      sem_t指针       |                 _POSIX_SEMAPHORES                  | _XOPEN_REALTIME |
|            Posix基于内存的信号量            |         (没有名字)          |      sem_t指针       |                                                    |                 |
|               Posix共享内存区               |        Posix IPC名字        |        描述符        |            _POSXI_SHARED_MEMORY_OBJECTS            | _XOPEN_REALTIME |
|              System V消息队列               |           key_t键           |  System V IPC描述符  |                      (未定义)                      |      强制       |
|               System V信号量                |           key_t键           |  System V IPC描述符  |                      (未定义)                      |      强制       |
|             System V共享内存区              |           key_t键           |  System V IPC描述符  |                      (未定义)                      |      强制       |
|                     门                      |           路径名            |        描述符        |                      (未定义)                      |    (未定义)     |
|                   Sun RPC                   |          程序/版本          |       RPC句柄        |                      (未定义)                      |    (未定义)     |
|                  TCP套接字                  |        Posix IPC名字        |        描述符        |                                                    |                 |
|                  UDP套接字                  |        Posix IPC名字        |        描述符        |                                                    |                 |
|                Unix域套接字                 |           路径名            |        描述符        |                                                    |                 |
|                    mmap                     |                             |                      | _POSIX_MAPPED_FILE 或 _POSIX_SHARED_MEMORY_OBJECTS |      强制       |
|                  实时信号                   |                             |                      |              _POSIX_REALTIME_SIGNALS               | _XOPEN_REALTIME |





## fork与exec和exit对IPC对象的影响

|        IPC类型         |                             fork                             |                             exec                             |                            _exit                             |
| :--------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|       管道与FIFO       |          子进程取得父进程的所有打开着的描述符的副本          | 所有打开着的描述符继续打开着, 除非已设置描述符的 FD_CLOEXEC位 | 关闭所有打开着的描述符, 最后一个关闭时删除管道或 FIFO中残留的所有数据 |
|     Posix消息队列      |      子进程取得父进程的所有打开着的消息队列描述符的副本      |                关闭所有打开着的消息队列描述符                |                关闭所有打开着的消息队列描述符                |
|    System V消息队列    |                           没有效果                           |                           没有效果                           |                           没有效果                           |
| Posix 互斥锁和条件变量 |      若驻留在共享内存区中而且具有进程间共享属性, 这共享      | 除非在继续打开着的共享内存区中而且具有进程间共享属性, 否则消失 | 除非在继续打开着的共享内存区中而且具有进程间共享属性, 否则消失 |
|      Posix读写锁       |      若驻留在共享内存区中而且具有进程间共享属性, 这共享      | 除非在继续打开着的共享内存区中而且具有进程间共享属性, 否则消失 | 除非在继续打开着的共享内存区中而且具有进程间共享属性, 否则消息 |
| Posix基于内存的信号量  |      若驻留在共享内存区中而且具有进程间共享属性, 这共享      | 除非在继续打开着的共享内存区中而且具有进程间共享属性, 否则消失 | 除非在继续打开着的共享内存区中而且具有进程间共享属性, 否则消失 |
|    Posix有名信号量     |      父进程中所有打开着的有名信号量在子进程中继续打开着      |                  关闭所有打开着的有名信号量                  |                  关闭所有打开着的有名信号量                  |
|     System V信号量     |                子进程中所有semadj值都设置为0                 |                  所有semadj值都携入新程序中                  |              所有semadj值都加到相信的信号量值上              |
|     fcntl记录上锁      |                 子进程不继承由父进程持有的锁                 |                只要描述符继续打开着, 锁就不变                |                解开由进程持有的所有未处理的锁                |
|      mmap内存映射      |               父进程中的内存映射存留到子进程中               |                         去除内存映射                         |                         去除内存映射                         |
|    Posix共享内存区     |               父进程中的内存映射存留到子进程中               |                         去除内存映射                         |                         去除内存映射                         |
|   System V共享内存区   |            附接着的共享内存区在子进程中继续附接着            |                  断开所有附接着的共享内存区                  |                  断开所有附接着的共享内存区                  |
|           门           | 子进程取得父进程的所有打开着的描述符, 但是客户在门描述符上激活其过程时, 只有父进程是服务器 |   所有门描述符都应关闭, 因为它们创建时设置了 FD_CLOEXEC位    |                    关闭所有打开着的描述符                    |



**(无名同步变量:  互斥锁, 条件变量, 读写锁, 基于内存的信号量)  .  从一个具有多个小城的进程中调用fork 将变得混乱不堪.**

**System  V  IPC三种形式没有打开或关闭的说法**





## 出错处理的包裹函数



```c++
#include <errno.h>
#include <sys/errno.h>

void Sem_post(sem_t *sem)
{
  if ( sem_post(sem) == -1 )
    err_sys("sem_post error");
}

void err_sys(const char* c){
  strerror(errno);
}

// 给 thread_mutex_lock() 定义的包裹函数
void
Pthread_mutex_lock(pthread_mutex_t* mptr){
    int n =0;
  	if( ( n = pthread_mutex_lock(mptr)) ==  0)
  	    return;
  
	  errno = n;
	  err_sys("pthread_mutex_lock error");
}

```

**线程函数出错时, 并不设置标准的 UNIX errno 变量,  相反 本该设置 errno 的值改由线程函数作为其返回值返回调用者. 就这需要每次调用任何一个线程函数时, 都得分配一个变量来保存函数返回值, 然后再调用 自定义的 err_sys() 函数**



## UNIXerrno值

**每当在一个UNIX 函数中发生错误时, 全局变量errno将被设置为一个指示错误类型的正数, 函数本身这通常返回-1**

**errno 的值只在某个函数发生错误时设置. 如果函数不返回错误, errno的值就无定义**

- 多线程环境中, 每个线程必须有自己的errno变量 . 提供一个局限于线程的errno变量的隐式请求时自动处理的, 不过通常要告诉编译器锁编译的程序时可重入的.
    - **给编译器指定类似  `-D_REENTRANT 或 -D_POSIX_C_SOURCE=199506L`  这样的命令行喧嚣时较典型的方法**
        - `<errno.h>` 头文件往往把 errno定义成一个宏, 当常值 _REENTRANT 有定义时, 该宏就拓展成一个函数, 由他访问 errno 变量的某个局限于线程的副本



