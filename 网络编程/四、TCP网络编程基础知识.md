## 套接字

### 套接字地址结构

套接字编程需要指定套接字地址作为参数，不同的协议族有不同的地址结构定义方式。这些套接字地址通常以sockaddr_开头，每个协议族有一个唯一的后缀。例如，对于以太网，其结构名称为sockaddr_in。

**1.通用套接字地址**
```C++
struct sockaddr{  //套接字地址结构体
		sa_family_t sa_family;    //协议族
		char sa_data[14];    //协议族数据
}
```
在上述结构中，协议族成员变量sa_family的类型为sa_family_t,其实这个类型是unsigned short类型，因此成员变量sa_family的长度为16位，2字节。

### 1️⃣ `sa_family` —— 协议族（Address Family）

`sa_family_t sa_family;`

**作用一句话：**

> **指明 `sa_data` 里的数据“该按哪种协议格式来解释”。**

常见取值：

|值|含义|
|---|---|
|`AF_INET`|IPv4|
|`AF_INET6`|IPv6|
|`AF_UNIX`|本地套接字|
|`AF_PACKET`|原始链路层|

📌 `AF` = Address Family  
📌 有时你会看到 `PF_INET`（Protocol Family），在 BSD/Linux 中基本等价

---

### 2️⃣ `sa_data[14]` —— 协议族相关数据

`char sa_data[14];`

**作用一句话：**

> **存放“具体协议所需的地址信息”，但不关心具体含义。**

例如在 **IPv4** 中，它通常表示：

`| 端口号(2B) | IPv4 地址(4B) | 填充(8B) |`

⚠️ 注意：

- 这 14 字节的“语义”**完全取决于 `sa_family`**
    
- 对程序员来说：
    
    - **不可读**
        
    - **不可直接操作**
        
    - **不可移植**


**实际使用的套接字数据结构**
在网络程序设计使用的函数中，几乎所有套接字都用sockaddr结构作为参数。但是使用sockaddr结构不方便进行设置，在一台网中，一般采用sockaddr_in结构进行设置，定义如下：
```C++
struct sockaddr_in{
	_kernel_sa_family_t sin_family; //通常为AF_INET。2字节
	_be16  sin_port;//端口号。2字节
	struct in_addr sin_addr;   //IP地址。 4字节
	/*为了是的sockaddr与sockaddr_in的结构体大小相等，sockaddr_in保留空字节*/
	unsigned char  sin_zero[8];// 填充字段。  8字节
}
```

```C++
struct in_addr{ //IP地址结构
	_be32    s_addr;//32位网络IP地址，网络字节序
}
```

由于sockaddr_in与sockaddr结构大小是一样的，因此进行地址结构设置时，通常的方法先用sockaddr_in设置，再强制转换成sockaddr，这两个结构体大小完全一样，因此这样转换不会有副作用。

## 用户层和内核的交互过程
### 向内核传入数据的交互过程
向内核传入数据有**send()和bind()** 等函数，从内核得到数据的函数有**accept()和recv()** 等函数。从用户空间向内核空间传递参数的过程如图所示。**bind()** 函数向内核中传入的参数有两个，分别是套**接字地址结构**参数和表示**地址结构长度**的这两个参数都与地址结构有关。
![[Pasted image 20260109162620.png]]
**bind()函数签名**
```C++
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
参数sockfd，是创建的套接字标识符，addr是指向sockaddr结构体的指针，addrlen是结构体长度。
在IPV4中无法直接使用sockaddr传入参数，因此，传入参数的过程是这样的：
```C++
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port   = htons(8080);
addr.sin_addr.s_addr = htonl(INADDR_ANY);
bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));
```

### 内核传出数据的交互过程
从内核向用户空间传递参数的过程如图所示，通过地址结构的长度和套接字地址结构指针，进行地址结构参数的传出操作。通常是使用两个参数完成传出操作，一个是表示地址结构长度的参数，另一个是表示套接字地址结构的指针。
![[Pasted image 20260109164418.png]]
传出过程和传入过程的参数区别是，表示地址结构长度的参数在传入过程中是传值，而在传出过程中是通过传址完成的。内核按照用户传入的地址结构长度进行套戒指地址数据的复制，将内核中的地址结构数据复制到用户传入的地址结构指针中。

------
## TCP网络编程流程

## TCP网路编程架构
TCP网络编程有两种模式，一种是服务器模式，另一种是客户模式。
- **服务器模式**创建一个服务程序，等待客户端用户的链接，接收到用户的链接请求后，根据用户的请求进行处理。
- **客户端模式**则根据目的服务器的地址和端口进行连接，向服务器发送请求并对服务器的响应进行行数据处理。

#### 服务器端的程序设计模式

```lua
socket()  int sockfd = socket(AF_INET, SOCK_STREAM, 0);//向内核申请一个TCP套接字。
  ↓
