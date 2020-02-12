**sctp 代码编译时,必须添加 `-lsctp` 选项.**

```bash
$gcc -Wall  -o a.out a.c -lsctp
```

**sctp 主要需求的头文件有: `<sys/types> , <sys/socket.h>, <netinet/sctp.h>`**

# 目录

- [安装和开启SCTP支持](#安装和开启SCTP支持)
- [基本SCTP套接字编程](#基本SCTP套接字编程)
  - [接口模型](#接口模型)
    - [一到一形式](#一到一形式)
    - [一到多形式](#一到多形式)
  - [sctp_bindx函数](#sctp_bindx函数)
  - [sctp_connectx函数](#sctp_connectx函数)
  - [sctp_getpaddrs函数](#sctp_getpaddrs函数)
  - [sctp_freepaddrs函数](#sctp_freepaddrs函数)
  - [sctp_getladdrs函数](#sctp_getladdrs函数)
  - [sctp_freeladdrs函数](#sctp_freeladdrs函数)
  - [sctp_sendmsg函数](#sctp_sendmsg函数)
  - [sctp_recvmsg函数](#sctp_recvmsg函数)
  - [sctp_opt_info函数](#sctp_opt_info函数)
  - [sctp_peeloff函数](#sctp_peeloff函数)
  - [shutdown函数](#shutdown函数)
- [通知](#通知)
  - [对应事件预定字段](#对应事件预定字段)
    - [SCTP_ASSOC_CHANGE    关联更改事件](#SCTP_ASSOC_CHANGE关联更改事件)
    - [SCTP_PEER_ADDR_CHANGE  地址事件](#SCTP_PEER_ADDR_CHANGE地址事件)
    - [SCTP_REMOTE_ERROR   远程错误事件](#SCTP_REMOTE_ERROR远程错误事件)
    - [SCTP_SEND_FAILED  数据发送失败事件](#SCTP_SEND_FAILED数据发送失败事件)
    - [SCTP_SHUTDOWN_EVENT    关机事件](#SCTP_SHUTDOWN_EVENT关机事件)
    - [SCTP_ADAPTION_INDICATION   适应层指示参数](#SCTP_ADAPTION_INDICATION适应层指示参数)
- [一到多式流分回射服务器程序](#一到多式流分回射服务器程序)
- 











- ==**SCTP 不提供半关闭状态**==
  - **SCTP 应该使用 `shutdown` 发起终止序列来关闭 关联**
- 





# 安装和开启SCTP支持

- **ubuntu  开启命令:** 

  - `sudo apt-get install libsctp-dev lksctp-tools libsctp1`

- **树莓派开启命令(64位自制版,原版和 ubuntu 相同):**

  - `sid-used sudo apt-get install libsctp-dev lksctp-tools libsctp1`

- **Fedora、RedHat、CentOS开启命令:**

  - `sudo yum install kernel-modules-extra.x86_64 lksctp-tools.x86_64`

- **macOS 需要安装内核驱动模块(`失败了`):**

  - ==**首先下载在本目录中的 `macOs-sctp-支持` 目录下的 [SCTP_NKE_ElCapitan_Install_01.dmg](SCTP_NKE_ElCapitan_Install_01.dmg) 文件到本地, 并且双击加载.**==

  - ==先备份 `socket.h ` 文件, 因为后面的步骤会覆盖它==

    ```bash
    首先关闭mac前面验证功能,因为涉及到 内核驱动模块.
    	安装电源键启动后立即按住command+r 进入recover模式
    	打开终端控制台
    	执行命令:
    		csrutil enable  --without kext
    
    
    进行下面操作之前,必须保证已经安装Xcode ,并且也是使用Xcode 进行开发, 否则另寻到 /usr/目录进行替换
    
    
    sudo cp -R /Volumes/SCTP_NKE_ElCapitan_01/SCTPSupport.kext /Library/Extensions
    sudo cp -R /Volumes/SCTP_NKE_ElCapitan_01/SCTP.kext /Library/Extensions
    
    
    sudo cp /Volumes/SCTP_NKE_ElCapitan_01/socket.h /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include 
    
    sudo cp /Volumes/SCTP_NKE_ElCapitan_01/sctp.h /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/netinet
    
    sudo cp /Volumes/SCTP_NKE_ElCapitan_01/sctp_uio.h /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/netinet
    
    sudo cp /Volumes/SCTP_NKE_ElCapitan_01/libsctp.dylib /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/lib
    
    
    
    - 加载模块 
    	sudo kextload /Library/Extensions/SCTP.kext
    - 卸载模块
    	sudo kextunload /Library/Extensions/SCTP.kext
    ```

  - **加载模块** 

    - **`sudo kextload /Library/Extensions/SCTP.kext`**

  - **卸载模块**
  	
  	- **`sudo kextunload /Library/Extensions/SCTP.kext`**

# 基本SCTP套接字编程

**SCTP 是一个可靠的面向消息的协议,在端点之间提供多个流,并为多宿提供传输级支持.**

**SCTP 中的 *通知* ,使得一个应用进程能够知晓用户数据到达以外的重要协议事件.**





## 接口模型

- ==**SCTP 套接字分类**==
  - **一到一套接字**
    - 对应一个单独的 SCTP 关联. (SCTP的关联是 两个系统之间的一个连接)
    - 类型是 `SCOK_STREAM` , 协议为 `IPPROTO_SCTP` 网际网套接字,协议族为`AF_INET 或 AF_INET6`
  - **一到多套接字**
    - 一个给定套接字上可以同时有多个活跃的SCTP 关联.`(类似于绑定了端口的UDP套接字 正在接收其他很多的套接字发送的数据报)`



### 一到一形式

- ==目的是 方便将现有的TCP应用程序移植到 SCTP上==
  - **在移植时,必须要注意与 TCP 的差异:**
    - **任何TCP套接字选项必须转化成等效的SCTP套接字选项.`(例如: TCP_NODELAT和TCP_MAXSEG ,应该应设成 SCTP_NOELAY和SCTP_MAXSEG)`**
    - **TCP保存消息边界, 因此应用层消息并非必需.**
    - **TCP应用使用半关闭(写关闭) 来告知对方数据流已经结束. 这样移植到SCTP需要额外重新应用层协议,让进程在应用数据流中告知对端传出数据流已经结束**
    - **`send`函数能够以普通方式使用, 但是`sendto` 和 `sendmsg` 函数时, 指定的任何地址都被认为是对 目的地 主地址的重写.**
- ==**一到一 类型是 `SCOK_STREAM` , 协议为 `IPPROTO_SCTP` 网际网套接字,协议族为`AF_INET 或 AF_INET6`**==

<img src="assets/SCTP一到一形式的套接字函数.png" alt="SCTP一到一形式的套接字函数" style="zoom:50%;" />

### 一到多形式

- **一到多给开发人员提供了这样的能力:**
  - **编写的服务器程序无需管理大量的套接字描述符.**
- **单个套接字描述符将代表多个关联,就像一个UDP套接字能够从多个客户接收消息那样.**
- ==**在一到多式套接字上, 用于标识单个关联的是一个 关联标识 (association identifier)**==
  - **关联标识  是一个类型为 `sctp_assoc_t` 的值, 通常是一个整数. (对开发人员不透明)**
    - ==**应用程序必须使用由内核给予的关联标识符**==
- ==**一到多套接字的用户应该掌握以下几点:**==
  - ==**当一个客户关闭其关联时, 其服务器也将自动关闭同一个关联, 服务器主机内核中不再有该关联的状态.**==
  - ==***可用于致使在四路握手的第三个或第四个分组中捎带用户数据的唯一办法就是使用一到多形式***==
  - **客户端使用 `sento, sendmsg, SCTP_sendmsg` 连接服务器,如果成功则建立一个与该地址的新关联, 这个行为的发生与执行分组的发生与服务器的应用程序是否调用过 `list`函数 无关.**
  - ==**用户必须使用 `sendto, sendmsg , sctp_sendmsg`  三个分组发送函数, 不可以使用`write , send` 这两个分组发送函数.**==
    - **除非已经使用 `sctp_peeloff` 函数从一个一到多式 套接字剥离出一个  一到一式的套接字**
  - **调用发送分组函数时, 所用的目的地址是由系统在关联建立阶段选定的  主目的地址**
    - **除非调用者在所提供的 `sctp_sndrcvinfo` 结构中设置了 `MSG_ADDR_OVER` 标志, 为了提供这个结构,调用者必须使用伴随辅助数据的 `sendmsg`	 函数或 `sctp_sendmsg` 函数 .**
  - **关联事件 可能被启用, 因此要是应用进程不希望收到这些事件, 就得使用 `SCTP_EVENTS` 套接字选项显示禁用他们.**
    - **默认情况下只有 `sctp_data_io_event` 事件会启用, 它给 `recvmsg` 和`sctp_recvmsg` 调用提供辅助数据.( 适用一到多 和 一到一)**
- ==**客户端创建完套接字并且初次调用 `sctp_sendto`函数时, 将导致隐式建立关联, 而数据请求由四路握手的第三个分组捎带到服务器**==

<img src="assets/SCTP一到多形式的套接字函数.png" alt="SCTP一到多形式的套接字函数" style="zoom:50%;" />

- **一到多 套接字 能够结合使用 `sctp_peeloff` 函数允许组合迭代服务器模型和并发服务器模型:**
  - ==**`sctp_peeloff` 函数能够将 一到多套接字剥离出某个特定关联, 独自构成一个  一到一式套接字**==
  - **剥离出来的关联所在的一到一到接字 锁后就可以遣送给自己的字线程或派生子进程.**
  - **这个时候, 主线程继续在原来的套接字上以迭代的方式处理来自任何剩余关联的消息.**

- ==**一到多 类型是 `SCOK_SEQPACKET` , 协议为 `IPPROTO_SCTP` 网际网套接字,协议族为`AF_INET 或 AF_INET6`**==



### sctp_bindx函数

**SCTP服务器可以绑定与所在主机系统相关IP地址的一个子集.(就是随意挑选主机上的哪几个IP作为捆绑)**

==**所有绑定IP时使用的端口号必须是同一个**==

==**如果对监听套接字使用,那么将来产生的关联将使用新的地址配置,已经存在的关联则不受影响**==

```c
#include <netinet/sctp.h>
int sctp_bindx (int sockfd, const struct sockaddr* addrs, int addrcnt, int flags);
	参数:
		 sockfd :已创建的套接字(socket()), 绑定或未绑定的套接字都可以使用.
     addrs  :需要与套接字绑定的IP,端口,协议 的数组. (是结构体数组 struct sockaddr_in a[10])
               这些结构体的 端口号必须全部一致,不可以出现不同的端口号.
     addrcnt: addrs地址数组的个数,也就是有多少个地址 (也就是数组中元素的个数) (10)
     flags  : 添加或删除参数addrs 中的IP地址
                SCTP_BINDX_ADD_ADDR   往套接字中添加地址
                SCTP_BINDX_REM_ADDR   从套接字中删除地址
	返回值 : 成功返回0 . 出错返回-1 , flags 两种参数同时指定时,返回错误码EINVAL
          addrs 数组中有端口号不同时,会返回错误码 EINVAL
```



### sctp_connectx函数

**用于连接到一个多宿对端主机.**(多宿主机: 拥有多个IP地址的主机)

```c
#include <netinet/sctp>
int sctp_connectx (int sockfd, const struct sockaddr* addrs, int addrcnt,
                    sctp_assoc_t*  id);
参数: 
    sockfd :已创建的套接字(socket()), 绑定或未绑定的套接字都可以使用.
     addrs :一个或多个 服务器的 IP,端口,协议 结构体数组,(一个或多个结构体)
    addrcnt: addrs地址数组的个数,也就是有多少个地址 (也就是数组中元素的个数)
	      id : 传出参数,保存的是 关联ID
 返回值: 成功返回0 .  出错返回 -1
```



### sctp_getpaddrs函数

**`getpeername` 函数不是为支持多宿概念的传输协议设计的, 当用于 SCTP时,仅仅返回主目的的地址(也就是对方的公有接口,而私有接口是无法获得的)**

==**使用 `sctp_getpaddrs` 时,可以返回对端的所有地址**==

==**使用`sctp_getpaddrs` 会分配资源, 必须使用 `sctp_freepaddrs` 函数来释放分配的资源**==

```c
#include <netinet/sctp.h>
int sctp_getpaddrs (int sockfd, sctp_assoc_t id, struct sockaddr **addrs);

参数:
      sockfd :已关联的套接字
          id :指定要查询的 关联ID (connectx获得) , 仅用于一对多, 一对一时忽略该参数.
       addrs :传出参数. 该套接字关联的对端地址的列表数组. IP,端口,协议 .数组长度是返回值
返回值:
		  成功返回 addrs 参数所传出的数组的长度, 如果此套接字上没有关联, 则返回0
	    错误返回-1  并且 addrs的值是未定义的.(也就不需要再调用 sctp_freepaddrs 函数了)
```



### sctp_freepaddrs函数

**释放由`sctp_getpaddrs` 函数分配的资源**

```c
#include <netinet/sctp.h>
void sctp_freepaddrs (struct sockaddr* addrs);

参数:
      addrs: 是 sctp_getpaddrs 所返回的地址数组的指针.
```



### sctp_getladdrs函数

**用于获取 属于某个关联的本地地址(可能是主机所有地址的一个子集)**

**一个本地端点会使用某些本地的地址(可能是个子集)**

==**使用`sctp_getladdrs` 会分配资源, 必须使用 `sctp_freeladdrs` 函数来释放分配的资源**==

```c
#include <netinet/sctp.h>
int sctp_getladdrs (int sockfd, sctp_assoc_t id, struct sockaddr** addrs);

参数:
      sockfd :已关联的套接字
          id :指定要查询的 关联ID(connectx获得) , 仅用于一对多, 一对一时忽略该参数.
       addrs :传出参数. 该套接字关联的本地地址的列表数组. IP,端口,协议 .数组长度是返回值
返回值:
	   成功返回 addrs 参数所传出的数组的长度, 如果此套接字上没有关联, 则返回0
    错误返回-1  并且 addrs的值是未定义的.(也就不需要再调用 sctp_freeladdrs 函数了)
```



### sctp_freeladdrs函数

**释放由`sctp_getladdrs` 函数分配的资源**

```c
#include <netinet/sctp.h>
void sctp_freeladdrs (struct sockaddr* addrs);

参数:
      addrs: 是 sctp_getladdrs 所返回的地址数组的指针.
```



### sctp_sendmsg函数

**发送伴随 辅助数据的函数. 这是个辅助函数库的调用(可能是系统调用), 方便应用进程使用SCTP高级特性**

- 如果实现把 `sctp_sendmsg` 函数映射成 `sendmsg` 函数,那么 `sendmsg` 的flag参数应该被设置为0.

```c
#include <netinet/sctp.h>
ssize_t sctp_sendmsg (int sockfd,  const void* msg,  size_t msgsz, 
                       const struct sockaddr* to,  socklen_t tolen,
                       uint32_t  ppid,       uint32_t flags,    uint16_t stream,
                       uint32_t  timetolive, uint32_t context);
参数:
     sockfd :套接字
        msg :发送缓冲区
      msgsz :发送缓冲区的字节长度
         to :接收数据的对端的地址结构 (应该只是一个对端,并不是群发,传入参数)
      tolen :对端地址结构的长度,to 参数的长度.(传入参数)
       ppid :指定 将随数据块传递的 净荷协议标识符.
      flags :该参数的值将传递给 SCTP栈, 用以标识任何 SCTP选项:
                  SCTP_ABORT   启动中止性的关联终止过程(仅限一对多)
                  SCTP_ADDR_OVER    指定SCTP不顾主目的地址而改用给定的地址
                  SCTP_EOF    发送完本消息后 启动正常的关联终止过程(仅限一对多)
                  SCTP_UNORDERED    指定本消息使用无序的消息传递服务
     stream :指定一个 SCTP流号
 timetolive :发送消息延迟限制时间 (毫秒).如果超过期限没有发送出去 则该消息过期,也就不会发送出去.
                设为0时将不会过期.
    context :指定可能有的用户上下文.(就是个标识符)
                 (如果该消息发送出错,该值会通过 消息通知机制与未传递的消息一起传递回上层)
返回值:
	 成功则返回所写字节数, 出错返回 -1 
```



### sctp_recvmsg函数

**接收对端发送过来的数据(`sctp_sendmsg`)**

**获取对端的地址.**

==**获取已读入消息缓冲区中的伴随所接收消息的`sctp_sndrcvinfo` 结构,(如果要使用`sctp_data_io_event`那么需要开启套接字选项`SCTP_EVENTS`,并使用 `setsockopt`函数, 参数是`sctp_event_subscribe` 结构体).**==

- 如果实现把 `sctp_recvmsg` 函数映射成 `recvmsg` 函数,那么 `recvmsg` 的flag参数应该被设置为0.

```c
#include <netinet/sctp.n>
ssize_t sctp_recvmsg (int scokfd,  void* msg,  size_t  msgsz, 
                      struct sockaddr* from,   socklen_t*  fromlen,
                      struct sctp_sndrcvinfo*  sinfo,   int* msg_flags);
参数: 
		sockfd :套接字
       msg :接收缓冲区(sctp_sendmsg)
     msgsz :接收缓存区msg的字节长度
      from :发送消息者的信息结构,包括地址,端口,协议
   fromlen :发送消息这的信息结构的字节长度
     sinfo :通知被启用时,会有与消息相关的信息来填充这个结构, 传出参数,成员 .sinfo_stream 是流号
 msg_flags :存放 可能有的 消息标志(通知), 传出参数.
   
返回值:
   成功返回所读字节数, 出错返回 -1
```



### sctp_opt_info函数

**`sctp_opt_info` 函数是为了无法为SCTP使用`getsockopt` 函数的那些实现提供的.**

**获取套接字的 某个套接字选项( [套接字选项](套接字选项.md))**

```c
#include <netinet/sctp.h>
int sctp_opt_info (int sockfd,  sctp_assoc_t assoc_id,  int opt, void* arg, 
                   socklen_t* siz);
参数:
    sockfd :套接字描述符
  assoc_id :可能存在的关联标识
       opt :是SCTP的套接字选项.
       arg :传出参数 , 已获取的选项当前值会存入这里
       siz :是arg参数的长度.( 是传入 传出参数 )
返回值:
	成功返回0,  出错返回-1
```



### sctp_peeloff函数

- **从  `一到多式套接字` 中抽取一个关联, 构成单独 的一个  `一到一式套接字`( 返回新的套接字)**
- **需要调用者使用 `一到多套接字的套接字描述符` 和 `关联标识ID` 传递给该函数调用, 该调用会返回一个新的 套接字描述符.**
- ==**整体上接收短期请求, 偶尔需要长期会话的应用系统可以利用`sctp_peeloff`来抽取一个关联.**==

```c
#include <netinet/sctp.h>
int sctp_peeloff (int sockfd, sctp_assoc_t id);

参数:
     sockfd :必须是 一到多式 套接字描述符
         di :需要抽取的某个 关联标识ID
返回值:
    成功则返回一个新的套接字描述符,他与关联ID绑定 
    出错返回 -1 
```



### shutdown函数

**[套接字函数和信号处理以及多进程](套接字函数和信号处理以及多进程.md) 这个文件包含 `shutdown` 函数的说明, 适用SCTP套接字和 UDP 以及TCP.**

- ==**SCTP 不提供半关闭状态**==
  - **任何一端发起终止序列时, 两个端点都得把已排队的任何数据发送掉, 然后关闭关联.**
- **关联主动关闭的发起端 应该使用 `shutdown` ,而不是使用 `close`**
  - **因为同一个端点可用于连接到一个新的对端 端点.(也就说明不需要重新初始化了)**

<img src="assets/调用shutdown关闭一个SCTP关联.png" alt="调用shutdown关闭一个SCTP关联" style="zoom:50%;" />

​	

## 通知

> - ==通知是由 事件产生的, 通知是本地的==
> - 通过 **通知** 可以追踪相关 关联的状态.
> - ==**通知**  传递的是传输级事件==
>   - 包括: 网络状态变得, 关联启动, 远程操作错误,  消息不可抵达.
> - ==默认情况下 除`sctp_data_io_event` 以外的所有事件都是被禁止的.==



- `SCTP_EVENTS` 套接字可以预定8个事件.  其中7个事件产生称为 **通知的额外数据**
  - **通知本身可经由普通的套接字描述符获取**
    - _当产生他们的事件发生时, 这些通知内嵌在数据中加入到套接字描述符_
    - **在预定相应通知的前提下** 读取某个套接字时,用户数据和通知将在缓冲区交错出现.
      - ==**区分对端数据和事件产生的通知:**==
        - **使用`recvmsg` 或 `sctp_recvmsg` 函数,读取时, 返回的`msg_flags`参数将含有 `MSG_NOTIFICATION` 标志, 这个标志告知进程刚刚读入的是来自本地 SCTP栈 的一个通知,而不是来自远端的数据.**
- 通知 都采用  **标签-长度-值** 格式,  前8个字节给出通知的类型和总长度.`(标签对应类型)`
- ==通知的信息将被写入 `sctp_recvmsg`函数的 `sinfo` 参数中 ,类型是 `struct sctp_sndrcvinfo`.==



> ==**通知格式如下:**==
>
> ```c
> /* 通知事件,每种通知有各自的结构, 给出在传输中发生的响应事件的具体信息 */
> union sctp_notification{
> struct sctp_tlv   sn_header;         // 首部,可以获得很多信息,但并不是全部.
> struct sctp_assoc_change   sn_assoc_change;    
> struct sctp_paddr_change   sn_paddr_change;   
> struct sctp_remote_error   sn_remote_error;
> struct sctp_send_failed    sn_send_failed;
> struct sctp_shutdown_event sn_shutdown_event;
> struct sctp_adaption_event sn_adaption_event;
> struct sctp_pdapi_event    sn_pdapi_vevnt;
> };
> 
> /* 该结构体用于解释类型值, 以便译解出所处理的实际消息. 是8个字节的首部 */
> struct sctp_tlv{
> u_int16_t   sn_type;       // 类型,可以对应事件预定字段.用于确定下面的 对应事件预定字段
> u_int16_t   sn_flags;      // 标志
> u_int32_t   sn_length;     // 长度
> };
> 
> struct sctp_tlv.sn_type;  这个值可以有如下对应关系:
> sctp_tlv.sn_type = SCTP_ASOC_CHANGE;      /* 预定字段: sctp_association_event */
> sctp_tlv.sn_type = SCTP_PEER_ADDR_CHANGE;   /* 预定字段: sctp_address_event */
> sctp_tlv.sn_type = SCTP_REMOTE_ERROR;       /* 预定字段: sctp_peer_error_event */
> sctp_tlv.sn_type = SCTP_SEND_FAILED;      /* 预定字段: sctp_send_failure_event */
> sctp_tlv.sn_type = SCTP_SHUTDOWN_EVENT;     /* 预定字段: sctp_shutdown_event */
> sctp_tlv.sn_type = SCTP_ADAPTION_INDEICATION; 
>                                    // 预定字段: sctp_adaption_layer_event
> sctp_tlv.sn_type = SCTP_PARTIAL_DELIVERY_EVENT;  
>                                   // 预定字段: sctp_partial_delivery_event
> sctp_tlv.sn_type = SCTP_ASOC_CHANGE;  // 预定字段: sctp_association_event
> 
> 
> // 下面是预定字段,  为1时开启 通知事件
> struct sctp_event_subscribe {
> 	uint8_t sctp_data_io_event;
> 	uint8_t sctp_association_event;    
> 	uint8_t sctp_address_event;
> 	uint8_t sctp_send_failure_event;
> 	uint8_t sctp_peer_error_event;
> 	uint8_t sctp_shutdown_event;
> 	uint8_t sctp_partial_delivery_event;
> 	uint8_t sctp_adaptation_layer_event;
> 	uint8_t sctp_authentication_event;
> 	uint8_t sctp_sender_dry_event;
> 	uint8_t sctp_stream_reset_event;
> };
> 
> 
> struct sctp_sndrcvinfo {
> 	uint16_t sinfo_stream;        // 流号
> 	uint16_t sinfo_ssn;  
> 	uint16_t sinfo_flags;         // SCTP选项
> #if defined(__FreeBSD__) && __FreeBSD_version < 800000
> 	uint16_t sinfo_pr_policy;
> #endif
> 	uint32_t sinfo_ppid;         // 净荷长度
> 	uint32_t sinfo_context;
> 	uint32_t sinfo_timetolive;
> 	uint32_t sinfo_tsn;
> 	uint32_t sinfo_cumtsn;
> 	sctp_assoc_t sinfo_assoc_id;
> 	uint16_t sinfo_keynumber;
> 	uint16_t sinfo_keynumber_valid;
> 	uint8_t  __reserve_pad[SCTP_ALIGN_RESV_PAD];
> };
> ```

### 对应事件预定字段

#### SCTP_ASSOC_CHANGE关联更改事件

==**本通知告知 应用进程 关联本身发生的 变动事件**==

**本通知告知 应用进程 关联本身发生变动; 或者已开始一个新的关联, 或者已结束一个现有的关联.**

```c
通知格式如下:
struct sctp_assoc_change{
	uint16_t sac_type;       /*该值等于  SCTP_ASOC_CHANGE 表示是 本事件发生 */
	uint16_t sac_flags;
	uint32_t sac_length;     /* 上面这三个是 解释类型值 ,是通用格式*/
	uint16_t sac_state;      /* 给出关联上发生的事件类型. 写在下面了 */
	uint16_t sac_error;      /* 导致本关联变动的SCTP 协议储物起因代码 */
	uint16_t sac_outbound_streams;  /* 本端输出流的协定的流数目 */
	uint16_t sac_inbound_streams;   /* 本端输入流的协定的流数目 */
	sctp_assoc_t sac_assoc_id;      /* 存放本关联的其他信息(比如:对端的某个自定义错误导致的终止) */
	uint8_t sac_info[0];         /* 0数组, 可以提供越界访问. 可以当作普通数组来进行使用. */
};

成员所代表的含义
	sac_state :  给出关联上发生的事件类型, 取下面值之一:
     SCTP_COMM_UP     某个新的关联刚刚启动.
     SCTP_COMM_LOST   关联标识字段给出的关联已经关闭,(对端不可达,或对端中止性关闭)
     SCTP_RESTART     对端主机崩溃 并已重启,(应该验证每个方向流的数目,重启后会发生变动)
     SCTP_SHUTDOWN_COMP   本地端点激发的关联终止过程已结束(调用shutdown或SCTP_EOF) 可建立新的连接了
     SCTP_CANT_STR_ASSOC  对端 对于本端的关联建立尝试(INT消息) 未曾给出相应
```



#### SCTP_PEER_ADDR_CHANGE地址事件

==**本通知告知 对端的某个地址 经历了状态变动**==

- 失败性质变动  (目的地不对所发送的消息作出相应). 
- 恢复性质  (早先处于故障状态的某个目的地恢复正常)

```c
通知格式如下:
struct  sctp_paddr_change{
	uint16_t spc_type;           /* 该值要等于 SCTP_PEER_ADDR_CHANGE */
	uint16_t spc_flags;
	uint32_t spc_length;
	struct sockaddr_storage spc_aaddr;   /* 本事件所影响的对端地址 */
	uint32_t spc_state;      /* SCTP对端地址状态通知, 列表放在下面 */
	uint32_t spc_error;      /* 用于提供关于事件更详细信息的通知错误代码 */
	sctp_assoc_t spc_assoc_id;   /* 存放关联标识 */
};

对端地址状态通知列表
	spc_state:  会成为下面的值之一:
     SCTP_ADDR_ADDED      地址现已加入关联 (仅用于支持动态地址选项的SCTP实现)
     SCTP_ADDR_AVAILABLE  地址现已可达
     SCTP_ADDR_CONFIRMED  地址现已证实有效
     SCTP_ADDR_MADE_PRIM  地址现已成为主目的地址
     SCTP_ADDR_REMOVED    地址不再属于关联 (仅用于支持动态地址选项的SCTP实现)
     SCTP_ADDR_UNREACHABLE  地址不再可达,重新路由到候选地址  (仅用于支持动态地址选项的SCTP实现)
```



#### SCTP_REMOTE_ERROR远程错误事件

==**远程端点可能给本地端点发送一个操作性错误消息**==

- 这些消息可以指出  **当前关联的各种出错条件.**
- **开启本通知时, 整个错误块(error chunk) 将以内嵌格式传递给应用进程**

```c
通知格式如下:
struct sctp_remote_error {
	uint16_t sre_type;      /* 该值要等于 SCTP_REMOTE_ERROR */
	uint16_t sre_flags;
	uint32_t sre_length;
	uint16_t sre_error;     /* SCTP协议错误起因代码 */
	sctp_assoc_t sre_assoc_id;    /* 关联标识 */
	uint8_t sre_data[0];    /* 内嵌格式存放完整的错误 */
};
```



#### SCTP_SEND_FAILED数据发送失败事件

==**无法递送到对端的消息通过本通知送回用户,   马上还会有一个 关联故障通知 到达.(该事件不建议使用)**==

- **大多数情况下 一个消息不能被递送的唯一原因是  *==关联已经失效==.***
- **关联有效前提下  消息递送失败的唯一情况是  ==*使用了SCTP的部分可靠性扩展*==**

```c
通知格式如下:
struct sctp_send_failed {
	uint16_t ssf_type;      /* 该值要等于 SCTP_SEND_FAILED */
	uint16_t ssf_flags;     /* 有两个取值 SCTP_DATA_UNSENT , SCTP_DATA_SENT  说明在下面 */
	uint32_t ssf_length;
	uint32_t ssf_error;    /* 不为0时:存放一个特定于本通知的错误代码 */
	struct sctp_sndrcvinfo ssf_info;   /* 发送数据时传递给内核的信息(流数目,上下文等) */
	sctp_assoc_t ssf_assoc_id;   /* 关联标识 */
	uint8_t ssf_data[0];     /* 存放未能递送的消息本身 */
};


ssf_flags:  会成为下面的值之一:
   SCTP_ADTA_UNSENT  指示相应消息无法发送到对端(例如生命周期短),因此对端永远无法收到消息.
   SCTP_DATA_SENT    指示相应消息以及至少发送到对端一次,而对端一直没有确认.(可能对端无法给出确认)
```



#### SCTP_SHUTDOWN_EVENT关机事件

==**对端 发送一个SHUTDOWN块到本地端点时,本通知被传递给应用进程**==

**本通知告知应用进程在相关套接字上不再接受新的数据.**

> - 所有当前已排队的数据将被发送出去,  发送完毕后相应关联就被终止.

```c
通知格式如下:
struct sctp_shutdown_event {
	uint16_t sse_type;       /* SCTP_SHUTDOWN_EVENT */
	uint16_t sse_flags;
	uint32_t sse_length;
	sctp_assoc_t sse_assoc_id;  /* 正在关闭中,不再接受数据的那个关联的 关联标识 */
};
```



#### SCTP_ADAPTION_INDICATION适应层指示参数

==**适应层指示参数,  该参数在 INIT和INIT-ACK 中交换, 用于通知对端执行什么类型的应用适应行为.**==

(直接内存访问/直接数据放置)

```c
struct sctp_adaption_event {
	uint16_t sai_type;      /* SCTP_ADAPTION_INDICATION */
	uint16_t sai_flags;
	uint32_t sai_length;
	uint32_t sai_adaption_ind;   /* 对端在INIT或INIT-ACK消息中传递给本地主机的32位整数 */
	sctp_assoc_t sai_assoc_id;   /* 本适应层通知的关联标识 */
};
```



#### SCTP_PARTIAL_DELIVERY_EVENT

==**部分递送应用程序接口,  部分递送API事件**==

>  对端发送非常大的一个消息, 本地接收时需要使用 `recvmsg`或 `sctp_recvmsg` 来进行.
>
> 而且部分递送API需要想应用进程传递状态信息. 也就是 SCTP_PARTAL_DELIVERY_EVENT 通知.

```c
struct sctp_pdapi_event {
	uint16_t pdapi_type;     /* SCTP_PARTIAL_DELIVERY_EVENT */
	uint16_t pdapi_flags;
	uint32_t pdapi_length;
	uint32_t pdapi_indication;  /* 存放发生的事件 */
	uint16_t pdapi_stream;      /* 未知, 应该是流信息 */
	uint16_t pdapi_seq;         /* 未知 , 应该是流数目 */
	sctp_assoc_t pdapi_assoc_id;  /* 部分递送API事件发生的关联标识,该字段唯一有效值是
                                SCTP_PARTIAL_DELIVERY_ABORTED, 指出当前活跃的部分递送已被终止*/
};
```





# 一到多式流分回射服务器程序





