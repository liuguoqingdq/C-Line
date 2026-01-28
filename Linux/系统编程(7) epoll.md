# Linux epoll 从零到工程用法

> 目标：你看完能回答 3 个问题：
> 1) **epoll 是什么、解决什么痛点**  
> 2) **epoll 的三个核心 API（create/ctl/wait）怎么用，参数是什么意思**  
> 3) **写一个可运行的 epoll 服务器主循环**（含常见坑：非阻塞、LT/ET、短读/短写）

---

## 1. 先从问题出发：为什么需要 epoll？

### 1.1 你想做什么
典型场景：一个 TCP 服务器同时维护 **上千/上万条连接**，每条连接可能随时可读/可写。

### 1.2 朴素方案的痛点
#### A) 轮询每个 socket（最原始）
- 你得不停 `recv`/`send` 试探每个 fd
- **绝大多数时间是空转**：很多连接根本没数据

#### B) `select` / `poll` 的问题
- `select`：有 fd 数量上限（FD_SETSIZE），每次都要把位图从用户态拷贝到内核态，扫描成本高
- `poll`：没有位图上限，但每次也要把整个数组传入内核并被扫描

这类接口的共同点：
- 每次等待都要把“我关心哪些 fd”整体交给内核
- 内核需要线性扫描

当连接数很大时：
- **O(N) 扫描 + O(N) 拷贝** 成为瓶颈

### 1.3 epoll 的核心思路
**把“关注集合”注册到内核里**，以后只告诉你“哪些 fd 就绪了”。

- 你用一次 `epoll_ctl` 把 fd 加入 epoll 关注集
- 内核维护就绪队列
- 你用 `epoll_wait` 直接拿到“就绪事件列表”（不是扫描全部）

> 直觉类比：
> - `poll/select`：你每次问“这 1 万个人里谁举手了？”（每次点名/扫描）
> - `epoll`：你先登记“这 1 万个人都归我管”，之后只需要问“刚刚谁举手了？”（直接拿名单）

---

## 2. epoll 是什么（一句话）

**epoll 是 Linux 的 I/O 多路复用机制**：让一个线程/进程可以高效地等待多个文件描述符（socket、pipe 等）上的事件（可读/可写/异常）。

它通常用于“高并发网络服务器”的 Reactor 模型。

---

## 3. 三个核心 API：create / ctl / wait

### 3.1 `epoll_create1`
```c
#include <sys/epoll.h>

int epoll_create1(int flags);
```
- 返回：epoll 实例对应的 fd（常叫 `epfd`），失败返回 -1
- `flags`：
  - `0`
  - `EPOLL_CLOEXEC`：exec 时自动关闭，防 fd 泄漏（工程里建议开）

用法：
```c
int epfd = epoll_create1(EPOLL_CLOEXEC);
```

> 老接口 `epoll_create(int size)` 已过时（size 不再有意义）。

---

### 3.2 `epoll_ctl`
```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
作用：管理关注集
- `op`：
  - `EPOLL_CTL_ADD`：加入关注
  - `EPOLL_CTL_MOD`：修改关注事件
  - `EPOLL_CTL_DEL`：移除关注（event 可为 NULL）

- `event`：描述你关心什么事件 + 你想携带什么“用户数据”

`struct epoll_event`：
```c
typedef union epoll_data {
    void        *ptr;
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events; // 事件掩码
    epoll_data_t data;   // 用户数据（常放 fd 或指针）
};
```

常见 `events` 标志：
- `EPOLLIN`：可读
- `EPOLLOUT`：可写
- `EPOLLERR`：错误
- `EPOLLHUP`：挂起（对端关闭等）
- `EPOLLRDHUP`：对端关闭写方向（半关闭），对 TCP 很有用
- `EPOLLET`：边缘触发（ET）
- `EPOLLONESHOT`：一次触发后自动禁用，需要你手动 MOD 重新启用（多线程 reactor 常用）

#### 边缘触发事件和条件触发事件 
 如果epoll_ctl()的参数event中的events项设置为EPOLLET，fd上的监听方式为边缘触发（Edge-triggered），否则为条件触发（Level-triggered）。 
 考虑下面的生产者和消费者在通过UNIX管道通信时的情况。 
 1．生产者向管道写入1KB数据。 
 2．消费者在管道上调用epoll_wait()，等待管道上有数据并可读。 
 通过条件触发监视时，在步骤2中epoll_wait()调用会立即返回，表示管道可读。通过边缘触发监视时，需要步骤1发生后，步骤2中的epoll_wait()调用才会返回。也就是说，对于边缘触发，在调用epoll_wait()时，即使管道已经可读，也只有当有数据写入之后，调用才会返回。 
 条件触发是默认行为，poll()和select()就是采用这种模式，也是大多数开发者所期望的。边缘触发需要不同的编程解决方案，通常是使用非阻塞I/O，而且需要仔细检查EAGAIN。 
 
**边缘触发 

边缘触发”这个术语源于电子工程领域。条件触发是只要有状态发生就触发。边缘触发是只有在状态改变的时候才会发生。条件触发关心的是事件状态，边缘触发关心的是事件本身。 

  举个例子，对于一个读操作的文件描述符，如果是条件触发，只要文件描述符可读了，就会收到通知，是“可读”这个条件触发了通知。如果是边缘触发，当数据可读时，会接收到通知，而且通知有且仅有一次：是“有数据”这个变化本身触发了通知。
注册示例：
```c
struct epoll_event ev;
ev.events = EPOLLIN;      // 关心可读
// ev.events = EPOLLIN | EPOLLET; // 关心可读 + ET

