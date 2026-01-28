# Linux 系统编程：epoll 完整教学（分步拆解）

你要的是“完整教学 + 一步一步拆解”。下面按 **从最小可跑 → 监听 socket → 多客户端 echo → ET/写缓冲进阶** 的顺序讲 `epoll`。每一步都是**独立可编译运行**的小程序。

> 编译统一用：`gcc xxx.c -O2 -Wall -Wextra -o xxx`

---

## 1. epoll 是什么（先建立模型）

### 1.1 为什么需要 epoll
- `select/poll`：每次调用都要把“我要关心的 fd 列表”交给内核，返回后你还要扫描整张表（典型 O(n)）。
- `epoll`：把“关心哪些 fd”这件事**注册一次**到内核里，之后 `epoll_wait()` 每次只返回“就绪的那批事件”（更适合大量连接）。

### 1.2 epoll 不是一个函数，是一套接口
最常用“三件套”：
1) `epoll_create1()`：创建 epoll 实例（返回一个 epoll fd）
2) `epoll_ctl()`：把某个 fd 加入/修改/删除到 epoll 的兴趣列表
3) `epoll_wait()`：等待事件发生，返回就绪事件数组

---

## 2. 你必须掌握的 API 与结构

### 2.1 创建 epoll 实例
```c
#include <sys/epoll.h>

int epoll_create1(int flags); // 常用 flags=EPOLL_CLOEXEC
```
- 返回值 `epfd`：也是一个 fd
- 用完要 `close(epfd)`

### 2.2 注册/修改/删除
```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *ev);
// op: EPOLL_CTL_ADD / EPOLL_CTL_MOD / EPOLL_CTL_DEL
```

`struct epoll_event`：
```c
struct epoll_event {
    uint32_t events; // 你关心的事件组合
    epoll_data_t data; // 你希望回传的“身份信息”
};

typedef union epoll_data {
    void    *ptr;
    int      fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
```

常用 `events` 位：
- `EPOLLIN`：可读
- `EPOLLOUT`：可写
- `EPOLLERR`：错误（通常要处理）
- `EPOLLHUP`：挂起
- `EPOLLRDHUP`：对端关闭写方向（TCP 半关闭常见）
- `EPOLLET`：边缘触发（ET）
- `EPOLLONESHOT`：一次性触发（多线程场景常用）

### 2.3 等待事件
```c
int epoll_wait(int epfd, struct epoll_event *evs, int maxevents, int timeout);
```
- 返回 `n`：**本次就绪事件的数量**
- 结果写在 `evs[0..n-1]`
- `evs[n..maxevents-1]` 不属于本次结果，不能读

---

## 3. 触发模式：LT vs ET（非常关键）

### 3.1 LT（Level Trigger，默认）
- 只要“仍然可读/可写”，就会持续通知。
- 代码更直观，更不容易丢事件。

### 3.2 ET（Edge Trigger）
- 只有“从不可读→可读”这种边缘变化才通知一次。
- **必须配合非阻塞 fd**，并在回调里：
  - 读：一直 `read/recv` 到 `EAGAIN`
  - accept：一直 `accept` 到 `EAGAIN`
  - 写：一直 `write/send` 到 `EAGAIN`

> 先学 LT，把骨架写对；再学 ET。

---

## 4. Step 1：最小程序（epoll 监控 stdin）

目标：你按回车输入一行，`epoll_wait()` 返回，读取并打印。

### step1_epoll_stdin.c
```c
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <sys/epoll.h>
#include <unistd.h>

int main() {
    int epfd = epoll_create1(EPOLL_CLOEXEC);
    if (epfd < 0) { perror("epoll_create1"); return 1; }

    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = STDIN_FILENO;

    if (epoll_ctl(epfd, EPOLL_CTL_ADD, STDIN_FILENO, &ev) < 0) {
        perror("epoll_ctl");
        return 1;
    }

    printf("Type something and press Enter...\n");

    for (;;) {
        struct epoll_event out[8];
        int n = epoll_wait(epfd, out, 8, -1);
        if (n < 0) {
            if (errno == EINTR) continue;
            perror("epoll_wait");
            break;
        }

        for (int i = 0; i < n; i++) {
            if (out[i].data.fd == STDIN_FILENO && (out[i].events & EPOLLIN)) {
                char buf[256];
                ssize_t r = read(STDIN_FILENO, buf, sizeof(buf)-1);
                if (r <= 0) { printf("stdin closed\n"); goto done; }
                buf[r] = '\0';
                if (strcmp(buf, "quit\n") == 0) goto done;
                printf("You typed: %s", buf);
            }
        }
    }

done:
    close(epfd);
    return 0;
}
```

