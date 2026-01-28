# 模块 4：基础 Socket API 与阻塞式 TCP/UDP 详解

> 目标：搞清楚 `socket/bind/listen/accept/connect/close/setsockopt` 等核心系统调用的作用和调用顺序，能独立写出最基本的**阻塞式 TCP/UDP 客户端与服务器**（echo 程序）。

---

## 一、Socket 在内核中的角色

### 1.1 什么是 socket？

在 Unix/Linux 下：

- 一切 I/O（文件、管道、终端、网络）都抽象为“**文件描述符（fd）**”；
- `socket()` 系统调用向内核申请了一个“网络端点对象”，并返回一个整数 fd；
- 程序后续通过 `read/write` 或 `recv/send` 等接口对这个 fd 进行读写，从而在网络上收发数据。

你可以把 socket 理解为：

> “内核里用来表示一端网络连接的对象 + 用户态看到的一个整数句柄（fd）”。

### 1.2 TCP 与 UDP 的 socket 类型

- TCP：面向连接、可靠、字节流
  - 类型：`SOCK_STREAM`
  - 协议：一般填写 `IPPROTO_TCP` 或 `0`
- UDP：无连接、不可靠、数据报（有边界）
  - 类型：`SOCK_DGRAM`
  - 协议：一般填写 `IPPROTO_UDP` 或 `0`

```c
// 创建一个 IPv4 TCP socket
int fd = socket(AF_INET, SOCK_STREAM, 0);

// 创建一个 IPv4 UDP socket
int fd = socket(AF_INET, SOCK_DGRAM, 0);
```

---

## 二、创建 & 关闭：`socket()` / `close()` / `shutdown()`

### 2.1 `socket()`：创建 socket

```c
#include <sys/types.h>
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

- `domain`：协议族（地址族）
  - `AF_INET`  ：IPv4
  - `AF_INET6` ：IPv6
- `type`：socket 类型
  - `SOCK_STREAM`：TCP
  - `SOCK_DGRAM` ：UDP
- `protocol`：具体协议，通常填 `0`，由内核自动选择默认（对于 `AF_INET+SOCK_STREAM` 就是 TCP）

返回值：

- 成功：一个新的文件描述符（>=0）
- 失败：`-1`，`errno` 描述错误原因

### 2.2 `close()`：关闭 fd

```c
#include <unistd.h>

int close(int fd);
```

- 释放该 fd 在进程中的引用：
  - 若该 socket 引用计数降为 0，内核会真正释放 socket 对象；
  - 对 TCP 来说，会触发 **四次挥手**（发送 FIN 等）。

注意：

- 两个进程/线程如果都持有某个 fd 的引用，都必须各自 `close`，才能完全释放；
- 多次 `close` 同一个 fd 或使用未初始化 fd 会导致未定义行为或关闭错误对象。

### 2.3 `shutdown()`：半关闭

```c
#include <sys/socket.h>

int shutdown(int sockfd, int how);
```

- `how` 可以是：
  - `SHUT_RD`   （0）：关闭读端
  - `SHUT_WR`   （1）：关闭写端（常见）
  - `SHUT_RDWR` （2）：同时关闭读写

意义：

- `shutdown` 只影响当前 socket 的一个方向的传输，而不会立刻释放 fd；
- 常见用法：
  - `shutdown(fd, SHUT_WR)` 告诉对方“我这边不再发送数据了，但还可以继续接收你的数据”，对端 `read` 会读到 EOF（返回 0）。

总结：

- `close`：释放 fd，最终导致连接完全关闭；
- `shutdown`：只关闭传输方向（半关闭），通常配合 `read`/`write` 实现更细粒度控制。

---

## 三、TCP 服务端：socket → setsockopt → bind → listen → accept

### 3.1 TCP 服务端典型流程

1. `socket()`：创建监听 socket
2. `setsockopt()`：设置端口复用选项（推荐）
3. `bind()`：绑定 IP + 端口
4. `listen()`：将 socket 变为“被动监听”状态
5. `accept()`：接受客户端连接，返回新的已连接 socket fd
6. 用 `read/write` 或 `recv/send` 与客户端收发数据
7. 关闭已连接 socket；最后关闭监听 socket

下面逐个说明。

### 3.2 `setsockopt()` 与端口复用[[setsockopt函数详解]]

```c
#include <sys/socket.h>

int setsockopt(int sockfd, int level, int optname,
               const void *optval, socklen_t optlen);
```

最常见用途：

- `SO_REUSEADDR`：允许“地址复用”，解决服务器重启时 `bind` 失败（`Address already in use`）的问题；
- `SO_REUSEPORT`：允许多个进程/线程绑定同一 IP:port（负载均衡场景）。

示例：

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
int opt = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
// 如果需要：
// setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
```

