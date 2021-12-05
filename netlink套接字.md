- [创建netlink套接字](#创建netlink套接字)
- [netlink套接字地址结构](#netlink套接字地址结构)
- [代码范例](#代码范例)
- 



应用于 3.9内核源码

- netlink套接字优势:
  - 该套接字不需要轮询
  - 内核可以自动向用户空间发送异步消息,而不需要用户空间来触发
  - 支持组播传输
- **netlink 专用于 联网的 Netlink套接字, 路由消息, 邻接消息, 链路消息和其他网络子系统消息**
- 



## 创建netlink套接字

用户和内核都是调用 `__netlink_create()` 来创建的

```c
#include <linux/netlink.h>
创建用户级netlink套接字:
	int	fd = socket(AF_NETLINK, SOCK_RAW | SOCK_CLOEXEC, NETLINK_ROUTE);
    // 更具体的 可以去  /include/uapi/linux/netlink.h 中去查看
创建内核级netlink套接字:
	netlink_kernel_create();
```



## netlink套接字地址结构

```c
#include <linux/netlink.h>
struct sockaddr_nl {
	__kernel_sa_family_t	nl_family;		/* AF_NETLINK */
  unsigned short				nl_pad;				/* 为0  */
  __u32									nl_pid;       /* 端口号, 内核套接字为0, 用户应为当前PID, 且保持唯一性 */
  __u32									nl_groups;		/* 组播组掩码 */
}
```



## 代码范例

内核模块代码

```c
#ifndef __KERNEL__
#define __KERNEL__
#endif  /* __KERNEL__ */
  
#include <linux/module.h>
#include <linux/init.h>
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/netdevice.h>
#include <linux/uaccess.h>
#include <net/sock.h>
#include <net/netlink.h>  // netlink
#include <linux/string.h>
#include <linux/ip.h>
  
#define NETLINK_TEST 31    // 自定义用户协议
  
static struct sock *nlfd = NULL;  // 内核socket文件描述符
static char *payload = "Hello user,i'm from kernel!";
  
// 向用户层发送消息
static int sendNLMsg(int pid,void *msg,int len)
{
    struct sk_buff *skb;
    int size,count;
    struct nlmsghdr *nlmsgh = NULL;
    char *pos = NULL;
    int retval = 0;
  
    size = NLMSG_SPACE(len);               // 加消息头部长度
    skb = alloc_skb(size,GFP_KERNEL);   // 申请空间
    if(!skb)
    {
        retval = -1;
        return retval;
    }
  
    nlmsgh = nlmsg_put(skb,0,0,0,len,0);
  
    // 填充数据
    pos = NLMSG_DATA(nlmsgh);
    memset(pos,0,len);
    memcpy(pos,msg,len);
  
    NETLINK_CB(skb).dst_group = 0;  // 单播
  
    count = netlink_unicast(nlfd,skb,pid,MSG_DONTWAIT);
    printk(KERN_ALERT "pid:%d send:%d\n",pid,count);
  
    return retval;
}
  
  
// 处理用户层传递的消息
static void handle_msg(struct sk_buff *_sk)
{
    struct sk_buff *skb;
    struct nlmsghdr *nlh = NULL;
    char str[100] = {};
    int pid;
  
    printk("==>handle_msg\n");
      
    skb = skb_get(_sk);   // 引用当前_sk
  
    if(skb->len >= NLMSG_SPACE(0))
    {
        nlh = nlmsg_hdr(skb);         // 获得信息头部
        pid = nlh->nlmsg_pid;         // 获取用户进程pid
  
        memcpy(str,NLMSG_DATA(nlh),100);
        printk(KERN_ERR "recv:%s\n",str);
  
        sendNLMsg(pid,payload,strlen(payload));  // 向用户层发送数据
  
        kfree_skb(skb);
    }
  
    printk("<==handle_msg\n");
}
  
// 建立netlink套接字
int NLCreate(void)
{
        // 消息回调函数为handle_msg，注：参数1区别以往内核版本API，设为init_net
    nlfd = netlink_kernel_create(&init_net,NETLINK_TEST,1,handle_msg,NULL,THIS_MODULE);
    if(!nlfd)
    {
        return -1;
    }
  
    return 0;
}
  
// 清除netlink套接字
int NLDestroy(void)
{
    if(nlfd)
    {
        sock_release(nlfd->sk_socket);
    }
  
    return 0;
}
  
  
static int __init netlink_init(void)
{
    NLCreate();
  
    return 0;
}
  
  
static void __exit netlink_exit(void)
{
    NLDestroy();
}
  
module_init(netlink_init);
module_exit(netlink_exit);
  
MODULE_LICENSE("GPL");
MODULE_AUTHOR("kettas");
MODULE_DESCRIPTION("Netlink Test Demo");
MODULE_VERSION("1.0.1");
MODULE_ALIAS("Netlink 01");
```



用户空间代码

```c
#include <unistd.h>
#include <stdio.h>
#include <linux/types.h>
#include <sys/socket.h>
#include <string.h>
#include <linux/netlink.h>
#include <assert.h>
#include <stdlib.h>
  
  
#define NETLINK_TEST 31
#define MAX_NL_MSG_LEN 1024
  
typedef struct _packet_u
{
    struct nlmsghdr hdr;
    char payload[1024];
}packet_u;
  
  
static int nls;  // socket文件描述符
  
// 向内核发送消息
int sendtokernel(char *buf,int len,int type)
{
    struct nlmsghdr nlmsg;
    struct sockaddr_nl nldest = {};
    int size;
  
    nldest.nl_family    = AF_NETLINK;
    nldest.nl_pid       = 0;
    nldest.nl_groups    = 0;
  
    // 填充netlink消息头
    nlmsg.nlmsg_len = NLMSG_LENGTH(len);
    nlmsg.nlmsg_pid = getpid();
    nlmsg.nlmsg_flags = 0;
    nlmsg.nlmsg_type  = type;
  
    // 填充负载
    memcpy(NLMSG_DATA(&nlmsg),buf,len);
  
    // 发送
    size = sendto(nls,&nlmsg,nlmsg.nlmsg_len,0,(struct sockaddr*)&nldest,sizeof(nldest));
    return size;
}
  
  
// 接收内核消息
int recvfromkernel(void)
{
    int size = 0;
  
        // 方法一：调用recvfrom方法接收内核数据，注此时message结构体包含有消息体
        /*
    packet_u message;
    struct sockaddr_nl nldest = {};
    int len = sizeof(nldest);
     
    memset(&message,0,sizeof(message));
 
    nldest.nl_family    = AF_NETLINK;
    nldest.nl_pid       = 0;
    nldest.nl_groups    = 0;
     
    // 接收消息
    size = recvfrom(nls,
            &message,
            sizeof(message),
            0,
            (struct sockaddr*)&nldest,
            &len);
    printf("size:%d recv:%s\n",size,message.payload);   // NLMSG_DATA()与message.payload结果一致
        */
    // 方法二：调用recvmsg方法
    struct sockaddr_nl nladdr;
    struct msghdr msg;
    struct iovec iov;
    struct nlmsghdr *nlhdr;
  
    nlhdr = (struct nlmsghdr *)malloc(MAX_NL_MSG_LEN);
    iov.iov_base = (void *)nlhdr;
    iov.iov_len = MAX_NL_MSG_LEN;
    msg.msg_name = (void *)&(nladdr);
    msg.msg_namelen = sizeof(nladdr);
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;
    size = recvmsg(nls, &msg, 0);
    printf("size:%d recv:%s\n",size,(char*)NLMSG_DATA(nlhdr));
  
    return size;
}
  
int main(int argc,char **argv)
{
    struct sockaddr_nl nlsource;
    int ret;
  
    // socket
    nls = socket(PF_NETLINK,SOCK_RAW,NETLINK_TEST);
    assert(nls!=-1);
  
    memset(&nlsource,0,sizeof(struct sockaddr_nl));
    nlsource.nl_family = AF_NETLINK;
    nlsource.nl_pid    = getpid();
    nlsource.nl_groups = 0;
  
    // bind
    ret = bind(nls,(struct sockaddr*)&nlsource,sizeof(nlsource));
    assert(ret!=-1);
  
    // send
    char *str = "Hello kernel,i'm from user";
    sendtokernel(str,strlen(str),0);
  
    // recv
    recvfromkernel();
  
    close(nls);
    return 0;
}  

```



测速运行

```bash
[scada@linux netlink]$ sudo insmod netlink_test.ko
[scada@linux netlink]$ ./netlink_u
size:44 recv:Hello user,i'm from kernel!
[scada@linux netlink]$ dmesg
==>handle_msg
recv:Hello kernel,i'm from user
pid:40122 send:44
<==handle_msg  
```

