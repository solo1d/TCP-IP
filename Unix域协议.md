- [Unix域套接字地址结构](#Unix域套接字地址结构)
- [创建Unix域套接字和绑定](#创建Unix域套接字和绑定)
- [socketpair函数](#socketpair函数)
- [Unix域套接字函数和限制差异](#Unix域套接字函数和限制差异)
- [Unix域字节流客户的和服务器程序](#Unix域字节流客户的和服务器程序)
- [描述符传递](#描述符传递)
- 









**Unix域协议并不是一个实际的协议族, 而是在单个主机上执行客户/服务器通信的一种方法, 所用的API就是普通套接字API , 就是IPC的方法之一**

- **Unix域提供两类套接字:**
  - 字节流套接字( 类似于 TCP)
  - 数据报套接字 (类似于 UDP)
- **使用 Unix域套接字的理由:**
  - Unix域套接字往往比通信两端位于同一个主机的TCP套接字快出一倍.
  - **Unix域套接字可用于在同一个主机上的不同进程之间传递描述符**
  - **Unix域套接字较新的实现把用户的凭证 (用户ID,组ID) 提供给服务器, 从而能够提供额外的安全检查措施**

> **Unix域中用于标识客户和服务器的协议地址是普通文件系统中的路径名, 这些路径名不是普通的Unix文件, 除非把这些文件和Unix域套接字关联起来, 否则无法读写这些文件**



# Unix域套接字地址结构

```c
#include <sys/un.h>
// UNIX IPC域的定义。 这个结构可以直接交给 socket() 系统调用来使用
struct  sockaddr_un {
	unsigned char   sun_len;        /* 当前结构长度 , 1字节 */
	sa_family_t     sun_family;     /* AF_UNIX 或 AF_LOCAL , 1字节*/
	char            sun_path[104];  /* 路径名, 必须以空字符 \0 结尾 */
};

// 下面的宏定义返回 当前结构体的长度参数, len+family+ strlen(path)已用长度, 可用作 bind参数
size_t SUN_LEN( struct sockaddr_un* un); 
// 具体实现为:
#define SUN_LEN(su)   (sizeof(*(su)) - sizeof((su)->sun_path) + strlen((su)->sun_path))
	
```



### 创建Unix域套接字和绑定

```c
// argv[1]  是个本地文件路径 , 该文件并不存在
int
main(int argc, char **argv)
{
    int  sockfd;
    socklen_t len;
    struct sockaddr_un  addr1, addr2;
    if(argc != 2)
        err_quit("usage: unixbind <pathname>");
    sockfd = socket(AF_LOCAL, SOCK_STREAM, 0);
    unlink(argv[1]);  // 将指定文件删除, 如果该文件被其他进程打开,那么会等待关闭,然后删除文件
    bzero(&addr1, sizeof(addr1));
    addr1.sun_family = AF_LOCAL;
    strncpy(addr1.sun_path, argv[1], sizeof(addr1.sun_path)-1);
    Bind(sockfd, (SA*)&addr1, SUN_LEN(&addr1));   // 这里进行绑定, 那么就会创建一个文件 S 类型
    len = sizeof(addr2);
    Getsockname(sockfd, (SA*)&addr2, &len);
    printf("bound name =%s , returned len = %d\n", addr1.sun_path, len);
    exit(0);
}
```



## socketpair函数

```c
#include <sys/socket.h>
int  socketpair (int family, int type, int protocol, int sockfd[2] );
/*  该函数会创建两个随后连接起来的套接字, 类似于 pipe函数
	参数: family : 必须是 AF_LOCAL
	       type : SOCK_STREAM(流管道 全双工)  或 SOCK_DGRAM(数据报)
     protocol : 必须为0
    sockfd[2] : 函数会将 新创建的两个套接字描述符通过该参数进行传出, 这两个套接字没有命名 不涉及隐式bind

返回值: 成功返回非0,  出错则为 -1
*/
```



## Unix域套接字函数和限制差异

- **Unix域套接字于普通TCP套接字之间有所差异和限制**
  - 由 bind 创建的路径名默认访问权限应为 0777 , 并按照当前的 `umask`  进行修正
  - 与Unix域 套接字关联的路径名必须是一个绝对路径名
  - 在 connect 调用中指定的路径名必须是一个当前绑定在某个打开的Unix域 套接字上的路径名, 而且它们的套接字类型( 字节流或数据报) 也必须一致.
  - 调用 connect 连接一个 Unix域 套接字涉及的权限测试等同于调用 open 以只写方式访问相应的路径名
  - Unix域字节流套接字类似 TCP 套接字: 它们都为进程提供一个无边界的字节流接口
  - 如果对某个Unix域 套接字的 connect 调用 发现这个监听套接字队列已满,就立即返回 ECONNREFUSED 错误.
  - Unix域数据报套接字类似 UDP 套接字: 它们都为进程提供一个保留边界的不可靠的数据报服务
  - 向未绑定Unix域套接字上发送数据报不会自动给这个套接字捆绑路径名.调用  connect 也是一样



## Unix域字节流客户的和服务器程序

```c
// 服务器

#include <sys/socket.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <netinet/in.h>
#include <sys/types.h>
#include <sys/time.h>
#include <sys/event.h>
#include <errno.h>
#include <sys/stat.h>
#include <stdlib.h>
#include <sys/un.h>

#define UNIXSTR_PATH   "/tmp/unix.str"
int
main(int argc, char **argv)
{
    int  listenfd, connfd;
    pid_t childpid;
    socklen_t   clilen;
    struct sockaddr_un  cliaddr , servaddr;
    void  sig_chld(int);
    listenfd = Socket(AF_LOCAL, SOCK_STREAM, 0);
    unlink(UNIXSTR_PATH);  // "/tmp/unix.str"
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sun_family = AF_LOCAL;
    strcpy(servaddr.sun_path, UNIXSTR_PATH);
    
    Bind(listenfd, (SA*)&servaddr, sizeof(servaddr));
    Listen(listenfd, LISTENQ);
    
    Signal(SIGCHLD, sig_chld);
    for(;;){
        clilen = sizeof(cliaddr);
        if( (connfd = accept(listenfd, (SA*)&cliaddr, &clilen)) < 0){
            if(errno == EINTR)
                continue;
            else
                err_sys("accept error");
        }
        if( (childpid = Fork()) == 0){
            Close(listenfd);
            str_echo(connfd);
            exit(0);
        }
        Close(connfd);
    }
}
```

```c
//  客户端
#include <netinet/ip.h>
#include <netinet/ip6.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <time.h>
#define UNIXSTR_PATH   "/tmp/unix.str"
int
main(int argc, char **argv)
{
    int  sockfd;
    struct sockaddr_un  servaddr;
    sockfd = Socket(AF_LOCAL, SOCK_STREAM, 0   );
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sun_family = AF_LOCAL;
     
    strcpy(servaddr.sun_path, UNIXSTR_PATH);
    Connect(sockfd, (SA*)&servaddr, sizeof(servaddr));
    str_cli(stdin, sockfd);
    exit(0);
}
```



## 描述符传递

- **Unix系统提供用于从一个进程向任意其他进程传递任一打开的描述符方法** (描述符不限类型)
  - **需要在这两个进程之间创建一个Unix域套接字.**
    - 随后使用 `sendmsg` 跨这个套接字发送一个特殊消息,这个消息由内核来专门处理, 会把打开的描述符从发送进程传递到接收进程
- **两个进程间传递描述符涉及步骤如下:**
  - **创建字节流的或数据报的Unix域套接字 socketpair**
  - **发送进程通过调用返回描述符的任一Unix函数打开一个描述符,这些函数的例子有 `open , pipe, mkfifo, socket 和 accept` , 进程间传递的描述符不限类型**
  - **发送进程创建一个 `msghdr结构`, 其中含有待传递的描述符,  一般是 `msghdr.msg_control成员`,  在使用 `sendmsg` 函数发送这个描述符之后 立即关闭该描述符, 那么对于接收进程来说 仍保持打开状态 , 发送描述符会使该描述符的引用计数加1** 
  - **接收进程调用 `recvmsg` 在来自第一步的Unix域套接字上接收这个描述符,  传递的描述符并不是传递一个描述符号, 而是涉及在接收进程中创建一个新的描述符 (inode),  调用 `recvmsg` 避免使用MSG_PEEK标志** 

```c
// 主程序, 编译为 main.out
#include <sys/socket.h>
#include <sys/param.h>
#include <errno.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <netinet/in.h>
#include <sys/types.h>
#include <sys/time.h>
#include <sys/event.h>
#include <errno.h>
#include <sys/stat.h>
#include <stdlib.h>
#include <sys/un.h>
#define  BUFFSIZE  8192

int my_open(const char* pathname, int mode);
ssize_t  read_fd(int fd, void *ptr, size_t nbytes, int *recvfd);

int
main(int argc, char **argv)
{
    int  fd, n;
    char buff[BUFFSIZE];  // 8192
    
    if(argc != 2)
        err_quit("usage : mycat <pathname>");
    if( (fd = my_open(argv[1], O_RDONLY)) < 0)
        err_sys("cannot open %s", argv[1]);
    while ((n = Read(fd, buff, BUFFSIZE)) >0)
        Write(STDOUT_FILENO, buff , n);  // STDOUT_FILENO 标准输出描述符
    
}

int my_open(const char* pathname, int mode){
    int     fd, sockfd[2], status;
    pid_t   childpid;
    char    c, argsockfd[10], argmode[10];
    Socketpair(AF_LOCAL, SOCK_STREAM, 0, sockfd);
    if((childpid = Fork()) == 0){
        Close(sockfd[0]);
        snprintf(argsockfd, sizeof(argsockfd), "%d", sockfd[1]); // 描述符
        snprintf(argmode, sizeof(argmode), "%d", mode);
        execl("./openfile", "openfile", argsockfd, pathname, argmode, (char*)NULL);
        err_sys("execl error");
    }
    Close(sockfd[1]);
    Waitpid(childpid, &status, 0);  // 回收子进程, 并获得结束状态
    if(WIFEXITED (status)  == 0) // 子进程非正常退出
        err_quit("child did not terminate");
    
    if( (status = WEXITSTATUS(status)) == 0)
        Read_fd(sockfd[0], &c, 1, &fd);
    else{
        errno  = status;
        fd = -1;
    }
    
    Close(sockfd[0]);
    return(fd);
}


ssize_t
read_fd(int fd, void *ptr, size_t nbytes, int *recvfd)
{
	struct msghdr	msg;
	struct iovec	iov[1];
	ssize_t			n;

#ifdef	HAVE_MSGHDR_MSG_CONTROL
	union {
	  struct cmsghdr	cm;
	  char				control[CMSG_SPACE(sizeof(int))];
	} control_un;
	struct cmsghdr	*cmptr;

	msg.msg_control = control_un.control;
	msg.msg_controllen = sizeof(control_un.control);
#else
	int				newfd;

	msg.msg_accrights = (caddr_t) &newfd;
	msg.msg_accrightslen = sizeof(int);
#endif

	msg.msg_name = NULL;
	msg.msg_namelen = 0;

	iov[0].iov_base = ptr;
	iov[0].iov_len = nbytes;
	msg.msg_iov = iov;
	msg.msg_iovlen = 1;

	if ( (n = recvmsg(fd, &msg, 0)) <= 0)
		return(n);

#ifdef	HAVE_MSGHDR_MSG_CONTROL
	if ( (cmptr = CMSG_FIRSTHDR(&msg)) != NULL &&
	    cmptr->cmsg_len == CMSG_LEN(sizeof(int))) {
		if (cmptr->cmsg_level != SOL_SOCKET)
			err_quit("control level != SOL_SOCKET");
		if (cmptr->cmsg_type != SCM_RIGHTS)
			err_quit("control type != SCM_RIGHTS");
		*recvfd = *((int *) CMSG_DATA(cmptr));
	} else
		*recvfd = -1;		/* descriptor was not passed */
#else
/* *INDENT-OFF* */
	if (msg.msg_accrightslen == sizeof(int))
		*recvfd = newfd;
	else
		*recvfd = -1;		/* descriptor was not passed */
/* *INDENT-ON* */
#endif

	return(n);
}
/* end read_fd */
```

```c
// 辅助程序,  编译为 openfile   , 为主程序提供服务
#include <netinet/ip.h>
#include <netinet/ip6.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <time.h>
#include    "unp.h"


ssize_t write_fd(int fd, void* ptr, size_t nbytes, int sendfd);

int
mainccc1(int argc, char **argv)
{
    int fd;
    if(argc != 4)
        err_quit("openfile <sockfd#> <filename> <mode>");
    if( (fd = open(argv[2], atoi(argv[3]))) < 0)
        exit( (errno > 0) ? errno : 255);
    if (write_fd(atoi(argv[1]), "", 1, fd) < 0)  // 保持发送一个字节,区别无数据
       exit((errno>0) ? errno : 255);

    exit(0);
    return 0;
}

ssize_t write_fd(int fd, void* ptr, size_t nbytes, int sendfd){
    struct  msghdr msg;
    struct  iovec  iov[1];
    
#ifdef  HAVE_MSGHDR_MSG_CONTROL
    union{
        struct cmsghdr  cm;
        char   control[CMSG_SPACE(sizeof(int))];
    }control_un;
    struct  cmsghdr* cmptr;
    
    msg.msg_control = control_un.control;
    msg.msg_controllen = sizeof(control_un.control);
    cmptr = CMSG_FIRSTHDR(&msg);
    cmptr->cmsg_len = CMSG_LEN(sizeof(int));
    cmptr->cmsg_level = SOL_SOCKET;
    cmptr->cmsg_type = SCM_RIGHTS;
    *((int*)CMSG_DATA(cmptr)) = sendfd;  // 转换为 int* 并解指针赋值
#else
    msg.msg_accrights = (caddr_t) & sendfd;
    msg.msg_accrightslen = sizeof(int);
#endif
    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    
    iov[0].iov_base = ptr;
    iov[0].iov_len = nbytes;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;
    
    return (sendmsg(fd, &msg, 0));
}

```









