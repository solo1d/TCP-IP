1.使用netstat命令，用来显示各种网络信息，比如网络连接、端口号、协议、状态、进程ID等信息；

   使用lsof命令，用来显示当前系统打开文件的信息，因为包括端口和网络状态在linux和mac中都属于文件，所以系统也为它们分配了文件描述法fd。

2.在Linux上面使用的方法通常为：netstat -nltp，参数含义为：查询TCP协议写Listen的信息

    -a (all)显示所有选项，默认不显示LISTEN相关
   -t (tcp)仅显示tcp相关选项
   -u (udp)仅显示udp相关选项
   -n 拒绝显示别名，能显示数字的全部转化成数字。
   -l 仅列出有在 Listen (监听) 的服務状态

   -p 显示建立相关链接的程序名
   -r 显示路由信息，路由表
   -e 显示扩展信息，例如uid等
   -s 按各个协议进行统计
   -c 每隔一个固定时间，执行该netstat命令

3.但是在Mac上执行该命令，会报如下的错误：

   netstat: option requires an argument -- p
   Usage:  netstat [-AaLlnW] [-f address_family | -p protocol]
   netstat [-gilns] [-f address_family]
   netstat -i | -I interface [-w wait] [-abdgRtS]
   netstat -s [-s] [-f address_family | -p protocol] [-w wait]
   netstat -i | -I interface -s [-f address_family | -p protocol]
   netstat -m [-m]
   netstat -r [-Aaln] [-f address_family]
   netstat -rs [-s]

4.在Mac上正确使用的方法是：即-f需要加上地址族，-p需要加上协议TCP或者UDP等

     netstat [-AaLlnW] [-f address_family | -p protocol]
     netstat [-gilns] [-v] [-f address_family] [-I interface]
     netstat -i | -I interface [-w wait] [-c queue] [-abdgqRtS]
     netstat -s [-s] [-f address_family | -p protocol] [-w wait]
     netstat -i | -I interface -s [-f address_family | -p protocol]
     netstat -m [-m]
     netstat -r [-Aaln] [-f address_family]
     netstat -rs [-s] 

   a）如果需要查询inet，netstat -anvf inet

   b）如果需要查询TCP， netstat -anvp tcp

   b）如果需要查询UDP，netstat -anvp udp

5.lsof用法

  lsof输出各列的信息的意义如下：

 command、pid、user 用来标识进程的名称、ID、拥有者

 fd文件描述符file description，应用程序通过文件描述符来识别该文件

 type、size、name，文件类型、大小、文件的确切名称

 device 磁盘的名称，

 node 索引节点，该文件在磁盘上的标识

6.  lsof -i:8080 查看8080端口的使用情况

    lsof -i4TCP:8080， 查看8080端口的TCP情况
