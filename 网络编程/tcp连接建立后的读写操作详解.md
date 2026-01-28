# TCP 连接建立后的读写操作详解

> 目标：在已经通过 `socket + connect`（客户端）或 `socket + bind + listen + accept`（服务端）建立 TCP 连接之后，系统性掌握**有哪些 I/O 操作可以用来读写数据**，它们的区别、注意事项，以及如何正确处理“部分读写”和消息边界。

假设你已经拿到了一个已连接的 TCP `fd`：

- 客户端这边：`int fd = socket(...); connect(fd, ...);` 之后的 fd
- 服务端这边：`int connfd = accept(listenfd, ...);` 返回的 fd

下面所有的读写操作都围绕这个 `fd` 展开。

---

## 一、TCP 是“字节流”：没有消息边界

首先必须建立一个**正确心智模型**：

> TCP 提供的是可靠、有序、双向的 **字节流 (byte stream)**，而不是“消息包”。

含义：

- 你 `send` 或 `write` 的每一次调用，并不对应对端一次 `recv` / `read` 调用；
- 对端可能：
  - 一次 `read` 读到多次 `send` 合并后的数据；
  - 也可能一次 `send` 被对端多次 `read` 分拆拿到；
- **应用层需要自己定义“消息边界”**：例如
  - 以 `\n` 换行结尾的文本协议（HTTP 请求行、Redis 文本协议等）；
  - 先发固定长度的消息头（包含 body 长度），再发 body；
  - 定长协议，每条消息就是固定字节数。

所以，后面讲所有读写函数时，要记住：

- **读写的是“多少个字节”而不是“多少条消息”。**

---

## 二、最基础的 I/O：`read` / `write`

在 POSIX 下，socket 也是“文件描述符”，因此最基础的读写函数就是：

```c
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

### 2.1 `read` 行为

- 尝试从 fd 读取最多 `count` 字节到 `buf`；
- 返回值：
  - `> 0`：实际读到的字节数（可能 **小于** `count`）
  - `== 0`：对端已关闭连接（EOF）；
  - `< 0`：出错，检查 `errno`：
    - `EINTR`：被信号中断，通常重试
    - `EAGAIN` / `EWOULDBLOCK`：非阻塞模式下当前无数据可读

### 2.2 `write` 行为

- 尝试向 fd 写入最多 `count` 字节；
- 返回值：
  - `>= 0`：实际写入字节数（可能 **小于** `count`）；
  - `< 0`：出错，检查 `errno`：
    - `EINTR`：被信号中断，可重试写剩余部分
    - `EAGAIN` / `EWOULDBLOCK`：非阻塞模式下当前无法写入更多数据。

**重点：**

> 在 TCP 上，`read` 和 `write` 都可以“部分成功”：
>
> - `read` 读到的数据长度，可以在 `[1, count]` 任意；
> - `write` 也可能一次只写成功一部分，特别是在非阻塞模式下。

因此：

- **读数据时**，如果你期望“读满 N 字节”或“读到 `\n`”，需要自己写循环；
- **写数据时**，如果你希望“把整个 buffer 可靠地发出去”，也要自己写循环。

下面会给常见的“读满 / 写满”示例。

---

## 三、socket 专用 I/O：`recv` / `send`

除了 `read/write`，socket 还提供了更“网络味”的接口：

```c
#include <sys/types.h>
#include <sys/socket.h>

ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

行为和 `read/write` 非常相似，只是多了一个 `flags` 参数，可以指定一些特殊行为，例如：

- `MSG_DONTWAIT`：这次调用使用非阻塞语义，不管 socket 自身是否被设置为非阻塞；
- `MSG_PEEK`：窥探数据，不从接收缓冲区移除（下次 `recv` 还能读到同样的数据）；
- `MSG_WAITALL`：尝试一直等到 buffer 填满或出错/EOF；
- `MSG_NOSIGNAL`：发送时若对端已关闭，不产生 `SIGPIPE` 信号，只返回错误（Linux）。

通常使用方式：

```c
ssize_t n = recv(fd, buf, sizeof(buf), 0);
ssize_t n = send(fd, buf, len, 0);
```

在大多数简单场景下，用 `recv/send` 和 `read/write` 没有本质差别；
只要统一用一套，注意好错误处理即可。

**工程上常见习惯：**

- C 代码里用 `recv/send`
- C++ 里有时也用 `read/write` 为了与文件 I/O 一致

---

## 四、写操作：如何“保证全部发送” (`sendall` 思路)

因为 `write`/`send` 可能只发送了一部分数据，所以**要把一段 buffer 全部发完，必须自己循环**：
先看 `write`/`send` 的原型（去掉不相干的参数）：

