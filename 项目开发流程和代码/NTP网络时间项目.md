- [时间拓展知识](#时间拓展知识)
- [代码会用到的内容](#代码会用到的内容)
- 



- **CST是 处理器内部实现的**
- **测量时间和测量时间间隔是有区别的:**
  - 当前时间:  当前是几分几秒的时间点 , 这个是指针 T*    ,只可以进行相同类型的相减操作 ,不可以进行同类型的相加操作,    与时间间隔可以进行相加
  - 时间间隔:  单位是秒 ,  这个是类似 int 的整数值, 可以进行加减法.  从1979-1-1 00:00:00  来计算的微秒, 可以表示万年(8字节)
- **编程解决的问题:**
  - **如何测得时间差  `clock error`**
    - 如果通信时间差在10个毫秒左右, 那么就代表硬件不是那么可靠
  - 时间 t 取模 86400, 但是 C++有可能会对这个运算出错,  应该写成 取模 86399
- 命令查看已经同步的NTP服务器 命令 `$ntpq -pn`

- **测试配置:**
  -  健全性检查
    -  同一主机，时钟误差应接近于零
    -  两台主机，时钟误差应该是对称的
  -  无NTP，自由运行
  -  一个启用了 NTP，另一个自由运行
  -  两者都启用了 NTP，但同步到不同的 NTP 服务器
  -  两者都启用了 NTP，一个作为另一个的 NTP 服务器



## 时间拓展知识

- UTC(原子铯的跃迁) 以前叫做 GMT
  - UTC 原子时钟, 民用,    UTC = TAI + Leap润秒修正
  - GMT 地球自转
- 添加于 6 月 300 日 : 0:00Z 或 12 月 31 日 00:00:00，无论 工作日和时间 zonuq 5 小时
  - 并非所有对等点都同步，您可能会收到来自未来的消息
  - 时钟在闰秒停止，但每次你打电话 gettimeofday()，内核假装你的时间越来越长
  - 在用户代码、内核、NTP 客户端甚至 N TP 服务器中测试不佳
-  谷歌博客：时间、技术和闰秒谎言
- UNIX time 是个整数,  UNIX元年是 1970-1-1 00:00:00 ,  美国时间是  1970-1-1 12:00:00





## 代码会用到的内容

```c++
struct Message  // 存储时间 , 客户端填写 request,  服务器填写 response
{
  int64_t request;
  int64_t response;
} __attribute__ ((__packed__));


// 获得当前时间
int64_t now()
{
  struct timeval tv = { 0, 0 };
  gettimeofday(&tv, NULL);
  return tv.tv_sec * int64_t(1000000) + tv.tv_usec;
}


//发送和接收数据 , 服务器
ssize_t nr = ::recvfrom(sock.fd(), &message, sizeof message, 0, addr, &addrLen);
    if (nr == sizeof message)
    {
      message.response = now();
      ssize_t nw = ::sendto(sock.fd(), &message, sizeof message, 0, addr, addrLen);
      if (nw < 0)
      {
        perror("send Message");
      }
      else if (nw != sizeof message)
      {
        printf("sent message of %zd bytes, expect %zd bytes.\n", nw, sizeof message);
      }
    }


// 客户端代码, 采用多线程发送和多线程接收
std::thread thr([&sock] () {
    while (true)
    {
      Message message = { 0, 0 };
      message.request = now();
      int nw = sock.send(&message, sizeof message);
      if (nw < 0)
      {
        perror("send Message");
      }
      else if (nw != sizeof message)
      {
        printf("sent message of %d bytes, expect %zd bytes.\n", nw, sizeof message);
      }

      ::usleep(200*1000);
    }
  });
```

