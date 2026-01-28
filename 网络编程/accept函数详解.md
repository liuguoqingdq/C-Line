# `accept` 函数详解

> 目标：搞清楚 `accept` 在 TCP 服务器中的角色、函数原型和参数含义、阻塞/非阻塞行为、错误处理和多连接场景下的典型用法。

---

## 一、`accept` 在服务器流程中的位置

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

其中：

- `socket` + `bind` + `listen`：准备好“听众席”（监听 socket）；
- **`accept`：真正“接待一位来访者”，为这位客户端创建一个新的通信通道（新的 fd）**；
- 后续与该客户端通信的是 `accept` 返回的那个 fd，而不是监听 fd。

可以把监听 fd (`lfd`) 比喻为“前台”的号码机，而 `accept` 就是：

> 从排队队列里取一个号码 → 为这个客户安排一间独立的会客室（新 fd）。

---

## 二、函数原型

标准原型：

```c
#include <sys/types.h>
#include <sys/socket.h>

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

Linux 上还有一个扩展版本 `accept4`：

```c
int accept4(int sockfd, struct sockaddr *addr, socklen_t *addrlen, int flags);
```

`accept4` 允许在接受时直接设置 `O_NONBLOCK` / `SOCK_CLOEXEC` 等，这里先聚焦 `accept`。

---

## 三、三个参数详细说明

### 3.1 `sockfd`：监听 socket 的 fd

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

- `sockfd` 必须是：
  - 使用 `socket(AF_INET, SOCK_STREAM, 0)`（或类似）创建的 TCP socket；
  - 已经成功 `bind` 到本地地址；
  - 已经成功 `listen`，处于监听状态。

- 这个 `sockfd` 通常被称为 **监听 fd**（`listen_fd` / `lfd`）：
  - 只用来 "等待并接收" 新连接（配合 `accept`）；
  - **不用于收发业务数据**。

如果对未监听的 TCP socket 调用 `accept`，会返回 `-1`，`errno = EINVAL`。

### 3.2 `addr`：用于返回对端地址

- 类型：`struct sockaddr *`，实际一般传 `struct sockaddr_in` 或 `sockaddr_in6` 的地址强转而来：

  ```c
  struct sockaddr_in cliaddr;
  socklen_t clilen = sizeof(cliaddr);

  int connfd = accept(listenfd, (struct sockaddr*)&cliaddr, &clilen);
  ```

- 调用成功后：
  - `cliaddr` 中会填入对端（客户端）的 IP 和端口等信息；
  - 可以使用 `inet_ntop` + `ntohs` 打印：

    ```c
    char ip[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &cliaddr.sin_addr, ip, sizeof(ip));
    printf("new connection from %s:%u\n", ip, ntohs(cliaddr.sin_port));
    ```

- 如果你对对端地址不感兴趣，可以把 `addr` 传 `NULL`，`addrlen` 也传 `NULL`：

  ```c
  int connfd = accept(listenfd, NULL, NULL);
  ```

### 3.3 `addrlen`：地址结构体长度（入参 + 出参）

- 调用前：
  - 需要把 `*addrlen` 设为 `addr` 所指向缓冲区的大小，例如：

    ```c
    struct sockaddr_in cliaddr;
    socklen_t clilen = sizeof(cliaddr);
    int connfd = accept(listenfd, (struct sockaddr*)&cliaddr, &clilen);
    ```

- 调用后：
  - 内核会把实际写入的地址长度返回到 `*addrlen` 中。

在 IPv4 的常用场景下，一般写死为 `sizeof(struct sockaddr_in)` 即可；在需要支持多种协议族的通用接口中，`addrlen` 的出参属性更有意义。

---

## 四、返回值：新连接 fd

```c
int connfd = accept(listenfd, ...);
```

- 成功：返回一个 **新的文件描述符**：
  - 表示一个与某个客户端之间 **已建立的 TCP 连接**；
  - 后续的 `read/write`（或 `recv/send`）都在这个 fd 上进行；
  - 原来的 `listenfd` 仍然保持监听状态，可以继续 `accept` 新连接。

- 失败：返回 `-1`，并设置 `errno`。

常见错误码：

- `EINTR`：调用被信号中断 → 通常可以重试
- `ECONNABORTED`：连接在三次握手后被意外中止
- `EMFILE`：单进程 fd 数量达到上限（`ulimit -n`）
- `ENFILE`：系统级 fd 总数达到上限
- `EAGAIN`/`EWOULDBLOCK`：在非阻塞模式下，当前没有可接受的连接

典型返回值处理（阻塞模式）：

```c
int connfd;
for (;;) {
    connfd = accept(listenfd, (struct sockaddr*)&cliaddr, &clilen);
    if (connfd < 0) {
        if (errno == EINTR) {
            continue; // 被信号打断，重试
        }
        perror("accept");
        break;
    }

    // 处理 connfd ...
}
```

---

## 五、阻塞与非阻塞行为

### 5.1 阻塞模式（默认情况）

默认情况下，socket 是阻塞的：

- 如果全连接队列中有已完成握手的连接：
  - `accept` 立即返回一个新 fd；
- 如果队列为空：
  - `accept` 会阻塞（睡眠），直到有新连接建立或被信号中断。

这就是最经典的“阻塞式服务器模型”：

```c
for (;;) {
    int connfd = accept(listenfd, (struct sockaddr*)&cliaddr, &clilen);
    // 阻塞在这里等待新连接
    // 一旦返回，处理这个 connfd
}
```

### 5.2 非阻塞模式[[accept非阻塞设置]]

当把监听 fd 设为非阻塞（`O_NONBLOCK`）后：

- 如果队列中没有可接受的连接：
  - `accept` **不会阻塞**，直接返回 `-1`，`errno = EAGAIN` 或 `EWOULDBLOCK`；
- 如果有连接：
  - 立即返回一个新 fd。

非阻塞模式通常配合 I/O 多路复用（`select/poll/epoll`）使用：先用 `epoll_wait` 等发现 `listenfd` 可读，再调用 `accept` 取出所有新连接。

简单示例：

```c
int connfd;
for (;;) {
    connfd = accept(listenfd, (struct sockaddr*)&cliaddr, &clilen);
    if (connfd < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            // 当前没有新连接，稍后再试
            break;
        } else if (errno == EINTR) {
            continue; // 信号中断，重试
        } else {
            perror("accept");
            break;
        }
    }

    // 成功接受到 connfd，加入 epoll 监控或放入线程池处理
}
```

---

## 六、与 listen/backlog/连接队列的关系

在调用 `listen` 后，内核会为 `listenfd` 维护两个队列：

1. **半连接队列（syn queue）**：只收到 SYN，还没完成 3 次握手；
2. **全连接队列（accept queue）**：3 次握手完成，等待 `accept` 的连接。

`accept`：

- 从 **全连接队列** 中取出一个连接；
- 返回一个新的 fd，同时从队列中移除该连接项；
- 如果队列为空：
  - 阻塞模式：睡眠等待；
  - 非阻塞模式：立即返回 `EAGAIN`。

`backlog`（`listen` 的第二个参数）：

- 控制全连接队列上限；
- 如果全连接队列满了，新完成握手的连接可能：
  - 被丢弃；
  - 或者重置，导致客户端连接错误。

因此，在高并发服务器中：

- `accept` 要及时从队列中取走连接；
- `backlog` 需要设置得足够大；
- 还可能需要调整系统参数（如 `/proc/sys/net/core/somaxconn` 等）。

---

## 七、典型使用模式

### 7.1 单线程、一次一个客户端（教学/demo）

```c
int listenfd = socket(...);
bind(listenfd, ...);
listen(listenfd, SOMAXCONN);