bind()   bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));//将套接字与本地IP端口绑定
  ↓
listen()  listen(sockfd, backlog);//将主动套接字变为被动监听套接字，backlog决定队列大小上限
  ↓
accept()  int connfd = accept(sockfd, NULL, NULL);//从全连接队列中取出一个已完成三次握手的连接。
  ↓
recv() / send()   ←（循环处理客户端）recv(connfd, buf, sizeof(buf), 0);
								  send(connfd, buf, len, 0);//收发数据
  ↓
close() 关闭
```

### 客户端的程序设计模式
```C++
socket()
  ↓
connect()     connect(sockfd, (struct sockaddr*)&serv, sizeof(serv));
  ↓
send() / recv()
  ↓
close()
```


**交互流程**
```scss
┌──────────────────┐            ┌──────────────────┐
│     TCP 服务端     │            │     TCP 客户端     │
└──────────────────┘            └──────────────────┘
        │                                  │
        │ socket()                         │ socket()
        │                                  │
        │ bind()                           │
        │                                  │
        │ listen()                         │
        │                                  │
        │─────────── 等待连接 ───────────▶│ connect()
        │                                  │
        │◀────── TCP 三次握手 ────────────│
        │ accept()                         │ 
        │                                  │
        │ read()   ◀────────            write()
        │                                  │
        │ write()  ────────▶             read()
        │                                  │
        │ close()  ────────▶               │
	    |                                  |
        │                                close()
        │                                  │
      （结束）                           （结束）

```

--------
### 创建网络插口函数socket()

**1.socket函数介绍**
socket函数的原型如下，该函数建立一个协议族为domain、协议类型为type、协议编号为protocol的套接字文件描述符。如果该函数调用成功，则会返回一个表示套接字的文件描述符，如果调用失败返回-1。
```C++
#include <sys/types.h>
#include <sys/socket.h>
int socket(int domain,int type,int protocol);
```
socket函数的**参数domain用于设置网络通信的域**，socket()函数根据这个参数选择通信协议的族。**通信协议族的文件sys/socket.h中定义，包含了下表所示的值**，在以太网中应该使用PF_INET这个域。在程序设计时会发现有的代码使用了AF_INET这个值，在头文件中，**AF_INET和PF_INET的值是一致的。**

**domain的值和含义**

| 名称                     | 含义                       |
| ---------------------- | ------------------------ |
| `PF_UNIX` / `PF_LOCAL` | 本地通信（Unix 域套接字）          |
| `PF_INET`              | IPv4 Internet 协议         |
| `PF_INET6`             | IPv6 Internet 协议         |
| `PF_IPX`               | IPX（Novell）协议            |
| `PF_NETLINK`           | 内核与用户空间通信接口              |
| `PF_X25`               | ITU-T X.25 / ISO-8208 协议 |
| `PF_AX25`              | Amateur Radio AX.25 协议   |
| `PF_ATMPVC`            | 原始 ATM PVC 访问            |
| `PF_APPLETALK`         | AppleTalk 协议             |
| `PF_PACKET`            | 底层数据包访问（链路层）             |

socket函数的参数type()用于设置套接字通信类型，下表为type格式定义的值及其含义，主要有SOCK_STREAM（流式套接字）和SOCK_DGRAM(数据包套接字)等。
##### 常用 `type` 参数表（重点）

| type 值           | 含义       | 典型协议 / 用途           |
| ---------------- | -------- | ------------------- |
| `SOCK_STREAM`    | 流式套接字    | **TCP**，面向连接、可靠、字节流 |
| `SOCK_DGRAM`     | 数据报套接字   | **UDP**，无连接、不可靠、报文  |
| `SOCK_RAW`       | 原始套接字    | ICMP、IP 报文、协议分析     |
| `SOCK_SEQPACKET` | 顺序数据包套接字 | 面向连接、保序、保消息边界       |
| `SOCK_RDM`       | 可靠数据报套接字 | 很少实现                |
| SOCK_PACKET      | 专用类型     | 不能在程序中使用，从驱动设备中读数据  |
```C++
int sock = socket(AF_INET,SOCK_STREAM);//创建TCP套接字
```

--------
#### 应用层socket()函数和内核层sys_socket()函数的关系

```C++
应用层:
    socket()
        ↓  (系统调用)