运行：
```bash
gcc step1_epoll_stdin.c -O2 -Wall -Wextra -o step1
./step1
```

你要看到：输入一行后，`epoll_wait` 返回，`out[0..n-1]` 给你就绪事件。

---

## 5. Step 2：监听 socket + epoll（新连接 accept）

目标：
- 监听端口
- `listen_fd` 可读时 `accept` 一个连接并打印
- 暂时不处理客户端读写

### step2_epoll_accept.c
```c
#include <arpa/inet.h>
#include <errno.h>
#include <netinet/in.h>
#include <stdio.h>
#include <string.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

static int create_listen(int port) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) return -1;
    int yes = 1;
    setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons((unsigned short)port);

    if (bind(fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) return -1;
    if (listen(fd, 128) < 0) return -1;
    return fd;
}

int main() {
    int port = 8080;
    int listen_fd = create_listen(port);
    if (listen_fd < 0) { perror("listen"); return 1; }

    int epfd = epoll_create1(EPOLL_CLOEXEC);
    if (epfd < 0) { perror("epoll_create1"); return 1; }

    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev) < 0) {
        perror("epoll_ctl add listen");
        return 1;
    }

    printf("Listening on %d, try: nc 127.0.0.1 %d\n", port, port);

    for (;;) {
        struct epoll_event out[16];
        int n = epoll_wait(epfd, out, 16, -1);
        if (n < 0) {
            if (errno == EINTR) continue;
            perror("epoll_wait");
            break;
        }

        for (int i = 0; i < n; i++) {
            if (out[i].data.fd == listen_fd && (out[i].events & EPOLLIN)) {
                int cfd = accept(listen_fd, NULL, NULL);
                if (cfd >= 0) {
                    printf("New client accepted: fd=%d\n", cfd);
                    close(cfd); // 先关掉，下一步再加入 epoll
                }
            }
        }
    }

    close(epfd);
    close(listen_fd);
    return 0;
}
```

---

## 6. Step 3：多客户端 Echo（LT + 非阻塞，教学版正确骨架）

这一关开始像“真正的服务器”了。

### 6.1 为什么要非阻塞
即使 epoll 告诉你“可读”，你 `recv` 一次也可能读不完；
并且在 ET 模式下必须读到 `EAGAIN`。

所以我们先把 fd 全部设为 `O_NONBLOCK`，并采用**读到 EAGAIN**的循环写法（这在 LT 下也更高吞吐）。

