**信号驱动式IO 是指进程预先告知内核, 使得当某个描述符上发生某事时, 内核使用信号通知相关进程.**

唯一的用途 就是 UDP的NTP服务器了



- [套接字的信号驱动式IO](#套接字的信号驱动式IO)
    - [UDP套接字信号驱动式IO的SIGIO信号](#UDP套接字信号驱动式IO的SIGIO信号)
    - [TCP套接字信号驱动式IO的SIGIO信号](#TCP套接字信号驱动式IO的SIGIO信号)
    - 



# 套接字的信号驱动式IO

- **套接字使用信号驱动式IO (SIGIO) 需要进程执行的步骤**

    - **建立 SIGIO信号的 信号处理函数, `sigaction()`**

    - **设置该套接字的属主, 通常使用 `fcntl()` 的 `F_SETOWN` 命令设置**

    - **开启该套接字的信号驱动式IO, 通常通过使用 `fcntl` 的 `F_SETFL` 命令打开 `O_ASYNC` 标志完成**

    - ```cpp
        #include <csignal>
        #include <fcntl.h>
        void func(int signal, siginfo_t* siginfo, void* t){  }
        int main(void)
        {
          struct  sigaction arc;
          memset(&arc, 0, sizeof(arc));
          arc.sa_sigaction = func;
          arc.sa_flags |= SA_RESTART;
          sigemptyset(&arc.sa_mask);
          sigaction(SIGIO, &(arc), nullptr);
          int fd =  socket(AF_INET, SOCK_STREAM, 0);
          fcntl(fd, F_SETOWN|O_ASYNC); 
        }
        ```

    - 



## UDP套接字信号驱动式IO的SIGIO信号

- **UDP使用信号驱动式IO很简单, SIGIO信号在发生以下事件时产生:**
    - ==**数据到达套接字**==
    - ==**套接字上发生异步错误  ( udp套接字必须调用 connect 连接, 才能接收异步错误)**==



## TCP套接字信号驱动式IO的SIGIO信号

**信号驱动式IO并不适合 TCP 套接字**

- **产生SIGIO信号的事件:**
    - 监听套接字上某个连接请求已经完成
    - 某个断连请求已经发起
    - 某个断连请求已经完成
    - 某个数据之半已经关闭
    - 数据到达套接字
    - 数据已经从套接字发送走  (即输出缓冲区有空闲空间)
    - 发生某个异步错误