内核层:
    sys_socket()    int sock = sys_socket(AF_INET, SOCK_STREAM, 0);
        ↓
    sock_create()   → 创建 struct sock | retval = sock_create(AF_INET, SOCK_STREAM, 0, &sock);
        ↓
    sock_map_fd()   → 分配 fd，建立映射 | retval = sock_map_fd(sock);
        ↓
返回 fd
```
在上面的图中，用户调用sock=socket(AF_INET,SOCK_STREAM,0),该函数会调用系统调用sys_socket(AF_INET,SOCK_STREAM,0)。系统调用函数sys_socket()分为两部分，一部分生成socket结构，另一部分与文件描述符绑定，将绑定的文件描述符的值传给应用层。内核sock结构如下（在linux/net.h中）：
```C++
/*
 * 内核中的 socket 抽象结构
 * 表示一个“套接字对象”
 */
struct socket {
    socket_state        state;      // 套接字状态（如 SS_UNCONNECTED, SS_CONNECTED）

    short               type;        // 套接字类型：SOCK_STREAM / SOCK_DGRAM 等

    unsigned long       flags;       // 套接字标志位（非阻塞、关闭等）

    struct file         *file;       // 关联的 struct file（用于 fd 机制）

    struct sock         *sk;          // 指向底层传输层 socket（TCP/UDP 的核心）

    const struct proto_ops *ops;      // 协议操作函数表（bind/connect/send/recv 等）

    struct socket_wq    wq;           // 等待队列（阻塞 / 唤醒）

    struct mutex        lock;         // socket 互斥锁（并发保护）
};
```
	内核函数sock_create()根据用户domain指定的协议族创建一个socket结构体绑定到当前的进程上，其中type与用户空间的设置值是相同的。
	 sock_map_fd()函数将socket结构与文件描述符列表中的某个文件描述符绑定，之后的操作可以查找文件描述符列表来对应内核的socket结构。

------
### 绑定一个端口
在成功建立套接字文件描述符之后，需要对套接字进行地址和端口的绑定，然后才能进行数据的接收和发送操作。

**1.bind()函数介绍**
bind()函数将长度为addlen的sockadd结构类型的参数my_addr与sockfd绑定在一起，将sockfd绑定到某个端口上，如果使用connect()函数则没有绑定的必要。绑定的函数原型如下：
```C++
#include <sys/types.h>
#include <sys/socket.h>
int bind(int sockfd,_CONST_SOCKADDR_ARG my_addr,socklen_t addrlen);
```
bind()**函数有三个参数**
- sockfd：**指定要进行绑定操作的套接字。**
- my_addr: **const struct sockaddr* 类型，指向一个描述“本地地址”的结构体，用于告诉内核要绑定到哪个地址。**
- addrlen：**my_addr的长度。为什么必须有这个参数？**
	- 不同协议族的地址结构长度不同：
    
    - IPv4：16 字节
        
    - IPv6：28 字节
        
    - Unix 域：不固定
        
	- 内核不能“猜长度”

**bind()函数实例**
```C++
int main(){
	int sockfd = socket(AF_INET,SOCK_STREAM，0);//创建TCP socket，如果要UDP，用SOCK_DGRAW
	if(sockfd==-1){
	perror("socket");
	exit(EXIT_FAILURE);
	}
	struct sockaddr_in addr;
	addr.sin_family = AF_INET;
	addr.sin_port = htons(8080);//host to net字节序转换
	addr.sin_addr_s_addr=htonl(INADDR_ANY);
	if(bind(sockfd,(struct sockaddr*)&addr,sizeof(addr))==-1){
		perror("bind()");
		exit(EXIT_FAILURE);
	}
	//`htons` 用于 16 位数据（short），`htonl` 用于 32 位数据（long）；两者都把“主机字节序”转换为“网络字节序（大端）”。
	.....
	close(sockfd);
}
```

**bind()函数与内核层sys_bind()函数的关系**
```C++
用户程序
   ↓
glibc::bind()
   ↓
syscall(SYS_bind, sockfd, addr, addrlen)
   ↓
内核态入口
   ↓
sys_bind()
   ↓