ev.data.fd = fd;          // 把 fd 塞进去，wait 返回时你就知道是谁

if (epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev) == -1) {
    perror("epoll_ctl ADD");
}
```

---

### 3.3 `epoll_wait`
```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```
- 返回：本次返回的就绪事件数量（0 表示超时；-1 表示错误，`EINTR` 可重试）
- `events`：你提供的数组，内核把“就绪事件”写进去
- `maxevents`：数组长度
- `timeout`：
  - `-1` 永久阻塞
  - `0` 立即返回（轮询）
  - `>0` 等待毫秒数

示例：
```c
struct epoll_event evs[1024];
int n = epoll_wait(epfd, evs, 1024, -1);
for (int i = 0; i < n; i++) {
    int fd = evs[i].data.fd;
    uint32_t e = evs[i].events;
    if (e & EPOLLIN)  { /* fd 可读 */ }
    if (e & EPOLLOUT) { /* fd 可写 */ }
    if (e & (EPOLLERR|EPOLLHUP|EPOLLRDHUP)) { /* 处理关闭/错误 */ }
}
```

---

## 4. LT vs ET：最容易踩坑的地方

### 4.1 LT（Level Trigger，水平触发，默认）
含义：
- 只要 fd 仍然“可读”（缓冲里还有数据），`epoll_wait` 就会不断返回它

写法：
- 你每次读一点也行（但工程上仍建议“读到 EAGAIN”以减少 wakeup）

### 4.2 ET（Edge Trigger，边缘触发）
含义：
- 只在状态从“不可读→可读”的**边缘变化**时通知一次

因此：
- 你必须把 fd 设为 **非阻塞**
- 并且在收到 `EPOLLIN` 后，要 **循环读到 `EAGAIN`**，否则可能再也收不到通知（数据还在缓冲，但没新的边缘变化）

ET 的读循环模板：
```c
for (;;) {
    ssize_t r = read(fd, buf, sizeof buf);
    if (r > 0) { /* 处理 r 字节 */ continue; }
    if (r == 0) { /* 对端关闭 */ close(fd); break; }
    if (errno == EINTR) continue;
    if (errno == EAGAIN || errno == EWOULDBLOCK) break; // 读干净了
    /* 其他错误 */ close(fd); break;
}
```

---

## 5. 从 0 写一个“最小可运行”的 epoll TCP 服务器（LT 版）

> 这个版本先用 LT，理解最直观；你跑通后再升级 ET。

### 5.1 准备：listen socket
关键点：
- `socket`、`bind`、`listen`
- 把 `listenfd` 加入 epoll，关心 `EPOLLIN`（表示有新连接可 accept）

### 5.2 主循环骨架（核心就是这几行）
```c
int epfd = epoll_create1(EPOLL_CLOEXEC);

// 1) 把 listenfd 加入 epoll
struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = listenfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &ev);

struct epoll_event evs[1024];

for (;;) {
    int n = epoll_wait(epfd, evs, 1024, -1);
    if (n < 0) {
        if (errno == EINTR) continue;
        perror("epoll_wait");
        break;
    }

    for (int i = 0; i < n; i++) {
        int fd = evs[i].data.fd;
        uint32_t e = evs[i].events;

        if (fd == listenfd) {
            // 2) accept 新连接
            int cfd = accept(listenfd, NULL, NULL);
            if (cfd >= 0) {
                struct epoll_event cev;
                cev.events = EPOLLIN; // LT: 可读
                cev.data.fd = cfd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &cev);
            }
            continue;
        }

        // 3) 处理连接上的事件
        if (e & (EPOLLERR | EPOLLHUP | EPOLLRDHUP)) {
            close(fd);
            continue;
        }

        if (e & EPOLLIN) {
            char buf[4096];
            ssize_t r = read(fd, buf, sizeof buf);
            if (r > 0) {
                // demo：原样写回
                write(fd, buf, (size_t)r);
            } else if (r == 0) {
                close(fd);
            } else {
                if (errno != EINTR) close(fd);
            }
        }
    }
}
```

这份骨架能跑，但还不够“工程化”：
- 没有非阻塞
- 没有处理短写（write 可能没写完）
- 没有对 ET 做读到 EAGAIN

下面升级。

---

## 6. 工程化必做：把连接 fd 设为非阻塞

```c
#include <fcntl.h>

