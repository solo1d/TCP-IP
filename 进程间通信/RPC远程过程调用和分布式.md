- **sun 是一个Unix操作系统.**
- ==**文件名以 `.x` 结尾的文件都称为 RPC说明书文件, 它们定义了服务器以及这些过程的参数和结果**==
- **过程调用 其实就是调用函数**
- 三种类型的过程调用
    - **本地过程调用**
        - 就是调用本地函数, 获得返回值和移交控制权, 是同步的, 调用者要等到调用过程返回后才能重新获得控制权
    - **远程过程调用 RPC ( 门 )**
        - **调用与被调用者属于不同进程, 被调用者会创建一个门,双方以函数参数和返回值交换信息, 可同步可异步**
    - **主机间的远程过程调用 RPC**
        - **就是通过网络来使用 RPC 门**
- **需要显示的通过网络来实现一个分布式系统, 就称为显示网络编程**
- **也通过网络来实现一个分布式系统, 但是网络通信内容对程序员不可见, 称为 隐式网络编程**



>==**远程调用的设计模式 就是 客户端传递参数进行远程调用, 服务器进行响应和执行, 并把结果传递给客户端**==





- [RPC说明书文件范例](#RPC说明书文件范例)
- [客户端获取句柄函数](#客户端获取句柄函数)
- [rpc默认端口映射器的端口为111](#rpc默认端口映射器的端口为111)
- [rpcgen编译命令参数](#rpcgen编译命令参数)
- [简单服务器和客户端代码范例](#简单服务器和客户端代码范例)
    - [服务器](#服务器)
        - [服务器需要编写的代码](#[服务器](#服务器)需要编写的代码)
        - [服务器无需编写的代码_通过rpcgen编译出来的而且不同的话可以无视](#服务器无需编写的代码_通过rpcgen编译出来的而且不同的话可以无视)
    - [客户端](#客户端)
        - [客户端需要编写的代码](#客户端需要编写的代码)
        - [客户端无需编写的代码_从服务器那里拿来的](#客户端无需编写的代码_从服务器那里拿来的)
- 



## 

## 



## RPC说明书文件范例

==**文件名以 `.x` 结尾的文件都称为 RPC说明书文件, 它们定义了服务器以及这些过程的参数和结果**==

所有的程序员都需要写是一个客户端程序([client.c](https://www.cs.rutgers.edu/~pxk/417/notes/rpc/client.c)),服务器功能([server.c](https://www.cs.rutgers.edu/~pxk/417/notes/rpc/server.c))和 RPC 接口定义([date.x](https://www.cs.rutgers.edu/~pxk/417/notes/rpc/date.x))。当 RPC 接口定义(后缀为.x 的文件，例如 date.x)是用 rpcgen 编译的,会创建三个或四个文件 , 其中有 服务器可直接使用的 main函数文件



==**该文件是可以进行编译的, 命令为 `rpcgen 文件名.x`,  并且会自动生成多个.c和 .h 文件.**==

```xquery
/* 文件名为  square.x   ,是RPC说明书文件 */
struct square_in{    /* 定义的一个结构体, 用来当 参数 */
	long  arg1;
};

struct square_out{    /* 定义的一个结构体, 用来当 返回值 */
	long res1;
};

program SQUARE_PROG {
	version SQUARE_VERS {
	    square_out  SQUAREPROC(square_in) = 1;   /* 当函数定义看, 返回值 函数名(参数) ,过程号 =1  */
	} = 1;           /* 与上面的 version 对应,  版本号 version =1  */
} = 0x31230000;    /* 程序号 ,  与上面的  program 对应*/

/*  详解: 
   定义一个名为 SQUARE_PROG 的PRC 程序.
      它由一个版本 SQUARE_VERS 构成.
         该版本内由定义了 单个名为 SQUAREPROC 的过程(函数), 在.c文件中会转换成 squareproc_1 ,1是版本号
            该过程返回值是 square_out,  参数是 square_in
               还给予了 该过程一个值为 1 的过程号,  SQUARE_VERS 过程号 =1  
       版本号 version 也为 1
    程序号设置为 0x31230000 一个 32位十六进制值.
        程序号划分:  0x00000000 ~ 0x1fffffff  由 sun 定义
                   0x20000000 ~ 0x3fffffff  由 用户 定义
                   0x40000000 ~ 0x5fffffff  临时 ( 供客户标写的应用程序用 )
                   0x60000000 ~ 0xffffffff  保留 未使用  
*/
```



# 客户端获取句柄函数

**编译时需要添加选项  `-pthread`** 

```c
#include <rpc/rpc.h>
CLIENT* clnt_create(const char *host, unsigned  long program, 
                    unsigned long versnum, const char *protocol);
// 获取客户句柄, 这是客户端调用的函数 ,不关心句柄指向什么内容, 可能是系统维护的某个信息结构
/* 参数
   host: 是服务器的主机名或IP地址.
program: 程序名, 来自 PRC.x 文件内 , program 程序名{} = 123;, PRC文件编译后可直接写入 程序名
versnum: 版本号, 来自 PRC.x 文件内 , version 版本名{} = 456;  PRC文件编译后可直接写入 版本名
protocol: 协议选择,  "tcp"  或  "udp"

返回值: 成功则返回 非空客户句柄, 出错则为 NULL
*/
范例:  CLIENT* cl = clnt_create("1.1.1.1", 1, 1, "tcp");
```



# rpc默认端口映射器的端口为111

**端口映射器本身就是一个 RPC程序, 服务器就是使用端口映射器注册自身的临时端口**

**服务器的主体 RPC程序会绑定一个临时端口, 并将这个临时端口注册到 RPC 端口映射器.**



**当 RPC 程序进行通信之前, 操作系统的 RPC端口注册程序会率先执行沟通, 获得服务器PRC端口映射器的内容, 知道即将通讯的服务器进程绑定的临时端口, 然后才会让 客户端与服务器进行连接**



**通过 `rpcinfo -p`  命令 可以查看已经在端口映射器注册的RPC程序**

# rpcgen编译命令参数

```bash
 sudo apt install  -y rpcgen rpcbind    #先安装依赖

rpcgen -C square.x    #告诉rcpgen 在相应的 square.h 中生成 ANSI C 原型

rpcgen -lnsl  square.x  # 指定 solaris 系统下存放网络支撑函数的系统函数库

rpcgen -M   square.x   #让服务器代码变得线程安全
rpcgen -A   square.x   #让服务器更具处理新客户请求的需要自动创建新线程, 可以与 -M 选项一起使用
```







# 简单服务器和客户端代码范例

## 服务器



**需要先安装依赖 `sudo apt install rpcbind`**

**服务器需要两个文件,  其中 `.x` 文件是与客户端共享的, 也就是 客户端要进行相应的编译**

**编译时 要添加参数 `-pthread`**

## 服务器需要编写的代码

```c
// 文件名为    square_prog_1.c     是服务器提供给客户端接口的具体实现
#include <stdio.h>
#include "square.h"

square_out*
squareproc_1_svc(square_in* inp, struct svc_req* rqstp){
    static square_out out;
    out.res1 = inp->arg1 * inp->arg1;
    return &out;
}
```

```idl
/* 文件名为  square.x   ,是RPC说明书文件, 通过 rpcgen 编译 */
struct square_in{    /* 定义的一个结构体, 用来当 参数 */
	long  arg1;
};

struct square_out{    /* 定义的一个结构体, 用来当 返回值 */
	long res1;
};

program SQUARE_PROG {
	version SQUARE_VERS {
	    square_out  SQUAREPROC(square_in) = 1;   /* 当函数定义看, 返回值 函数名(参数) ,过程号 =1  */
	} = 1;           /* 与上面的 version 对应,  版本号 version =1  */
} = 0x31230000;    /* 程序号 ,  与上面的  program 对应*/
```





## 服务器无需编写的代码_通过rpcgen编译出来的而且不同的话可以无视

编译出来的有很多文件, 其中有的是服务器的, 有的是客户端的

```c
//  square_xdr.c 文件
#include "square.h"

bool_t
xdr_square_in(xdrs, objp)
	XDR *xdrs;
	square_in *objp;
{

	if (!xdr_long(xdrs, &objp->arg1))
		return (FALSE);
	return (TRUE);
}

bool_t
xdr_square_out(xdrs, objp)
	XDR *xdrs;
	square_out *objp;
{

	if (!xdr_long(xdrs, &objp->res1))
		return (FALSE);
	return (TRUE);
}
```



```c
//   square.h   头文件
#ifndef _SQUARE_H_RPCGEN
#define _SQUARE_H_RPCGEN

#define RPCGEN_VERSION	199506

#include <rpc/rpc.h>
#include <unistd.h>
#include <string.h>

struct square_in {
	long arg1;
};
typedef struct square_in square_in;
#ifdef __cplusplus
extern "C" bool_t xdr_square_in(XDR *, square_in*);
#elif __STDC__
extern  bool_t xdr_square_in(XDR *, square_in*);
#else /* Old Style C */
bool_t xdr_square_in();
#endif /* Old Style C */


struct square_out {
	long res1;
};
typedef struct square_out square_out;
#ifdef __cplusplus
extern "C" bool_t xdr_square_out(XDR *, square_out*);
#elif __STDC__
extern  bool_t xdr_square_out(XDR *, square_out*);
#else /* Old Style C */
bool_t xdr_square_out();
#endif /* Old Style C */


#define SQUARE_PROG ((rpc_uint)0x31230000)
#define SQUARE_VERS ((rpc_uint)1)

#ifdef __cplusplus
#define SQUAREPROC ((rpc_uint)1)
extern "C" square_out * squareproc_1(square_in *, CLIENT *);
extern "C" square_out * squareproc_1_svc(square_in *, struct svc_req *);

#elif __STDC__
#define SQUAREPROC ((rpc_uint)1)
extern  square_out * squareproc_1(square_in *, CLIENT *);
extern  square_out * squareproc_1_svc(square_in *, struct svc_req *);

#else /* Old Style C */
#define SQUAREPROC ((rpc_uint)1)
extern  square_out * squareproc_1();
extern  square_out * squareproc_1_svc();
#endif /* Old Style C */

#endif /* !_SQUARE_H_RPCGEN */
```



```c
/*   square_svc.c  文件,  是自动编译来的  提供给 服务器使用的 main() 函数文件
 * Please do not edit this file.
 * It was generated using rpcgen.
 */

#include "square.h"
#include <sys/ioctl.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <netdb.h>
#include <signal.h>
#include <sys/ttycom.h>
#include <memory.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <syslog.h>
#include <rpc/rpc.h>

#ifdef __STDC__
#define SIG_PF void(*)(int)
#endif

#ifdef DEBUG
#define RPC_SVC_FG
#endif

#define _RPCSVC_CLOSEDOWN 120
static int _rpcpmstart;		/* Started by a port monitor ? */
static int _rpcfdtype;		/* Whether Stream or Datagram ? */
static int _rpcsvcdirty;	/* Still serving ? */

static
void _msgout(msg)
	char *msg;
{
#ifdef RPC_SVC_FG
	if (_rpcpmstart)
		syslog(LOG_ERR, "%s", msg);
	else
		(void) fprintf(stderr, "%s\n", msg);
#else
	syslog(LOG_ERR, "%s", msg);
#endif
}

static void
closedown()
{
	if (_rpcsvcdirty == 0) {
		extern fd_set svc_fdset;
		static int size;
		int i, openfd;

		if (_rpcfdtype == SOCK_DGRAM)
			exit(0);
		if (size == 0) {
			size = getdtablesize();
		}
		for (i = 0, openfd = 0; i < size && openfd < 2; i++)
			if (FD_ISSET(i, &svc_fdset))
				openfd++;
		if (openfd <= (_rpcpmstart?0:1))
			exit(0);
	}
	(void) alarm(_RPCSVC_CLOSEDOWN);
}

static void
square_prog_1(rqstp, transp)
	struct svc_req *rqstp;
	SVCXPRT *transp;
{
	union {
		square_in squareproc_1_arg;
	} argument;
	char *result;
	bool_t (*xdr_argument)(), (*xdr_result)();
	char *(*local)();

	_rpcsvcdirty = 1;
	switch (rqstp->rq_proc) {
	case NULLPROC:
		(void) svc_sendreply(transp, (xdrproc_t) xdr_void, (char *)NULL);
		_rpcsvcdirty = 0;
		return;

	case SQUAREPROC:
		xdr_argument = xdr_square_in;
		xdr_result = xdr_square_out;
		local = (char *(*)()) squareproc_1_svc;
		break;

	default:
		svcerr_noproc(transp);
		_rpcsvcdirty = 0;
		return;
	}
	(void) memset((char *)&argument, 0, sizeof (argument));
	if (!svc_getargs(transp, xdr_argument, (caddr_t) &argument)) {
		svcerr_decode(transp);
		_rpcsvcdirty = 0;
		return;
	}
	result = (*local)(&argument, rqstp);
	if (result != NULL && !svc_sendreply(transp, (xdrproc_t) xdr_result, result)) {
		svcerr_systemerr(transp);
	}
	if (!svc_freeargs(transp, xdr_argument, (caddr_t) &argument)) {
		_msgout("unable to free arguments");
		exit(1);
	}
	_rpcsvcdirty = 0;
	return;
}



int
main(argc, argv)
int argc;
char *argv[];
{
	SVCXPRT *transp = NULL;
	int sock;
	int proto = 0;
	struct sockaddr_in saddr;
	int asize = sizeof (saddr);

	if (getsockname(0, (struct sockaddr *)&saddr, &asize) == 0) {
		int ssize = sizeof (int);

		if (saddr.sin_family != AF_INET)
			exit(1);
		if (getsockopt(0, SOL_SOCKET, SO_TYPE,
				(char *)&_rpcfdtype, &ssize) == -1)
			exit(1);
		sock = 0;
		_rpcpmstart = 1;
		proto = 0;
		openlog("square", LOG_PID, LOG_DAEMON);
	} else {
#ifndef RPC_SVC_FG
		int size;
		int pid, i;

		pid = fork();
		if (pid < 0) {
			perror("cannot fork");
			exit(1);
		}
		if (pid)
			exit(0);
		size = getdtablesize();
		for (i = 0; i < size; i++)
			(void) close(i);
		i = open("/dev/console", 2);
		(void) dup2(i, 1);
		(void) dup2(i, 2);
		i = open("/dev/tty", 2);
		if (i >= 0) {
			(void) ioctl(i, TIOCNOTTY, (char *)NULL);
			(void) close(i);
		}
		openlog("square", LOG_PID, LOG_DAEMON);
#endif
		sock = RPC_ANYSOCK;
//		(void) pmap_unset(SQUARE_PROG, SQUARE_VERS);
	}

	if ((_rpcfdtype == 0) || (_rpcfdtype == SOCK_DGRAM)) {
		transp = svcudp_create(sock);
		if (transp == NULL) {
			_msgout("cannot create udp service.");
			exit(1);
		}
		if (!_rpcpmstart)
			proto = IPPROTO_UDP;
		if (!svc_register(transp, SQUARE_PROG, SQUARE_VERS, square_prog_1, proto)) {
			_msgout("unable to register (SQUARE_PROG, SQUARE_VERS, udp).");
			exit(1);
		}
	}

	if ((_rpcfdtype == 0) || (_rpcfdtype == SOCK_STREAM)) {
		if (_rpcpmstart)
			transp = svcfd_create(sock, 0, 0);
		else
			transp = svctcp_create(sock, 0, 0);
		if (transp == NULL) {
			_msgout("cannot create tcp service.");
			exit(1);
		}
		if (!_rpcpmstart)
			proto = IPPROTO_TCP;
		if (!svc_register(transp, SQUARE_PROG, SQUARE_VERS, square_prog_1, proto)) {
			_msgout("unable to register (SQUARE_PROG, SQUARE_VERS, tcp).");
			exit(1);
		}
	}

	if (transp == (SVCXPRT *)NULL) {
		_msgout("could not create a handle");
		exit(1);
	}
	if (_rpcpmstart) {
		(void) signal(SIGALRM, (void(*)()) closedown);
		(void) alarm(_RPCSVC_CLOSEDOWN);
	}
	svc_run();
	_msgout("svc_run returned");
	exit(1);
	/* NOTREACHED */
}

```



## 客户端

**需要先安装依赖 `sudo apt install rpcbind`**

**会从服务器那里 拿出一些文件出来, 也就是 rpcgen 生成出来的文件**



## 客户端需要编写的代码

```c
#include "square.h"
#include <pthread.h>
#include <rpc/rpc.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <errno.h>

// client
int main(void){
    CLIENT *cl;
    square_in  in;
    square_out *outp;
    const char * NAME  = "pi.local";
    cl = clnt_create(NAME, SQUARE_PROG, SQUARE_VERS, "tcp");
    in.arg1 =  12;
    if ((outp = squareproc_1(&in, cl)) == NULL){
        fprintf(stderr, "%s", clnt_sperror(cl, NAME));
        err_quit("squareproc_1", "");
    }
    printf("result : %ld\n", outp->res1);
    exit(errno);
}
```





## 客户端无需编写的代码_从服务器那里拿来的

```c
//   square.h   头文件
#ifndef _SQUARE_H_RPCGEN
#define _SQUARE_H_RPCGEN

#define RPCGEN_VERSION	199506

#include <rpc/rpc.h>
#include <unistd.h>
#include <string.h>

struct square_in {
	long arg1;
};
typedef struct square_in square_in;
#ifdef __cplusplus
extern "C" bool_t xdr_square_in(XDR *, square_in*);
#elif __STDC__
extern  bool_t xdr_square_in(XDR *, square_in*);
#else /* Old Style C */
bool_t xdr_square_in();
#endif /* Old Style C */


struct square_out {
	long res1;
};
typedef struct square_out square_out;
#ifdef __cplusplus
extern "C" bool_t xdr_square_out(XDR *, square_out*);
#elif __STDC__
extern  bool_t xdr_square_out(XDR *, square_out*);
#else /* Old Style C */
bool_t xdr_square_out();
#endif /* Old Style C */


#define SQUARE_PROG ((rpc_uint)0x31230000)
#define SQUARE_VERS ((rpc_uint)1)

#ifdef __cplusplus
#define SQUAREPROC ((rpc_uint)1)
extern "C" square_out * squareproc_1(square_in *, CLIENT *);
extern "C" square_out * squareproc_1_svc(square_in *, struct svc_req *);

#elif __STDC__
#define SQUAREPROC ((rpc_uint)1)
extern  square_out * squareproc_1(square_in *, CLIENT *);
extern  square_out * squareproc_1_svc(square_in *, struct svc_req *);

#else /* Old Style C */
#define SQUAREPROC ((rpc_uint)1)
extern  square_out * squareproc_1();
extern  square_out * squareproc_1_svc();
#endif /* Old Style C */

#endif /* !_SQUARE_H_RPCGEN */
```



```c
//  square_clnt.c
#include "square.h"

/* Default timeout can be changed using clnt_control() */
static struct timeval TIMEOUT = { 25, 0 };

square_out *
squareproc_1(argp, clnt)
	square_in *argp;
	CLIENT *clnt;
{
	static square_out clnt_res;

	memset((char *)&clnt_res, 0, sizeof(clnt_res));
	if (clnt_call(clnt, SQUAREPROC, xdr_square_in, argp, xdr_square_out, &clnt_res, TIMEOUT) != RPC_SUCCESS)
		return (NULL);
	return (&clnt_res);
}

```





```c
//  square_xdr.c
#include "square.h"

bool_t
xdr_square_in(xdrs, objp)
	XDR *xdrs;
	square_in *objp;
{

	if (!xdr_long(xdrs, &objp->arg1))
		return (FALSE);
	return (TRUE);
}

bool_t
xdr_square_out(xdrs, objp)
	XDR *xdrs;
	square_out *objp;
{

	if (!xdr_long(xdrs, &objp->res1))
		return (FALSE);
	return (TRUE);
}
```