内核完成地址绑定
   ↓
返回结果
```

-----
### 监听本地端口函数listen()

listen()函数用来初始化服务器可连接队列，可以将连接的客户端添加到队列中一次处理。
```C++
#include <sys/socket.h>
int listen(int sockfd,int backlog);
```
当listen()函数运行成功时，返回值为0；当函数运行失败时，返回值为-1。**backlog表示等待队列的长度。**
在接收一个连接之前，需要用listen()函数监听端口，**listen()函数中的backlog参数表示在accept()函数处理之前在等待队列中的客户端的长度**，如果超过这个长度，客户端会返回一个ECONNERFUSED错误。

**listen()函数实例**
```C++
int main(){
	int sockfd = socket(AF_INET,SOCK_STREAM);
	if(sockfd==-1){
	perror("socket");
	exit(EXIT_FAILURE);
	}
	struct sockaddr_in my_addr;
	my_addr.sin_family=AF_INET;
	my_addr.sin_addr.s_addr=htonl(INADDR_ANY);
	my_addr.sin_port=htons(8080);
	if(bind(sockfd,(struct sockaddr*)&my_addr,sizeof(my_addr))==-1){
		perror("bind");
		exit(EXIT_FAILURE);
	}
	if(listen(sockfd,5)==-1){
	perror("listen()");
	exit(EXIT_FAILURE);
	}
	....
	close(sockfd);
}
```

**listen()与sys_listen()的关系**
真实发生顺序：
```C++
用户程序
   ↓
glibc::listen()
   ↓
syscall(SYS_listen, sockfd, backlog)
   ↓
内核态入口
   ↓
sys_listen()
   ↓
内核设置监听状态
   ↓