```C++
ssize_t write(int fd, const void *buf, size_t count);
ssize_t send(int fd, const void *buf, size_t len, int flags);
```

它们的语义都是：

> “**尝试** 从你的用户态缓冲区中拿出最多 `count/len` 字节，拷贝到**内核发送缓冲区**中。  
> 实际拷贝了多少，就返回多少。”

关键点：**“尝试”**两个字。

- 你**希望**发 10000 字节，不代表系统这次一定能把 10000 全部拷进去；
    
- 如果内核发送缓冲区**快满了**，可能这次只能拷 4000 字节，其余 6000 暂时放不下；
    
- 这时 `write`/`send` 的返回值是 `4000`，不是 `10000`。
    

**剩下那 6000 字节，还躺在你的 `buf` 里，一点没发出去。  
如果你不再调用一次 `write`，对端永远收不到那 6000。**
### 4.1 C 风格“发送全部”的封装

```c
#include <unistd.h>
#include <errno.h>
#include <sys/socket.h>

ssize_t send_all(int fd, const void *buf, size_t len) {
    size_t total = 0;        // 已经发送了多少
    const char *p = buf;

    while (total < len) {
        ssize_t n = send(fd, p + total, len - total, 0);
        if (n > 0) {
            total += n;
            continue;
        }
        if (n == 0) {
            // 对端关闭了连接
            break;
        }

        if (n < 0) {
            if (errno == EINTR) {
                continue; // 信号中断，重试
            }
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 非阻塞下，发送缓冲区满了，应该等待可写事件再继续
                // 这里简单返回已发送的字节数
                break;
            }
            // 其他错误
            return -1;
        }
    }

    return (ssize_t)total; // 返回总共发了多少
}
```

阻塞模式下，如果网络正常，这个函数最终要么：

- 返回 `len`（成功发完）；
- 遇到错误返回 `-1`；
- 对端中途关闭可能导致少于 `len`。

非阻塞模式下：

- 如果遇到 `EAGAIN`，需要配合 `select/poll/epoll` 等等待“可写”再继续发剩余部分。

### 4.2 C++ RAII 封装中的写接口思路

在 C++ 网络类里，可以把它封成成员函数：

```cpp
class TcpSocket {
public:
    explicit TcpSocket(int fd = -1) : fd_(fd) {}
    ~TcpSocket() { if (fd_ != -1) ::close(fd_); }

    ssize_t sendAll(const void* buf, size_t len) {
        size_t total = 0;
        const char* p = static_cast<const char*>(buf);
        while (total < len) {
            ssize_t n = ::send(fd_, p + total, len - total, 0);
            if (n > 0) { total += n; continue; }
            if (n == 0) break;
            if (errno == EINTR) continue;
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 非阻塞下需要上层配合事件循环
                break;
            }
            return -1;
        }
        return (ssize_t)total;
    }

private:
    int fd_;
};
```

---

## 五、读操作：常见三种模式

### 5.1 "读尽可能多"：一次 `read/recv`，拿到当前缓冲区所有可用数据

```c
ssize_t n = recv(fd, buf, sizeof(buf), 0);
if (n > 0) {
    // 处理 buf[0..n-1]
} else if (n == 0) {
    // 对端关闭
} else {
    if (errno == EINTR) { /* 重试 */ }
    else if (errno == EAGAIN || errno == EWOULDBLOCK) { /* 非阻塞无数据 */ }
    else { /* 其他错误 */ }
}
```

适用场景：

- echo 服务器：接多少就回多少；
- 只需要“有就读一点”，不追求一次读满。

### 5.2 "读满固定字节数"

例如：你的协议规定：

- 消息头固定 8 字节，其中包含 body 长度；
- 则你需要先 **读满 8 字节**，才能解析出 body 的长度。

封装一个 `read_n`：

```c
ssize_t read_n(int fd, void *buf, size_t n) {
    size_t total = 0;
    char *p = buf;

    while (total < n) {
        ssize_t m = recv(fd, p + total, n - total, 0);
        if (m > 0) {
            total += m;
            continue;
        }
        if (m == 0) {
            // 对端关闭
            break;
        }
        if (m < 0) {
            if (errno == EINTR) continue;
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 非阻塞模式下数据还不够，交给上层决定何时重试
                break;
            }
            return -1;
        }
    }

    return (ssize_t)total; // 实际读了多少（可能小于 n）
}
```

配合协议头：

