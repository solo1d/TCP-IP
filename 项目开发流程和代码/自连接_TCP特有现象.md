

仅仅在 Linux下有效

```c++
#include <iostream>
#include <thread>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <net/if.h>
#include <sys/types.h>
#include <arpa/inet.h>

int main(int argc, const char * argv[]) {
    int sockfd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    sockaddr_in   addr;
    socklen_t     addrLen = sizeof(addr);
    int ret = -1;
    ret = inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);
    if( ret != -1){
        printf("inet_pton ,yes\n");
    }
    else{
        printf ("inet_pton. 失败\n");
    }
    
    addr.sin_port = htons(50100);
    addr.sin_family = AF_INET;
    
    ret = bind(sockfd,  (sockaddr*)&addr , addrLen);
    if( ret != -1){
        printf("bind, yes\n");
    }
    else{
        printf ("bind, 失败\n");
    }
    ret =  connect(sockfd,  (sockaddr*)&addr , addrLen);
    if( ret != -1){
        printf("connect, yes\n");
        //  这里就造成了自连接,   应该检测和关闭当前的自连接
        //  检测的就是  客户端地址和 服务器地址 是否相同 即可
    }
    else{
        printf ("connect, 失败\n");
    }
    return 0;
}	
```