返回结果
```

-----
### 接收一个网路请求函数accept()

当一个客户端的连接请求到达服务器主机监听端口时，客户端的连接会在等待队列中等待，直到服务器处理接收请求。
accept()函数执行成功之后，会返回一个新的套接字文件描述符表示客户端连接，客户端连接的信息可以通过这个新的描述符来获得。因此当服务器成功处理客户端的请求连接后，会有两个文件描述符，**老的文件描述符表示正在监听的socket**，新产生的文件描述符表示客户端的连接，**send()和recv()函数通过新的文件描述符进行数据收发。**

**1.accept函数原型**
```C++
#include <sys/types.h>
#include <sys/socket.h>
int accept(int sockfd,_SOCKADDR_ARG *addr,socklen_t *addrlen);
```

通过accept()函数可以得到连接客户端的IP地址、端口和协议族等信息，**这个信息是通过参数addr来获得的。** 当accept()函数返回时，会将客户端的信息存储在参数addr中。参数addrlen表示第**2**个参数（addr）所指内容的长度，可以使用sizeof(addrlen)来获取。
！！addrlen 是一个指针不是结构体，accept()函数将这个addrlen指针传给TCP/IP协议栈。

- accept()函数返回值是新连接的客户端套接字文件描述符，服务器主机与客户端的通信是通过accept()函数返回的新的套接字文件描述符来完成的。
- accpet()函数发生错误会返回-1。

**accept()函数实例**
```C++
int main(){
	int sockfd,client_fd;
	struct sockaddr_in my_addr;
	struct sockaddr_in client_addr;
	int addr_length;
	sockfd=socket(AF_INET,SOCK_STREAM);
	if(sockfd==-1){
	perror("socket");
	exit(EXIT_FAILURE);
	}
	my_addr.sin_family=AF_INET;
	my_addr.sin_port=htons(8080);
	my_addr.sin_addr.s_addr=htonl(INADDR_ANY);
	bzero(&(my_addr.sin_zero),8);
	if(bind(sockfd,(struct sockaddr*)&my_addr,sizeof(my_addr))==-1){
	perror("bind");
	exit(EXIT_FAILURE);
	}
	if(listen(sockfd,5)==-1){
	perror("listen");
	exit(EXIT_FAILURE);
	}
	addr_length=sizeof(struct sockaddr_in);
	/*
	`addrlen`：

- **输入**：调用前，告诉内核 `addr` 缓冲区有多大
    
- **输出**：返回时，内核告诉你实际写入了多少字节
  
	*/
	client_fd=accept(sockfd,&client_addr,&addr_length);
	if(client_fd==-1){
	perror("client_fd");
	exit(EXIT_FAILURE);
	}
	....
	close(client_fd);
	close(sock_fd);
}
```
**！！！在使用addr_length之前，必须初始化为sizeof(addr)**
### 为什么要这样做？

- 内核在 `accept()` 时：
    
    - 会先读取 `*addrlen`
        
    - 判断 `addr` 是否有足够空间
        
- 如果你**不初始化 `len`**：
    
    - `*addrlen` 是随机值
        
    - 可能导致：
        
        - `accept` 失败（`EINVAL` / `EFAULT`）
            
        - 地址信息被截断
            
        - 未定义行为

----

## 连接目标网络服务器函数connect()
客户端在建立套接字之后，不用进行地址绑定，就可以直接连接服务器。连接服务器的函数为connect()，此函数连接指定参数的服务器，如IP地址和端口等。

**1.connect()函数介绍**
```C++
#include <sys/types.h>
#include <sys/socket.h>
int connect(int sockfd,_CONST_SOCKADDR_ARG addr,int addrlen);
```
- 参数sockfd是建立套接字时返回的文件描述符，他是由系统调用socket函数返回的。
- 参数addr是指向一个数据结构sockaddr的指针，其中包括客户端需要连接的服务器目的端口和ip以及协议类型。
- 参数addrlen表示addr参数指定的结构体所占用的内存大小不。
- **connect()函数成功返回0，发生错误返回-1，可以查看errno查看错误原因**

**2.connect函数实例**
```C++
int main(){
	int sockfd = socket(AF_INET,SOCK_SREAM,0);
	if(sockfd==-1){
	perror("socket");
	exit(EXIT_FAILURE);
	}
	struct sockaddr_in addr;
	addr.sin_family=AF_INET;
	addr.sin_port=htons(8080);
	addr.sin_addr.s_addr=htonl(INADDR_ANY);
	bzero(&(addr.sin_zero),8);
	int ret = connect(sockfd,(struct sockaddr*)&addr,sizeof(addr));
	if(ret==-1){
	perror("connect");
	exit(EXIT_FAILURE);
	}
	...
	close(sockfd)
}
```


-----
### 写入函数write()

当服务器接收到一个客户端的连接时，可以通过套接字描述符进行数据的写入操作。
```C++
int size;
char data[1024];
size = write(s,data,1024);
```

----
### 读数据read()
```C++
size=read(s,data,1024);
```

---
### 关闭套接字函数shutdown()
关闭socket连接可以使用close()函数实现，该函数的作用是关闭已经打开的socket连接，内核会释放相关资源，关闭套接字之后，就不能够再使用这个套接字文件描述符进行读写操作了。
**shutdown函数可以使用多种更多的方式关闭连接。** 允许单方向切断通信或者切断双方的通信。该函数签名如下：
```C++
#include <sys/socket.h>
int shutdown(int s,int how);
```
- s为套接字文件描述符
- how为关闭方法：
	- SHUT_RD：值为0，表示切断读，之后不能使用该文件描述符进行读操作。
	- SHUT_WR：值为1，表示切断写，之后不能使用该文件描述符进行写操作。
	- SHUT_RDWR：值为2，表示切断读写，之后不能使用该文件描述符进行读写操作，与close()函数功能相同。

-----
### 截取信号实例

#### 信号处理
信号是发生某件事情时的一个通知，有时也将其称为软中断。信号将事件发送给相关的进程，相关进程可以捕捉信号并处理。信号的捕捉由系统自动完成，信号处理函数的注册通过函数signal()完成。函数signal()原型如下：
```C++
#include <signal.h>
typedef void (*sighandler_t)(int);//宏函数
sighandler_t signal(int signum,sighandler_t handler);
```
signal()函数向信号signum注册一个void(\*sighandler_t)(int)函数，函数句柄为handler。当进程捕捉到注册信号时，会调用响应函数的句柄handler。**信号处理函数会在系统默认的函数处理之前被调用。**

#### SIGPIPE信号
如果正在写入套接字，那么当读取端关闭时，可以得到一个SIGPIPE信号。SIGPIPE信号会终止当前进程，因为信号系统在调用系统默认处理函数之前会先调用用户注册的函数，所以可以通过注册SIGPIPE信号的处理函数来获取这个信号，并进行相应的处理。

#### SIGINT信号
SIGINT信号通常是由于按Ctrl+C快捷键终止进程形成的，与Ctrl+C一样，kill命令默认会向当前活动的进程发送SIGINT信号，用户终止进程。
