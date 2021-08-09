- [Unix管道](#Unix管道)
- [Socket管道](#Socket管道)



## Unix管道

```c
#include <unistd.h>
int  pipe (int fildes[2] );

/*  参数: fildes : 两个int 的数组, 传出参数, 使用一个int数组来接收
		返回值:  成功0,  错误 -1 , 并设置 errno
		
		管道是全双工的.
*/
```



## Socket管道

```c
#include <sys/socket.h>
#include <sys/types.h>
int  socketpair(int domain, int type, int protocol, int socket_vector[2]);
/*
	参数:  domain:  AF_INET, AF_INET6 , PF_INET, PF_INET6, PF_UNIX
   				type:  SOCK_STREAM,  SOCK_DGRAM
			protocol:  0
 socket_vector:  接收套接字的数组, 传出参数, 使用一个int数组来接收

	返回值: 成功 返回 0； 出错将返回值-1，并且将全局变量errno设置为指示错误。
*/	
```

