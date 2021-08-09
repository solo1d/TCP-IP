- 正确的使用TCP
- [正确的关闭TCP连接](#正确的关闭TCP连接)





- 与文件描述符打交道
- **并发模型:**
  - 多线程与阻塞 I/O 配合使用
  - I/O复用与非阻塞I/O  配合使用  ,  IO复用就是一个进程同时处理多个文件描述符 (同步的)
    - 注意在 select中的 写事件, 并且在发送一次数据之后就清除写状态
    - 注意双方网络速率不对等时, 会将写缓冲区写满, 丢失数据
- **TCP关闭是个难点**
- **套接字选项需要设置:**
  - **关闭 Nagle算法, 可降低 小分组数据的网络延迟, 不会去一直等待ACK 之后再发送数据,  `setsockopt(sockfd, IPPROTO_TCP,TCP_NODELAY,(int)1,sizeof(int) );`**
  - **将发送缓冲区低水位标记设置为1或者其他合理的值 `setsockopt(sockfd, SOL_SOCKET, SO_SNDLOWAT,(int)1, sizeof(int));`**
  - **设置监听套接字地址复用: `setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR,(int)1,sizeof(int))`**
- **设置信号 `SIGPIPE` 的捕获和处理**





## 正确的关闭TCP连接

- **正确关闭的流程:**
  - 进入半关闭状态  `shutdown(sockfd, SHUT_WR)`
  - 然后再读数据, 清空内核套接字的接收缓冲区 , 进行丢弃
  - 如果不清空套接字接收缓冲区, 那么会导致发送一个 RST 分节给对方, 造成数据接收不完整
- **当客户端关闭了套接字, 然后服务器再通过该套接字相客户端发送消息 就会返回 `SIGPIPE` 错误信号**
  - **处理方式: 使用全局的对象, 里面封装忽略 `SIGPIPE` 信号的构造代码来处理. 但是也要注意各种函数的返回值**

- **服务器:**
  - send()  + shutdown(WR)  + {read() =0}  + close()
    - shutdown(WR) 调用后,  对方的 read() 会返回0
    - read() 也要有定时器存在, 防止恶意连接 导致无法从read() 返回, 但是最好添加头部长度消息, 免得循环read
- **客户端:**
  - {read() =0 } + 判断 if( send () +close() )