### 6.2 代码：step3_epoll_echo_lt.c
```c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <netinet/in.h>
#include <stdio.h>
#include <string.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

static int set_nonblock(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags < 0) return -1;
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

static int create_listen(int port) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) return -1;

    int yes = 1;
    setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons((unsigned short)port);

    if (bind(fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) return -1;
    if (listen(fd, 128) < 0) return -1;
    if (set_nonblock(fd) < 0) return -1;
    return fd;
}

static void add_epoll(int epfd, int fd, uint32_t events) {
    struct epoll_event ev;
    ev.events = events;
    ev.data.fd = fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
}

static void del_close(int epfd, int fd) {
    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
    close(fd);
}

int main() {
    int port = 8080;
    int listen_fd = create_listen(port);
    if (listen_fd < 0) { perror("listen"); return 1; }

    int epfd = epoll_create1(EPOLL_CLOEXEC);
    if (epfd < 0) { perror("epoll_create1"); return 1; }

    // 监听 socket：可读表示可以 accept
    add_epoll(epfd, listen_fd, EPOLLIN);

    printf("Echo server on %d. Try: nc 127.0.0.1 %d\n", port, port);

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

            if (e & (EPOLLERR | EPOLLHUP)) {
                if (fd != listen_fd) del_close(epfd, fd);
                continue;
            }

            if (fd == listen_fd) {
                // accept 要“接干净”，直到 EAGAIN
                for (;;) {
                    int cfd = accept4(listen_fd, NULL, NULL, SOCK_NONBLOCK);
                    if (cfd < 0) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) break;
                        perror("accept4");
                        break;
                    }
                    add_epoll(epfd, cfd, EPOLLIN | EPOLLRDHUP);
                }
                continue;
            }

            if (e & EPOLLRDHUP) {
                del_close(epfd, fd);
                continue;
            }

            if (e & EPOLLIN) {
                // 读到 EAGAIN
                char buf[4096];
                for (;;) {
                    ssize_t r = recv(fd, buf, sizeof(buf), 0);
                    if (r > 0) {
                        // 教学版：直接回显（真实工程要处理短写 + 写缓冲）
                        ssize_t w = send(fd, buf, (size_t)r, 0);
                        if (w < 0 && (errno == EAGAIN || errno == EWOULDBLOCK)) {
                            // 这里为了教学不展开写缓冲，简单丢弃
                            // 下一阶段会用 EPOLLOUT + outbuf 正确处理
                        }
                    } else if (r == 0) {
                        del_close(epfd, fd);
                        break;
                    } else {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) break;
                        perror("recv");
                        del_close(epfd, fd);
                        break;
                    }
                }
            }
        }
    }

    close(epfd);
    close(listen_fd);
    return 0;
}
```

你在这一关应该学会：
- epoll_wait 返回 `n`，只遍历 `0..n-1`
- listen_fd 可读 → accept 循环到 EAGAIN
- client_fd 可读 → recv 循环到 EAGAIN
- 断开/错误 → DEL + close

---

## 7. Step 4（进阶）：ET 模式怎么改（核心规则）

如果把 client 改成 ET：
- 注册时加 `EPOLLET`
- accept/recv/send 必须“做干净到 EAGAIN”（Step 3 已经按这个习惯写了读和 accept）

示例（把 add_epoll 那行改成）：
```c
add_epoll(epfd, cfd, EPOLLIN | EPOLLRDHUP | EPOLLET);
```

你会发现：只要你一直读到 `EAGAIN`，ET 就不会丢事件。

---

## 8. Step 5（真正工程关键）：写缓冲 + EPOLLOUT（为什么必须学）

上面的 echo 里我们“偷懒”了：
- `send()` 可能短写或 `EAGAIN`
- 正确做法：
  1) 每个连接维护 `outbuf`（未发送完的数据）
  2) outbuf 非空时，把该 fd 的监听事件改成 `EPOLLIN | EPOLLOUT ...`
  3) 收到 `EPOLLOUT` 时继续 flush，直到 outbuf 清空，再把 `EPOLLOUT` 去掉

这是所有高并发网络库的核心。

> 如果你确认已经理解 Step 3，我可以在下一条消息把 Step 5 写成“最小写缓冲版本”（仍然一步一步，不会突然给巨型代码）。

---

## 9. 常见坑清单（你写代码时对照）

1) **把 `n` 当成下标**：错。`n` 是数量，事件在 `evs[0..n-1]`
2) **忘了处理 EPOLLERR/EPOLLHUP/EPOLLRDHUP**：会出现“死连接挂着不清理”
3) **ET 模式没读到 EAGAIN**：经典卡死
4) **accept 不循环**：大量新连接到来时吞吐差
5) **写不做缓冲**：send EAGAIN/短写时会丢数据

---

## 10. 你下一步怎么学最顺

按顺序跑：
1) Step 1：stdin
2) Step 2：listen + accept
3) Step 3：echo（理解 accept/recv 循环与 close 清理）
4) Step 4：给 client 加 EPOLLET 看看还能不能跑（验证你的循环是否写对）
5) Step 5：实现 outbuf + EPOLLOUT（最关键进阶）

如果你把你想要的目标说清楚（例如“我要做一个可靠 echo server”“我要做聊天室”“我要做代理/转发”），我就按目标把 Step 5 的缓冲结构设计出来（含数据结构和最小可跑代码）。

