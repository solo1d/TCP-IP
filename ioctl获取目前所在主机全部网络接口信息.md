- [ioctl操作](#ioctl操作)
  - [用于ioctl请求的ifconf和ifreq结构](#用于ioctl请求的ifconf和ifreq结构)
  - [获得系统中的所有接口](#获得系统中的所有接口)
  - 







- ==**ioctl操作 可以获取主机全部网络接口的信息, 包括:**==
  - **接口地址,  是否支持广播, 多播  等等**



## ioctl操作

```c
#include <unistd.h>
#include <sys/ioctl.h>
#include <sys/sockio.h>
#include <sys/filio.h>
#include <net/if_arp.h>
#include <sys/ioccom.h>

int  ioctl(int fd, unsigned  long request, ... /* void* arg */ );
/*
参数 :       fd : 已经打开的文件描述符
      resquest : 网络相关请求, 分为6类 , 具体内容在下面, 有表格
                     套接字操作
                     文件操作
                     接口操作
                     ARP高速缓存操作
                     路由表操作
                     流系统
                   都需要先划分缓冲区.
       参数... : 根据 resquset 的不同, 这里的数据类型也不同, 下面有表格, 都是指针
返回值:  成功返回0, 出错则为-1 并且 errno设置为指示错误。
          			EBADF   fd不是有效的描述符
                EINVAL  请求或argp无效
                ENOTTY  fd没有与特殊字符设备关联
                ENOTTY  指定的请求不适用于描述符过滤的对象类型参考
*/
```



|                类别 |  request 参数   | 说明                             | ...参数指针  数据类型 |
| ------------------: | :-------------: | :------------------------------- | :-------------------: |
|      **套接字操作** | **SIOCATMARK**  | 是否位于外带标记(1有,0无)        |          int          |
|      <sys/sockio.h> |  **SIOCSPGRP**  | 设置套接字的进程ID或进程组ID     |          int          |
|                     |  **SIOCGPGRP**  | 获取套接字的进程ID或进程组ID     |          int          |
|        **文件操作** |   **FIONBIO**   | 设置或清除 非阻塞式I/O标志       |          int          |
|       <sys/filio.h> |  **FIOASYNC**   | 设置或清除 信号驱动异步I/O标志   |          int          |
|                     |  **FIONREAD**   | 获取接收缓冲区中的字节数         |          int          |
|                     |  **FIOSETOWN**  | 设置打开文件的进程ID或进程组ID   |          int          |
|                     |  **FIOGETOWN**  | 获取打开文件的进程ID或进程组ID   |          int          |
|        **接口操作** | **SIOCGIFCONF** | **获取所有接口的列表**<net/if.h> |     struct ifconf     |
|      <sys/sockio.h> |   SIOCSIFADDR   | 设置接口地址                     |     struct ifreq      |
|                     |   SIOCGIFADDR   | 获取接口地址                     |     struct ifreq      |
|                     |  SIOCSIFFLAGS   | 设置接口标志                     |     struct ifreq      |
|                     |  SIOCGIFFLAGS   | 获取接口标志                     |     struct ifreq      |
|                     | SIOCSIFDSTADDR  | 设置点到点地址                   |     struct ifreq      |
|                     | SIOCGIFDSTADDR  | 获取点到点地址                   |     struct ifreq      |
|                     | SIOCGIFBRDADDR  | 获取广播地址                     |     struct ifreq      |
|                     | SIOCSIFBRDADDR  | 设置广播地址                     |     struct ifreq      |
|                     | SIOCGIFNETMASK  | 获取子网掩码                     |     struct ifreq      |
|                     | SIOCSIFNETMASK  | 设置子网掩码                     |     struct ifreq      |
|                     |  SIOCGIFMETRIC  | 获取接口的测度                   |     struct ifreq      |
|                     |  SIOCSIFMETRIC  | 设置接口的测度                   |     struct ifreq      |
|                     |   SIOCGIFMTU    | 获取接口MTU                      |     struct ifreq      |
|                     |     SIOCxxx     | (还有很多,取决于实现)            |     struct ifreq      |
| **ARP高速缓存操作** |  **SIOCSARP**   | 创建或修改ARP表项 sa_data        |     struct arpreq     |
|      <net/if_arp.h> |    SIOCGARP     | 获取ARP表项                      |     struct arpreq     |
|                     |    SIOCDARP     | 删除ARP表项                      |     struct arpreq     |
|      **路由表操作** |    SIOCADDRT    | 增加路径                         |    struct rtentry     |
|                     |    SIOCDELRT    | 删除路径                         |    struct rtentry     |
|          **流系统** |      I_xxx      |                                  |                       |



## 用于ioctl请求的ifconf和ifreq结构

```c
// 下面的内容来自于 <net/if.h>  头文件

struct  ifconf {
	int     ifc_len;                /* 关联缓冲区的大小 */
	union {
		caddr_t ifcu_buf;
		struct  ifreq *ifcu_req;
	} ifc_ifcu;
};

#define ifc_buf ifc_ifcu.ifcu_buf       /* 缓冲区地址 */
#define ifc_req ifc_ifcu.ifcu_req       /* 返回的结构数组 */

#define IF_NAMESIZE     16
#define IFNAMSIZ        IF_NAMESIZE

struct  ifreq {
	char    ifr_name[IFNAMSIZ];             /* 如果 名称，例如“ en0” */
	union {      // 下面都是联合体
		struct  sockaddr ifru_addr;
		struct  sockaddr ifru_dstaddr;
		struct  sockaddr ifru_broadaddr;
		short   ifru_flags;
		int     ifru_metric;
		int     ifru_mtu;
		int     ifru_phys;
		int     ifru_media;
		int     ifru_intval;
		caddr_t ifru_data;
		struct  ifdevmtu ifru_devmtu;
		struct  ifkpi   ifru_kpi;
		u_int32_t ifru_wake_flags;
		u_int32_t ifru_route_refcnt;
		int     ifru_cap[2];
		u_int32_t ifru_functional_type;
	} ifr_ifru;
};



#define ifr_addr        ifr_ifru.ifru_addr      /* 地址 */
#define ifr_dstaddr     ifr_ifru.ifru_dstaddr   /* 点对点链接的另一端 */
#define ifr_broadaddr   ifr_ifru.ifru_broadaddr /* 广播地址 */
#define ifr_flags       ifr_ifru.ifru_flags     /* 标志 */
#define ifr_metric      ifr_ifru.ifru_metric    /* 公制 */
#define ifr_mtu         ifr_ifru.ifru_mtu       /* mtu */
#define ifr_phys        ifr_ifru.ifru_phys      /* 网络线 */
#define ifr_media       ifr_ifru.ifru_media     /* 实体设备介质 */
#define ifr_data        ifr_ifru.ifru_data      /* 用于接口 */
#define ifr_devmtu      ifr_ifru.ifru_devmtu
#define ifr_intval      ifr_ifru.ifru_intval    /* 整数值 */
#define ifr_kpi         ifr_ifru.ifru_kpi
#define ifr_wake_flags  ifr_ifru.ifru_wake_flags /* 网络唤醒功能 */
#define ifr_route_refcnt ifr_ifru.ifru_route_refcnt /* 路线参考计数 */
#define ifr_reqcap      ifr_ifru.ifru_cap[0]    /* 要求的功能 */
#define ifr_curcap      ifr_ifru.ifru_cap[1]    /* 目前的能力 */
  
  
#define IFRTYPE_FUNCTIONAL_UNKNOWN              0
#define IFRTYPE_FUNCTIONAL_LOOPBACK             1
#define IFRTYPE_FUNCTIONAL_WIRED                2
#define IFRTYPE_FUNCTIONAL_WIFI_INFRA           3
#define IFRTYPE_FUNCTIONAL_WIFI_AWDL            4
#define IFRTYPE_FUNCTIONAL_CELLULAR             5
#define IFRTYPE_FUNCTIONAL_INTCOPROC            6
#define IFRTYPE_FUNCTIONAL_COMPANIONLINK        7
#define IFRTYPE_FUNCTIONAL_LAST                 7


struct arpreq {
	struct  sockaddr arp_pa;                /* protocol address */
	struct  sockaddr arp_ha;                /* hardware address */
	int     arp_flags;                      /* flags */
};
#define ATF_INUSE       0x01    /* entry in use */
#define ATF_COM         0x02    /* completed entry (enaddr valid) */
#define ATF_PERM        0x04    /* permanent entry */
#define ATF_PUBL        0x08    /* publish entry (respond for other host) */
#define ATF_USETRAILERS 0x10    /* has requested trailers */

```





## 获得系统中的所有接口

```c++
//  这些代码在 macos11.3 bate 版本中, 无法得到正常的运行结果
//  Created by lq on 2021/4/9.
//

#ifdef  __APPLE__
    #include <malloc/malloc.h>
    #include <sys/event.h>
#endif

#include <iostream>
#include <unistd.h>
#include <sys/types.h>
#include <cerrno>

#include <csignal>

#include <ctime>
#include <cmath>
#include <string>
#include <map>

#include <netinet/in.h>
#include <net/if.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <poll.h>
#include <fcntl.h>
#include <netdb.h>
#include <sys/ioctl.h>

#include <sys/uio.h>
#include <sys/un.h>
#include <sys/wait.h>
#include <sys/select.h>

#include <sys/stat.h>
#include <sys/sysctl.h>



using namespace std;

int main()
{
    //得到套接字描述符
    int sockfd;
    if ((sockfd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
    {
        perror("socket error");
        exit(-1);
    }

    struct ifconf ifc;
    caddr_t buf;

    //初始化ifconf结构
    ifc.ifc_len = 2048;
    if ((buf = (caddr_t)malloc(2048)) == NULL)
    {
        cout << "malloc error" << endl;
        exit(-1);
    }
    ifc.ifc_buf = buf;

    //获取所有接口信息
    if (ioctl(sockfd, SIOCGIFCONF, &ifc) < 0)
    {
        perror("ioctl error");
        exit(-1);
    }

    //遍历每一个ifreq结构
    struct ifreq *ifr;
    struct ifreq ifrcopy;
    ifr = (struct ifreq*)buf;
    char  ip[INET_ADDRSTRLEN];
    memset(ip, 0, INET_ADDRSTRLEN);
    for(int i = (ifc.ifc_len/sizeof(struct ifreq)); i>0; i--)
    {
        //接口名
        cout << "interface name: "<< ifr->ifr_name << endl;
        //ipv4地址
        cout << "inet addr: "
             << inet_ntoa(((struct sockaddr_in*)&(ifr->ifr_addr))->sin_addr)
             << endl;

        //获取广播地址
        ifrcopy = *ifr;
        if (ioctl(sockfd, SIOCGIFBRDADDR, &ifrcopy) < 0)
        {
            perror("ioctl error");
            std::cerr << "无广播地址" << endl;
            //exit(-1);
        }
        cout << "broad addr: " << inet_ntop(AF_INET, &((struct sockaddr_in*)&(ifrcopy.ifr_addr))->sin_addr, ip, INET_ADDRSTRLEN)
             << endl;
        //获取mtu
        ifrcopy = *ifr;
        if (ioctl(sockfd, SIOCGIFMTU, &ifrcopy) < 0)
        {
            perror("ioctl error");
            // exit(-1);
        }
        cout << "mtu: " << ifrcopy.ifr_mtu << endl;
        cout << endl;
        ifr++;
    }

    return 0;
}
```