### 3.3 `bind()`：绑定本地 IP 和端口

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

- 对 TCP 服务端来说，`bind` 用于指定：
  - 监听在哪个本地 IP 地址（网卡）；
  - 使用哪个端口号。

常见写法（IPv4）：

```c
struct sockaddr_in servaddr;
memset(&servaddr, 0, sizeof(servaddr));

servaddr.sin_family      = AF_INET;
servaddr.sin_port        = htons(8080);       // 绑定端口 8080
servaddr.sin_addr.s_addr = htonl(INADDR_ANY); // 绑定到所有本地 IP

bind(fd, (struct sockaddr*)&servaddr, sizeof(servaddr));
```

#### `INADDR_ANY` 的含义

- 表示绑定到本机“所有可用 IPv4 地址”；
- 等价于：“不关心具体是 127.0.0.1 还是 192.168.x.x，只要是发到这个端口的包都收”。

如果只想监听本机回环地址：

```c
inet_pton(AF_INET, "127.0.0.1", &servaddr.sin_addr);
```

### 3.4 `listen()`：将 socket 设为监听状态[[listen函数详解]]

```c
int listen(int sockfd, int backlog);
```

- 将一个主动 socket（默认）转为被动监听 socket，内核开始为其维护：
  - 半连接队列（SYN 队列）；
  - 全连接队列（已完成三次握手，等待 `accept`）。

- `backlog`：全连接队列的长度上限（不同系统有调整规则）。

示例：

```c
listen(fd, SOMAXCONN); // SOMAXCONN 是系统支持的最大队列长度
```

### 3.5 `accept()`：接受客户端连接[[accept函数详解]]、[[C++网络编程 Select详解]]

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

- 从监听 socket 的“已完成连接队列”中取出一个连接：
  - 若队列为空，则在阻塞模式下会阻塞等待；
- 返回一个新的 fd：表示这个客户端连接；
- 原来的 `sockfd` 继续用于监听后续连接。

示例：

```c
struct sockaddr_in cliaddr;
socklen_t clilen = sizeof(cliaddr);

int connfd = accept(fd, (struct sockaddr*)&cliaddr, &clilen);

char ip[INET_ADDRSTRLEN];
inet_ntop(AF_INET, &cliaddr.sin_addr, ip, sizeof(ip));
printf("new connection from %s:%u\n", ip, ntohs(cliaddr.sin_port));
```

### 3.6 阻塞式 TCP echo 服务器（核心循环框架）

伪代码：

```c
int listenfd = socket(AF_INET, SOCK_STREAM, 0);
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, ...);

bind(listenfd, ...);
listen(listenfd, SOMAXCONN);

for (;;) {
    int connfd = accept(listenfd, ...);
    // 简单起见：一次只服务一个客户端，处理完再 accept 下一个
    for (;;) {
        ssize_t n = read(connfd, buf, sizeof(buf));
        if (n > 0) {
            write(connfd, buf, n); // echo back
        } else if (n == 0) {
            // 对端关闭连接
            break;
        } else {
            // 出错
            perror("read");
            break;
        }
    }
    close(connfd);
}

close(listenfd);
```

> 注意：这是**最简单的阻塞模型**，一次只处理一个客户端，用于理解流程。要支持多客户端，需要配合多进程/多线程或 I/O 多路复用。

---

## 四、TCP 客户端：socket → connect → read/write[[Tcp连接建立后的读写操作详解]]

### 4.1 `connect()`：主动发起连接

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

- 客户端调用 `connect` 发起三次握手，连接到服务端。
- 在阻塞模式下：
  - 如果连接成功，`connect` 返回 0；
  - 失败返回 -1，并设置 `errno`（如 `ECONNREFUSED` 等）。

### 4.2 阻塞式 TCP echo 客户端（核心框架）

伪代码：

```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);

struct sockaddr_in servaddr;
servaddr.sin_family = AF_INET;
servaddr.sin_port   = htons(8080);
inet_pton(AF_INET, "127.0.0.1", &servaddr.sin_addr);

connect(sockfd, (struct sockaddr*)&servaddr, sizeof(servaddr));

// 发送一行数据
write(sockfd, "hello\n", 6);

// 读取 echo 回来的数据
ssize_t n = read(sockfd, buf, sizeof(buf));

printf("recv: %.*s\n", (int)n, buf);

close(sockfd);
```

要点：

- 客户端只需要 `socket → connect → read/write`；
- 不需要 `bind`（除非你想指定本地端口/IP，一般让内核自动选择）。

---

## 五、UDP：`sendto` / `recvfrom` 基本流程

### 5.1 UDP 服务端流程（无连接）

1. `socket(AF_INET, SOCK_DGRAM, 0)` 创建 UDP socket
2. `bind()` 绑定本地 IP + 端口
3. 循环调用 `recvfrom` 接收数据 + `sendto` 回应（如果需要）

