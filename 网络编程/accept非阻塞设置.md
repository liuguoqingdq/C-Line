# `accept` 函数详解（含如何把监听 fd 设置为 `O_NONBLOCK`）

> 目标：在理解 `accept` 的基础上，进一步掌握“如何把监听 fd 设置为非阻塞”，以及非阻塞模式下正确的使用方式。

---

## 一、`accept` 在服务器流程中的位置（回顾）

典型 TCP 服务器端流程：

```c
socket()     // 创建监听 socket
setsockopt() // 设置 REUSEADDR 等，可选
bind()       // 绑定本地 IP + 端口
listen()     // 变成监听状态
accept()     // 从队列中取出一个已完成连接
read/write   // 在已连接 socket 上收发数据
close()      // 关闭已连接 socket
```

- `listenfd`：监听 fd，只用于 `accept`；
- `accept` 返回的 `connfd`：与单个客户端通信的 fd，用于 `read/write`。

---

## 二、函数原型与参数（回顾）

```c
#include <sys/types.h>
#include <sys/socket.h>

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

- `sockfd`：监听 socket 的 fd（必须先 `bind` + `listen`）；
- `addr`：出参，返回客户端地址（不关心可以传 `NULL`）；
- `addrlen`：入参+出参，传入缓冲区长度，返回实际长度（不关心可传 `NULL`）；
- 返回值：
  - 成功：新连接的 fd（`connfd`）；
  - 失败：`-1`，`errno` 指示错误。

---

## 三、阻塞模式下的 `accept`

默认情况下，socket 是阻塞的：

- 全连接队列 **非空**：`accept` 立即返回一个 `connfd`；
- 全连接队列 **为空**：
  - 阻塞模式：`accept` 会睡眠等待新连接；
  - 被信号打断：返回 `-1`，`errno = EINTR`。

典型阻塞式服务器循环：

```c
for (;;) {
    struct sockaddr_in cli;
    socklen_t len = sizeof(cli);

    int connfd = accept(listenfd, (struct sockaddr*)&cli, &len);
    if (connfd < 0) {
        if (errno == EINTR) {
            continue; // 信号打断，重试
        }
        perror("accept");
        break;
    }

    handle_client(connfd);
    close(connfd);
}
```

---

## 四、非阻塞监听 fd：如何把监听 fd 设置为 `O_NONBLOCK`

要把 `accept` 从“可能阻塞”变成“立即返回（要么成功，要么 EAGAIN）”，关键是：

> **把监听 fd（`listenfd`）设置为非阻塞模式（`O_NONBLOCK`）**。

### 4.1 使用 `fcntl` 设置 `O_NONBLOCK`

```c
#include <fcntl.h>
#include <unistd.h>

int set_nonblock(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) {
        return -1;
    }
    if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) {
        return -1;
    }
    return 0;
}
```

在创建监听 socket 之后、`listen` 之前或之后，都可以设置非阻塞：

```c
int listenfd = socket(AF_INET, SOCK_STREAM, 0);
// setsockopt(...)
// bind(...)
// listen(...)

if (set_nonblock(listenfd) < 0) {
    perror("set_nonblock");
}
```

> 注意：将 `listenfd` 设为非阻塞后，**`accept` 也变成非阻塞行为**：
>
> - 队列中有连接：立即返回 `connfd`；
> - 队列为空：立即返回 `-1`，`errno = EAGAIN` 或 `EWOULDBLOCK`。

### 4.2 一次性非阻塞 + close-on-exec：`accept4`

Linux 提供了扩展接口 `accept4`：

```c
#define _GNU_SOURCE
#include <sys/socket.h>

int accept4(int sockfd, struct sockaddr *addr, socklen_t *addrlen,
            int flags);
```

- `flags` 可以是：
  - `SOCK_NONBLOCK`：让 **新返回的 `connfd`** 直接是非阻塞的；
  - `SOCK_CLOEXEC`：为新 fd 设置 `FD_CLOEXEC`（防止被 `exec` 继承）。

示例：

```c
struct sockaddr_in cli;
socklen_t len = sizeof(cli);

int connfd = accept4(listenfd, (struct sockaddr*)&cli, &len,
                     SOCK_NONBLOCK | SOCK_CLOEXEC);