static int set_nonblock(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) return -1;
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}
```

为什么要非阻塞？
- epoll 只负责“告诉你可能就绪”，但你调用 `read/write` 仍可能阻塞（尤其 ET 模式要求非阻塞）。

工程实践：
- `listenfd` 和 `accept` 出来的 `cfd` 都设成非阻塞
- `accept4(listenfd, ..., SOCK_NONBLOCK|SOCK_CLOEXEC)` 更省事

---

## 7. ET 版主循环改动点（读/accept 都要循环到 EAGAIN）

### 7.1 listenfd 上的 accept 必须循环
因为 ET 只通知一次，可能同时来了很多连接。

```c
for (;;) {
    int cfd = accept(listenfd, NULL, NULL);
    if (cfd >= 0) { set_nonblock(cfd); /* add to epoll */; continue; }
    if (errno == EINTR) continue;
    if (errno == EAGAIN || errno == EWOULDBLOCK) break;
    /* 其他错误 */ break;
}
```

### 7.2 连接 fd 上的 read 必须循环到 EAGAIN
见上面 ET 的读循环模板。

---

## 8. 什么时候关心 EPOLLOUT？（写缓冲与背压）

很多新手误区：一直注册 `EPOLLOUT`。

实际上：
- 对 TCP 来说，fd 大多数时候都是“可写”的
- 一直监听 `EPOLLOUT` 会导致 epoll_wait 经常返回它，造成 CPU 空转

正确策略：
1) 平时只监听 `EPOLLIN`
2) 当你 `write` 返回 `EAGAIN`（内核发送缓冲满）时，说明“暂时写不动”
   - 把未写完的数据放到应用层发送队列
   - 用 `epoll_ctl MOD` **临时加上 `EPOLLOUT`**
3) 当收到 `EPOLLOUT`，继续把队列写空
   - 写空后再 `MOD` 去掉 `EPOLLOUT`

这是背压（backpressure）和高性能网络编程的关键。

---

## 9. 常见错误清单（你照着避坑）

1) **ET 模式没设非阻塞**：直接灾难（可能卡死）。
2) **ET 模式没有读到 EAGAIN / accept 到 EAGAIN**：丢事件、连接卡住。
3) **一直监听 EPOLLOUT**：CPU 飙高。
4) **只看 EPOLLIN 不处理 EPOLLHUP/EPOLLERR/EPOLLRDHUP**：死连接不清理。
5) **close(fd) 后没从 epoll 中移除**：通常 close 会自动移除，但如果你复用 fd 数值过快、逻辑混乱，可能出现“幽灵事件”。工程上建议：close 前 `EPOLL_CTL_DEL`（或用稳定的连接对象指针放进 `data.ptr`）。
6) **把 `epoll_event` 放栈上后保存指针**：epoll 只在 ctl 时读取 event 内容，不会保存你 event 指针本身；不要把“指针地址”误当成可复用数据。

---

## 10. 一句话总结

- epoll 解决的是：**大量 fd 监听时，避免每次 O(N) 扫描和拷贝**
- 你掌握 epoll 的标志：
  1) 会用 `create1/ctl/wait`
  2) 能解释 LT/ET 区别
  3) 能写出非阻塞 + 读到 EAGAIN 的 ET 模板
  4) 知道什么时候开/关 `EPOLLOUT`

---

## 11. 你想继续往“更高级”走，可以学这些

- `EPOLLONESHOT` + 线程池：避免多个线程同时处理同一连接
- `eventfd/timerfd/signalfd`：把“定时器/信号/跨线程通知”也统一进 epoll
- `io_uring`：更现代的异步 I/O（超高性能场景）

---

如果你愿意，我可以在下一步给你：
1) 一个**完整可编译**的 `epoll echo server`（LT + ET 两个版本）
2) 加上“发送队列 + 动态注册 EPOLLOUT”
3) 再加上 `readv/writev`（把 header/body 分段写回），把你前面的分散/聚集 I/O 和 epoll 串起来