for (;;) {
    struct sockaddr_in cli;
    socklen_t len = sizeof(cli);

    int connfd = accept(listenfd, (struct sockaddr*)&cli, &len);
    if (connfd < 0) {
        if (errno == EINTR) continue;
        perror("accept");
        break;
    }

    // 处理这个客户端，直到断开
    handle_client(connfd);
    close(connfd);
}

close(listenfd);
```

特点：

- 简单直观，但一次只能服务一个客户端；
- 适合教学、演示基本流程。

### 7.2 多进程或多线程模型

- `accept` 返回后，为每个新连接创建一个子进程/线程处理：

```c
for (;;) {
    int connfd = accept(listenfd, (struct sockaddr*)&cli, &len);
    if (connfd < 0) {
        // 错误处理
        continue;
    }

    pid_t pid = fork();
    if (pid == 0) {
        // 子进程
        close(listenfd);      // 子进程不再需要监听 fd
        handle_client(connfd);
        close(connfd);
        exit(0);
    } else if (pid > 0) {
        // 父进程
        close(connfd);        // 父进程交给子进程处理后，自己不再使用这个 connfd
    } else {
        perror("fork");
        close(connfd);
    }
}
```

或者：用线程池 + `accept` + 任务队列的方式，这属于并发模型设计范畴。

### 7.3 配合 epoll 的高并发模型（简略）

- `listenfd` 用 `epoll` 监控 `EPOLLIN`；
- `epoll_wait` 返回 `listenfd` 可读，说明有新连接；
- 在事件处理逻辑中循环 `accept`，直到返回 `EAGAIN`；
- 对每个 `connfd` 设置非阻塞、加入 `epoll` 监控。

---

## 八、常见错误与注意事项

1. **忘记 `listen` 就 `accept`**

   - 结果：`accept` 返回 `-1`，`errno = EINVAL`。

2. **对 `accept` 返回值不做错误处理**

   - 忽略 `EINTR`、`EAGAIN`，或者在非阻塞模式下忙等，会导致 CPU 飙高或连接异常。

3. **忘记关闭 `connfd`**

   - 每次 `accept` 都会创建一个新 fd，如果不用了不 `close`，会耗尽进程的 fd 上限（`EMFILE`）。

4. **多进程服务器中忘记在子进程关闭 `listenfd`**

   - 会导致监听 socket 的引用计数增加，父进程退出/重启后，端口仍被占用。

5. **在非阻塞模式下没正确处理 `EAGAIN`**

   - 可能出现空转循环狂刷 `accept` 导致 CPU 100%。

---

## 九、小结

- `accept` 的核心作用：
  - 从内核的“已完成连接队列”中取出一个连接；
  - 返回一个新的 fd，用于与该客户端收发数据。

- 调用条件：
  - `sockfd` 必须是已 `bind` + `listen` 的监听 socket；

- 参数含义：
  - `sockfd`：监听 fd；
  - `addr` / `addrlen`：用于返回客户端地址信息，如无需要可传 `NULL`；

- 阻塞/非阻塞行为：
  - 阻塞：无连接则睡眠等待；
  - 非阻塞：无连接则立即返回 `EAGAIN`；

- 多连接处理：
  - 多进程、多线程、epoll 等模式都围绕 `accept` 取得的新 fd 展开。

把 `accept` 放到整个 TCP 服务器生命周期中去理解，它就不再是一个“孤立的系统调用”，而是真正把抽象的“连接请求”变成一个可以 `read/write` 的具体 fd 的关键一步。