```c
struct Header {
    uint32_t len; // body 长度 (网络序)
    uint16_t type;
};

struct Header hdr;
ssize_t hn = read_n(fd, &hdr, sizeof(hdr));
if (hn != sizeof(hdr)) {
    // 连接关闭或错误或数据不足
}

uint32_t body_len = ntohl(hdr.len);

char *body = malloc(body_len);
ssize_t bn = read_n(fd, body, body_len);
```

### 5.3 "按行读取"（到 `\n` 为止）

常见于文本协议（如简单的聊天协议、Redis 文本模式等）。基本思路：

- 建立一个缓冲区 `buffer`；
- 循环 `recv` 填充 buffer；
- 每次检查 buffer 里是否出现 `\n`，如果有，就切出一行出来处理。

简单示例（教学用，未做所有错误细节）：

```c
#include <string.h>

ssize_t read_line(int fd, char *buf, size_t maxlen) {
    size_t total = 0;

    while (total < maxlen - 1) { // 留 1 个字节给 '\0'
        char c;
        ssize_t n = recv(fd, &c, 1, 0);
        if (n > 0) {
            buf[total++] = c;
            if (c == '\n') break;
        } else if (n == 0) {
            // 对端关闭
            break;
        } else {
            if (errno == EINTR) continue;
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 非阻塞，当前读不到更多
                break;
            }
            return -1;
        }
    }

    buf[total] = '\0';
    return (ssize_t)total;
}
```

这种“一次读 1 字节”效率较差，实际工程中会使用环形缓冲区/分隔符扫描来优化（一次读一大块，自己在 buffer 里找 `\n`）。但思路是一样的。

---

## 六、关闭连接：`close` 与 `shutdown`

在读写完毕后，需要正确关闭 TCP 连接：

### 6.1 `close(fd)`

- 关闭 socket 的读写两个方向；
- 对 TCP：触发四次挥手；
- 若进程中还有其他引用（fork 后子进程），只有最后一个 `close` 才会真正关闭。

### 6.2 `shutdown(fd, how)`

```c
#include <sys/socket.h>

int shutdown(int sockfd, int how);
```

- `how`：
  - `SHUT_RD` (0)：关闭读
  - `SHUT_WR` (1)：关闭写（常用）
  - `SHUT_RDWR` (2)：读写都关闭

典型用法：

- **半关闭写方向**：

  ```c
  // 我不再发送数据了，但还要继续接收对端发来的
  shutdown(fd, SHUT_WR);

  // 此后，对端 read 会读到 EOF（返回 0），但你这边还可以 recv
  ```

- 完全关闭前仍然可以 `shutdown` 再 `close`，但直接 `close` 也可以让系统处理。

---

## 七、综合示例：最小阻塞 echo 服务器的收发逻辑

这是服务端在 `accept` 到 `connfd` 后的处理逻辑：

```c
void echo_loop(int connfd) {
    char buf[4096];

    for (;;) {
        ssize_t n = recv(connfd, buf, sizeof(buf), 0);
        if (n > 0) {
            // 原样发回去
            ssize_t sent = send_all(connfd, buf, (size_t)n);
            if (sent < 0) {
                perror("send_all");
                break;
            }
        } else if (n == 0) {
            // 对端关闭连接
            printf("client closed\n");
            break;
        } else {
            if (errno == EINTR) {
                continue; // 信号打断重试
            }
            perror("recv");
            break;
        }
    }

    close(connfd);
}
```

配合之前的 `send_all`，这是一个最基础又正确的阻塞 echo 实现：

- 每次读到多少，就发回多少；
- 正确处理 `n == 0`（对端关闭）；
- 正确处理 `EINTR`；
- 写操作使用循环保证“尽量发完”。

---

## 八、练习建议

1. **实现一个完整的阻塞式 echo server/client**：
   - 服务器：`socket + bind + listen + accept` 后，调用上面的 `echo_loop`；
   - 客户端：从 stdin 读一行 → `send_all` → `recv` 回一行 → 打印；

2. **实现“有消息头”的协议**：
   - 消息头 `Header{ uint32_t len; uint16_t type; }`；
   - 先用 `send_all` 发送 header（记得 `htonl/htons`），再发送 body；
   - 收到时先 `read_n` 读满 header，再按 `len` 读 body；

3. **尝试将 fd 设为非阻塞**：
   - 在 `read_n` / `send_all` / `read_line` 中处理 `EAGAIN`；
   - 配合 `select` 或 `epoll` 写一个简单的多连接 echo 服务器。

通过这些练习，你会把：

- TCP 字节流模型
- `read/write` vs `recv/send`
- 部分读写/循环读写
- 消息边界的应用层处理

这一整套 socket I/O 的基本功打牢，这是后面写任何网络程序（HTTP 服务器、RPC 框架、自定义协议系统）的必备基础。

