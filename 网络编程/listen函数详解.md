# listen 函数与两个参数详解

> 目标：搞清楚 `listen` 在 TCP 服务器中的作用，以及它的两个参数分别代表什么、如何选择合适的 backlog。

---

## 一、listen 是干嘛用的？

在 TCP 服务器端完整流程里：

```c
socket()     // 创建套接字
setsockopt() // 设置端口复用等选项（可选）
bind()       // 绑定本地 IP + 端口
listen()     // 把 socket 变成“被动监听”状态
accept()     // 从已完成队列中取出一个连接
```

其中：

- `socket` 创建的是一个“主动”的 TCP 套接字；
- `bind` 只是告诉内核：“这个套接字对应哪个 IP + 端口”；
- **`listen` 才是把这个套接字变成真正的“监听 socket”**：
  - 内核开始为它维护 **半连接队列** 和 **全连接队列**；
  - 只有调用了 `listen`，后面的 `accept` 才有意义。

**简单说：**

> `listen` = 把普通 TCP socket 切换为“被动接受连接”的服务器 socket，并设置连接队列上限。

---

## 二、函数原型

```c
#include <sys/types.h>
#include <sys/socket.h>

int listen(int sockfd, int backlog);
```

- 返回值：
  - 成功：`0`
  - 失败：`-1`，并设置 `errno`

---

## 三、第一个参数：sockfd

```c
int listen(int sockfd, int backlog);
```

- `sockfd`：
  - 必须是：
    - 用 `socket(AF_INET, SOCK_STREAM, 0)` 或类似方式创建的 **TCP socket**；
    - 并且已经成功 `bind` 到某个本地地址（IP + 端口）。
  - 如果没 `bind` 就 `listen`，系统可能自动帮你绑定到一个临时端口，但这对服务器基本没用：
    - 客户端根本不知道该连哪个端口。

典型用法：

```c
int lfd = socket(AF_INET, SOCK_STREAM, 0);
// setsockopt(lfd, ...) 可选
bind(lfd, (struct sockaddr*)&addr, sizeof(addr));
listen(lfd, SOMAXCONN);
```

注意：

- 这个 `lfd`（listen fd）只用来 `accept` 新连接；
- 每调用一次 `accept`，内核会返回一个 **新的 fd**（已连接 socket）给你；
- `lfd` 本身不会用于读写数据。

---

## 四、第二个参数：backlog

```c
int listen(int sockfd, int backlog);
```

`backlog` 这个参数是最容易被忽略、但又很重要的一个：

> 它指定了内核中 **“已完成连接队列”（全连接队列）** 的最大长度。

### 4.1 两个队列的概念

在 Linux 内核（基于经典设计）中，针对一个监听 socket 通常有两个队列：

1. **半连接队列（SYN 队列）**：
   - 存放那些只收到了 SYN、尚未完成三次握手的连接；

2. **全连接队列（accept 队列）**：
   - 存放 **已经完成了三次握手，但用户程序还没有调用 `accept` 取走** 的连接；

`backlog` 主要影响的是 **全连接队列的长度上限**。

### 4.2 backlog 太小会怎样？

- 当全连接队列满了：
  - 新完成握手的连接将被丢弃或重置（不同实现行为略有差异）；
  - 客户端可能看到：连接超时、RST、`ECONNREFUSED` 等；

这意味着：

> 如果你的服务器 `accept` 处理不够快，又把 `backlog` 设得很小，就会在高并发时出现“明明服务没挂，但客户端连不上来”的情况。

### 4.3 backlog 的有效值与 SOMAXCONN

在代码里常见写法：

```c
listen(lfd, SOMAXCONN);
```

- `SOMAXCONN` 是一个系统定义的“建议最大 backlog 值”；
- 实际可用最大值通常还受 `/proc/sys/net/core/somaxconn` 等内核参数限制；
- 就算你写了一个很大的 backlog，内核也可能自动“截断”到它允许的上限。

**建议**：

- 普通应用：直接写 `SOMAXCONN`；
- 高并发服务：可以适当调大内核参数（系统层面），再配合较大的 `backlog`。

### 4.4 backlog 的常用取值

- 开发/测试环境：可以写一个中等值，例如 `128`、`256`；
- 生产环境：一般直接用 `SOMAXCONN`，再配合系统参数调整；

示例：

```c
if (listen(lfd, SOMAXCONN) < 0) {
    perror("listen");
    exit(1);
}
```

---

## 五、listen 之后会发生什么？

当你对一个 `bind` 过的 TCP socket 调用 `listen` 后：

- 内核开始在此端口上接收 TCP SYN（连接请求）；
- 完成三次握手的连接被放入全连接队列中；
- 你的程序调用 `accept(lfd, ...)`：
  - 如果队列不为空，立即返回一个新 fd；
  - 如果队列为空，在阻塞模式下会睡眠等待新连接到来；

注意：

- **`listen` 本身并不阻塞**；
- 阻塞的是 `accept`（在监听 socket 为阻塞模式时）。

---

## 六、常见错误与坑

1. **忘记 `listen`**

   ```c
   socket();
   bind();
   // 忘了 listen
   accept(); // 这里会失败，errno=EINVAL
   ```

2. **`backlog` 设得过小**

   - 在高并发下，连接频繁超时或被拒绝；
   - 日志里可能看到大量错误，但服务看着“没挂”。

3. **以为 backlog 控制的是总连接数**

   - 实际上它控制的是“处于 accept 队列中的待处理连接的数量”；
   - 一旦连接被 `accept` 取出，就不再占用这个队列；
   - 总连接数还取决于：
     - 你 accept 多少
     - 每个已连接 socket 是否还在使用

4. **listen 在 UDP 上调用**

   - UDP 是无连接的，`listen` 对 `SOCK_DGRAM` 没意义；
   - 只能在 `SOCK_STREAM`（TCP）或某些“类似流”的协议上使用。

---

## 七、小结

- `listen(sockfd, backlog)` 的本质：
  - `sockfd`：一个已经 `bind` 到本地 IP:port 的 TCP socket；
  - `backlog`：控制“已完成握手、等待 `accept` 的连接队列”的最大长度。
- 典型模式：

  ```c
  int lfd = socket(AF_INET, SOCK_STREAM, 0);
  // setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, ...);
  bind(lfd, (struct sockaddr*)&addr, sizeof(addr));
  listen(lfd, SOMAXCONN);

  for (;;) {
      int cfd = accept(lfd, ...);
      // 用 cfd 和客户端收发数据
  }
  ```

理解了 `listen` 的这两个参数和背后两个队列的含义，你就真正搞懂了“服务器为什么能同时排队接待很多个 TCP 连接”，以及为什么高并发服务器需要同时调节 `backlog` 和系统内核参数。