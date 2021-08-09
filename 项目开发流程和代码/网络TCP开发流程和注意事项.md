写代码要注意的流程

- [网络具体分层叫法](#网络具体分层叫法)
- [所有要注意的事项](#所有要注意的事项)
- [简单测试收发数据的命令](#简单测试收发数据的命令)
- [网络包分布](#网络包分布)
- [服务器](#服务器)
- [客户端](#客户端)



**安装依赖 `$sudo apt install -y libboost-dev`**



## 网络具体分层叫法

- **Ethernet frame    , 以太网层  称为  帧**
- **IP packet   , IP网络层  称为 分组**
- **TCP  segment ,   TCP 传输层, 称为  分节**
- **Application  message  , 应用层  一般说 消息**



# 所有要注意的事项

- **尽量不要让和socketAPI打交道的代码  与业务逻辑混在一起**
- **TCP断开的时机与条件,  `close()` 太早的话, 会导致协议栈发送 `RSP` 分节, 将连接重置, 导致数据丢失,无法全部收完**
- **套接字选项需要设置:**
  - **关闭 Nagle算法, 可降低 小分组数据的网络延迟, 不会去一直等待ACK 之后再发送数据,  `setsockopt(sockfd, IPPROTO_TCP,TCP_NODELAY,(int)1,sizeof(int) );`**
  - **将发送缓冲区低水位标记设置为1或者其他合理的值 `setsockopt(sockfd, SOL_SOCKET, SO_SNDLOWAT,(int)1, sizeof(int));`**
  - **设置监听套接字地址复用: `setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR,(int)1,sizeof(int))`**
- **设置信号 `SIGPIPE` 的捕获和处理**
- **注意网络字节序**
- **使用阻塞IO发送数据时, 要注意 发送和接收的内核缓冲区长度, 避免已满而无法接收或发送新数据的情况, 会造成死阻塞 `$sysctl -A | grep tcp.*mem   #可查看内核分配给的套接字缓冲区长度,默认67108864字节 `  **
  - MacOS 使用如下命令 `sysctl -A | grep "tcp.*buf:"`   默认给 8MiB
  - **而且阻塞IO 会自动限制发送的速率**
- 查看可以供用户使用的端口号命令 `$sysctl -A |grep range`
- **小心自连接**
- **并发模型:**
  - 多线程与阻塞 I/O 配合使用
  - I/O复用与非阻塞I/O  配合使用  ,  IO复用就是一个进程同时处理多个文件描述符 (同步的)
    - 注意在 select中的 写事件, 并且在发送一次数据之后就清除写状态
    - 注意双方网络速率不对等时, 会将写缓冲区写满, 丢失数据





## 简单测试收发数据的命令

```bash
服务端 , 监听 5001 端口, 将收到的数据写到 /dev/null, 后面 pv -W 表示显示网络速率 , 
$ nc -l 5001 | pv -W  > /dev/null
# 显示: 1000MiB 0:00:04 [ 59.2MiB/s] ,   1MiB = 1024Kib = 1048576 Byte   1024进制
# nc 从内核tcp 读数据 , 再写入到内核管道,  pv从内核管道读取数据, 再重定向到 内核null, 4次进入内核态


客户端 , dd 出 1GB 数据,  发送给 服务器. 
$ dd if=/dev/zero bs=1m count=1000  | nc pi.local 5001
# 当前为 前兆网络.  数据为
#1000+0 records in
#1000+0 records out
#1048576000 bytes transferred in 18.233581 secs (57507957 bytes/sec)
#换算: 1GB(1000*1024*1024=1048576000Bytes)
#      传输速率 每秒 : 57507957 bytes/sec  =   54.843861579895 MB/s      1000进制
#    用时 18秒

# dd 从内核读数据, 写入到内核管道, nc 从内核管道读出来, 写入	内核tcp栈 , 4次进入内核态

第二种方法, 需要使用 /tmp 的文件, 毕竟在内存里 , 能到 126MiB/s
$ nc pi.local 5001 < /tmp/file 



K单独出现时，代表1000 或 1024
K与Ki一起出现时，K代表1000，Ki代表1024
K与k一起出现时，K代表1024，k代表1000
```



# 网络包分布

<img src="assets/网络包分布.png" alt="网络包分布.png" style="zoom:66%;" />

# 服务器

- 信号处理
  - `sigaction`
- 创建套接字 `socket`
  - 设置套接字属性
    - `setsockopt`,  `getsockopt`
- 初始化绑定的本地地址
  - `bind`,  `list`, `inet_pton`,  `htos` ,`hotel(INADDR_ANY)`
- 监听套接字
  - `accept`
- 处理连接过来的套接字
  - 收发数据
    - `read, recv, recvfrom, recvmsg`  , `write, send, sendto, sendmsg`
  - 处理问题:
    - 固长数据
    - 变长数据
    - 粘包  -> 拆包
    - 少包  -> 组包
- 关闭套接字
  - `shutdown` , `close`





# 客户端

- 信号处理
  - `sigaction`
- 创建套接字 `socket`
  - 设置套接字属性
    - `setsockopt`,  `getsockopt`
- 初始化绑定的本地地址
  - `bind`,  `list`, `inet_pton`,  `htos` ,`hotel(INADDR_ANY)`
- 连接服务器
  - `connect`
- 收发数据
  - `read, recv, recvfrom, recvmsg`  , `write, send, sendto, sendmsg`
- 关闭套接字
  - `shutdown` , `close`