```c
int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
struct sockaddr_in servaddr;

servaddr.sin_family      = AF_INET;
servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
servaddr.sin_port        = htons(9999);

bind(sockfd, (struct sockaddr*)&servaddr, sizeof(servaddr));

for (;;) {
    char buf[1024];
    struct sockaddr_in cliaddr;
    socklen_t len = sizeof(cliaddr);

    ssize_t n = recvfrom(sockfd, buf, sizeof(buf), 0,
                         (struct sockaddr*)&cliaddr, &len);

    // 输出客户端 IP:port
    char ip[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &cliaddr.sin_addr, ip, sizeof(ip));
    printf("udp from %s:%u, len=%zd\n", ip, ntohs(cliaddr.sin_port), n);

    // 原样返回
    sendto(sockfd, buf, n, 0,
           (struct sockaddr*)&cliaddr, len);
}

close(sockfd);
```

### 5.2 UDP 客户端流程

1. `socket(AF_INET, SOCK_DGRAM, 0)`
2. 构造服务端 `sockaddr_in`
3. 使用 `sendto` 发送，`recvfrom` 接收

```c
int sockfd = socket(AF_INET, SOCK_DGRAM, 0);

struct sockaddr_in servaddr;
servaddr.sin_family = AF_INET;
servaddr.sin_port   = htons(9999);
inet_pton(AF_INET, "127.0.0.1", &servaddr.sin_addr);

sendto(sockfd, "hello", 5, 0,
       (struct sockaddr*)&servaddr, sizeof(servaddr));

char buf[1024];
ssize_t n = recvfrom(sockfd, buf, sizeof(buf), 0, NULL, NULL);

printf("recv: %.*s\n", (int)n, buf);

close(sockfd);
```

> 注意：UDP 是无连接的，不需要三次握手，也不保证可靠性和顺序。

---

## 六、TIME_WAIT 与端口复用

### 6.1 TIME_WAIT 状态

在 TCP 四次挥手结束后，**主动关闭连接的一方**会进入 `TIME_WAIT` 状态一段时间（典型是 2MSL）：

- 目的是：
  - 确保对端收到最后的 ACK；
  - 防止网络中延迟的旧报文影响之后新建的连接。

### 6.2 为何会导致“端口被占用”？

场景：

- 你写了一个 TCP 服务器绑定在 `0.0.0.0:8080`；
- 停止服务器后立即重启；
- 可能会遇到：

```text
bind: Address already in use
```

原因：

- 某些老连接仍在 `TIME_WAIT` 状态，内核不允许立即重新绑定同样的 IP:port。

### 6.3 解决方法：SO_REUSEADDR / SO_REUSEPORT

一般做法：

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
int opt = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

- `SO_REUSEADDR`：允许在 `TIME_WAIT` 状态时重新绑定相同 IP:port（符合特定条件）；
- `SO_REUSEPORT`（Linux 3.9+）：允许多个进程/线程绑定同一 IP:port 以实现更高级的负载均衡（进阶内容）。

> 一般开发服务器程序时，**习惯性在监听 socket 上设置 `SO_REUSEADDR`**，可以避免频繁重启时 `bind` 失败。

---

## 七、建议练习与进阶方向

### 7.1 阻塞式 echo server/client

1. 参考上文伪代码，写出完整的：
   - 阻塞式 TCP echo 服务器：一次只服务一个客户端
   - 阻塞式 TCP echo 客户端：从标准输入读一行，发给服务器，再打印响应
2. 加上：
   - 打印客户端 IP:port（用 `inet_ntop` + `ntohs`）
   - 为服务器监听 socket 设置 `SO_REUSEADDR`

### 7.2 RAII 封装 socket（C++）

在掌握 C 版本 API 后，可以在 C++ 中封装一个类：

```cpp
class Socket {
public:
    explicit Socket(int fd = -1) : fd_(fd) {}
    ~Socket() { if (fd_ != -1) ::close(fd_); }

    int fd() const { return fd_; }
    // 禁止拷贝，支持移动等

private:
    int fd_;
};
```

- 构造函数中保存 fd；
- 析构函数中自动 `close(fd)`；
- 这样可以用 RAII 管理 socket 生命周期，避免忘记关闭。

后续可以继续在这个类上叠加：

- `bind`/`listen`/`accept` 成员函数；
- `connect`/`send`/`recv` 封装；
- 非阻塞模式设置、错误处理封装等。

---

通过本模块，你应该对“如何从零写出最基本的阻塞式 TCP/UDP 服务端和客户端”有完整、清晰的认知。接下来在此基础上再引入 I/O 多路复用（`select/poll/epoll`）和多线程/线程池，就能迈向高并发网络编程。