if (connfd < 0) {
    if (errno == EAGAIN || errno == EWOULDBLOCK) {
        // 目前没有连接，稍后再试
    } else if (errno == EINTR) {
        // 信号打断，可考虑重试
    } else {
        perror("accept4");
    }
}
```

**对比：**

- 如果只想让 **监听 fd** 非阻塞：用 `fcntl(listenfd, ...)`；
- 如果希望：
  - 监听 fd 阻塞或非阻塞都无所谓；
  - 但每个新 `connfd` 必须是非阻塞的；
  - → 则用 `accept4` + `SOCK_NONBLOCK` 更方便。

> 推荐实践：高并发服务器通常会：
>
> - 用 `accept4(listenfd, ..., SOCK_NONBLOCK | SOCK_CLOEXEC)`；
> - 监听 fd 本身可以是阻塞或非阻塞，但在 epoll 模型中也常设为非阻塞，以统一行为。

---

## 五、非阻塞模式下如何正确使用 `accept`

当 `listenfd` 或 `connfd` 被设置成 `O_NONBLOCK` 后，`accept` 的错误处理逻辑要升级：

```c
for (;;) {
    struct sockaddr_in cli;
    socklen_t len = sizeof(cli);

    int connfd = accept(listenfd, (struct sockaddr*)&cli, &len);
    if (connfd < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            // 当前没有可接受的连接，跳出循环，等下次 epoll/select 告知
            break;
        } else if (errno == EINTR) {
            // 被信号打断，重试
            continue;
        } else {
            // 其他错误，视情况处理
            perror("accept");
            break;
        }
    }

    // 成功得到一个 connfd，可以：
    // - 若没有使用 accept4，将它也设置为非阻塞
    // - 然后加入 epoll 监控，或丢给线程池处理
}
```

### 5.1 与 epoll 的典型配合

1. `listenfd` 加入 epoll，关心 `EPOLLIN`；
2. `epoll_wait` 返回事件，若 `listenfd` 就绪：

   ```c
   if (event_fd == listenfd) {
       for (;;) {
           struct sockaddr_in cli;
           socklen_t len = sizeof(cli);
           int connfd = accept4(listenfd, (struct sockaddr*)&cli, &len,
                                SOCK_NONBLOCK | SOCK_CLOEXEC);
           if (connfd < 0) {
               if (errno == EAGAIN || errno == EWOULDBLOCK) {
                   // 已无更多连接
                   break;
               } else if (errno == EINTR) {
                   // 重试
                   continue;
               } else {
                   perror("accept4");
                   break;
               }
           }

           // 将 connfd 加入 epoll 监控读事件
           add_to_epoll(epfd, connfd);
       }
   }
   ```

- 用 `for(;;)` 循环 `accept` 是因为：一个 epoll 事件到来时，可能队列中不止一个待接受连接；
- 需要一直 `accept` 到返回 `EAGAIN` 为止。

---

## 六、典型组合：监听 fd & 连接 fd 的阻塞策略

常见几种策略：

1. **教学/入门（阻塞式）：**
   - `listenfd`：阻塞
   - `connfd`：阻塞
   - 优点：简单
   - 缺点：一次只能处理一个客户端，或需要多线程/多进程配合。

2. **简单非阻塞（手撸轮询，不推荐）：**
   - `listenfd`：非阻塞
   - 在循环中频繁调用 `accept`，检查是否有新连接 → 容易变成忙轮询，浪费 CPU。

3. **高并发 + epoll：**
   - `listenfd`：非阻塞
   - 使用 `epoll` 等 I/O 多路复用检测 `listenfd` 可读；
   - 使用 `accept4(..., SOCK_NONBLOCK | SOCK_CLOEXEC)` 确保每个 `connfd` 也是非阻塞；
   - 这是 Linux 下通用的高性能网络服务器模式。

---

## 七、小结

1. `accept` 本质：
   - 从“已完成连接队列”里取出一个客户端连接，返回新 fd；

2. 监听 fd 设为非阻塞：
   - 用 `fcntl(listenfd, F_SETFL, flags | O_NONBLOCK)`；
   - 或者直接使用 `accept4` 为新 `connfd` 加上 `SOCK_NONBLOCK`；

3. 非阻塞下的关键：
   - 正确处理 `EAGAIN/EWOULDBLOCK` 和 `EINTR`；
   - 通常配合 `epoll/select/poll` 使用，触发后在一个循环里把所有可接受连接取完。

掌握了“如何设置 `O_NONBLOCK` + 如何在非阻塞模式下写一段正确的 `accept` 循环”，基本就已经迈入了高并发网络编程的门槛。后续再叠加线程池、连接池、状态机等组件，就能完成相当复杂的服务器框架。

