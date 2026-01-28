# Linux 系统编程：IO 多路复用（poll）逐步拆解

你说得对：直接上“大而全”示例会让人看不清结构。下面用**循序渐进**的方式，从 1 个 fd 开始，逐步加到“监听 socket + 多客户端 echo”。每一步都是**独立可编译运行**的小程序，你可以按顺序跑。

---

## 0. 先建立直觉：poll 在做什么

- `poll()` 不会帮你读写数据，它只做一件事：
  - **阻塞等待**（或超时）直到你关心的一组 fd 里有**就绪事件**发生。
- “就绪”是**readiness**：
  - `POLLIN`：读不会阻塞（对 socket 还可能意味着对端关闭，`recv` 返回 0）
  - `POLLOUT`：写不会阻塞（发送缓冲区有空间）
  - `POLLERR/POLLHUP/POLLNVAL`：错误/挂起/无效

你需要记住一句话：
> poll 只告诉你“哪些 fd 该去处理”，真正的 `read/recv/accept` 还是要你自己调用。

**函数原型**
```C
#include <poll.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

---

## 1. poll 的核心数据结构：struct pollfd

```c
#include <poll.h>

struct pollfd {
    int   fd;      // 你要监控的文件描述符
    short events;  // 你希望监控的事件（输入）
    short revents; // poll 返回后告诉你实际发生了什么（输出）
};
```

- 你填：`fd` + `events`
- `poll()` 返回后读：`revents`

常用事件：
- `POLLIN`：可读
- `POLLOUT`：可写
- `POLLERR`：错误
- `POLLHUP`：挂起（常见：对端断开）
- `POLLNVAL`：fd 无效（比如你 close 了但没从数组移除）

---

## 2. 第一步：只监控 stdin（最小可跑）

目标：按回车输入一行，`poll()` 检测到 stdin 可读，然后读出来打印。

### step1_poll_stdin.c
```c
#include <poll.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    struct pollfd pfd;
    pfd.fd = STDIN_FILENO;      // 0
    pfd.events = POLLIN;        // 关心“可读”

    printf("Type something and press Enter...\n");

    for (;;) {
        int ret = poll(&pfd, 1, -1); // 只监控 1 个 fd，永久等待
        if (ret < 0) {
            perror("poll");
            return 1;
        }

        if (pfd.revents & POLLIN) {
            char buf[256];
            ssize_t n = read(STDIN_FILENO, buf, sizeof(buf) - 1);
            if (n <= 0) {
                printf("stdin closed.\n");
                return 0;
            }
            buf[n] = '\0';
            printf("You typed: %s", buf);
        }

        // 这一行不是必须，但能帮助你理解“revents 是输出，每轮会重置”
        pfd.revents = 0;
    }
}
```

编译运行：
```bash
gcc step1_poll_stdin.c -O2 -Wall -Wextra -o step1
./step1
```

你要观察到的：
- 没输入时程序“卡住”，说明 `poll()` 在等事件
- 一回车，`poll()` 返回，`revents` 里出现 `POLLIN`

---

## 3. 第二步：监控两个 fd（stdin + 监听 socket）

目标：
- `poll()` 同时盯住：
  - `stdin`（你输入命令）
  - `listen_fd`（有新连接就 accept）

先不处理客户端读写，只要 accept 后打印“来了个连接”。

### step2_poll_listen_and_stdin.c
```c
#include <arpa/inet.h>
#include <netinet/in.h>
#include <poll.h>
#include <stdio.h>
#include <string.h>
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
    if (listen_fd < 0) {
        perror("listen");
        return 1;
    }

    printf("Listening on %d. Try: nc 127.0.0.1 %d\n", port, port);
    printf("Type 'quit' to exit.\n");

    struct pollfd pfds[2];
    pfds[0].fd = listen_fd;
    pfds[0].events = POLLIN;

    pfds[1].fd = STDIN_FILENO;
    pfds[1].events = POLLIN;

    for (;;) {
        int ret = poll(pfds, 2, -1);
        if (ret < 0) {
            perror("poll");
            return 1;
        }

        // 1) 有新连接
        if (pfds[0].revents & POLLIN) {
            int cfd = accept(listen_fd, NULL, NULL);
            if (cfd >= 0) {
                printf("New client accepted: fd=%d\n", cfd);
                close(cfd); // 先不处理客户端，直接关掉
            }
        }

        // 2) stdin
        if (pfds[1].revents & POLLIN) {
            char line[128];
            if (!fgets(line, sizeof(line), stdin)) break;
            if (strcmp(line, "quit\n") == 0) break;
            printf("stdin: %s", line);
        }

        pfds[0].revents = pfds[1].revents = 0;
    }

    close(listen_fd);
    return 0;
}
```

编译运行：
```bash
gcc step2_poll_listen_and_stdin.c -O2 -Wall -Wextra -o step2
./step2
```

你要观察到的：
- `nc 127.0.0.1 8080` 连接上来时，`listen_fd` 的 `POLLIN` 触发
- 你输入内容时，stdin 的 `POLLIN` 触发

---

## 4. 第三步：把“客户端 fd”加入 poll 数组（动态增长）

这一关是关键：
- `poll()` 的输入是一个数组 `pfds[]`
- 新连接来了，你要把 `accept()` 得到的 `client_fd` 也塞进数组
- 客户端断开，要把它从数组里移除（否则会 `POLLNVAL`）

我们用一个简单但实用的删除策略：
- 删除下标 `i` 时：用数组最后一个元素覆盖 `i`，然后 `nfds--`
- 好处：O(1) 删除

---

## 5. 第四步：完整的 poll 多客户端 Echo（仍保持“易懂优先”）

说明：
- 这是“教学版正确骨架”，能跑、逻辑清晰
- 这里写操作用 `send()`，为了教学先不引入复杂的写缓冲模型
  - 大流量/非阻塞时会涉及 `POLLOUT` + 缓冲队列（下一阶段再学）

### step3_poll_echo.c
```c
#define _GNU_SOURCE
#include <arpa/inet.h>
#include <errno.h>
#include <netinet/in.h>
#include <poll.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
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

static int send_all(int fd, const char* buf, size_t len) {
    size_t sent = 0;
    while (sent < len) {
        ssize_t n = send(fd, buf + sent, len - sent,
#ifdef MSG_NOSIGNAL
                         MSG_NOSIGNAL
#else
                         0
#endif
        );
        if (n < 0) {
            if (errno == EINTR) continue;
            return -1;
        }
        sent += (size_t)n;
    }
    return 0;
}

static void remove_fd(struct pollfd* pfds, nfds_t* nfds, nfds_t idx) {
    close(pfds[idx].fd);
    pfds[idx] = pfds[*nfds - 1];
    (*nfds)--;
}//用最后一个元素覆盖idx，再缩窄

int main(int argc, char** argv) {
    int port = 8080;
    if (argc >= 2) port = atoi(argv[1]);

    signal(SIGPIPE, SIG_IGN);

    int listen_fd = create_listen(port);
    if (listen_fd < 0) {
        perror("listen");
        return 1;
    }

    printf("Listening on %d\n", port);
    printf("Try: nc 127.0.0.1 %d\n", port);
    printf("Type in server console to broadcast to all clients. Type 'quit' to exit.\n");

    const int MAX = 4096;
    struct pollfd pfds[MAX];
    nfds_t nfds = 0;

    // pfds[0] = listen_fd
    pfds[nfds++] = (struct pollfd){ .fd = listen_fd, .events = POLLIN, .revents = 0 };
    // pfds[1] = stdin
    pfds[nfds++] = (struct pollfd){ .fd = STDIN_FILENO, .events = POLLIN, .revents = 0 };

    for (;;) {
        int ret = poll(pfds, nfds, -1);
        if (ret < 0) {
            if (errno == EINTR) continue;
            perror("poll");
            break;
        }

        // A) stdin：读取一行并广播
        if (pfds[1].revents & POLLIN) {
            char line[4096];
            if (!fgets(line, sizeof(line), stdin)) {
                printf("stdin EOF, exit\n");
                break;
            }
            if (strcmp(line, "quit\n") == 0 || strcmp(line, "quit\r\n") == 0) {
                printf("quit\n");
                break;
            }
            for (nfds_t i = 2; i < nfds; ++i) {
                if (send_all(pfds[i].fd, line, strlen(line)) < 0) {
                    remove_fd(pfds, &nfds, i);
                    i--; // idx 被最后一个覆盖了，需重新检查
                }
            }
        }

        // B) listen_fd：有新连接
        if (pfds[0].revents & POLLIN) {
            int cfd = accept(listen_fd, NULL, NULL);
            if (cfd < 0) {
                perror("accept");
            } else if (nfds >= (nfds_t)MAX) {
                printf("Too many clients\n");
                close(cfd);
            } else {
                printf("New client fd=%d\n", cfd);
                pfds[nfds++] = (struct pollfd){ .fd = cfd, .events = POLLIN, .revents = 0 };
            }
        }

        // C) 客户端：可读则 recv 并 echo
        for (nfds_t i = 2; i < nfds; ++i) {
            short re = pfds[i].revents;

            if (re & (POLLERR | POLLHUP | POLLNVAL)) {
                remove_fd(pfds, &nfds, i);
                i--;
                continue;
            }

            if (re & POLLIN) {
                char buf[4096];
                ssize_t n = recv(pfds[i].fd, buf, sizeof(buf), 0);
                if (n == 0) {
                    printf("Client fd=%d closed\n", pfds[i].fd);
                    remove_fd(pfds, &nfds, i);
                    i--;
                } else if (n < 0) {
                    perror("recv");
                    remove_fd(pfds, &nfds, i);
                    i--;
                } else {
                    if (send_all(pfds[i].fd, buf, (size_t)n) < 0) {
                        remove_fd(pfds, &nfds, i);
                        i--;
                    }
                }
            }
        }

        // 教学清晰起见，手动清 0（不是必须）
        for (nfds_t i = 0; i < nfds; ++i) pfds[i].revents = 0;
    }

    // 清理
    for (nfds_t i = 0; i < nfds; ++i) {
        if (pfds[i].fd != STDIN_FILENO) close(pfds[i].fd);
    }
    return 0;
}
```

编译运行：
```bash
gcc step3_poll_echo.c -O2 -Wall -Wextra -o step3
./step3 8080
```

---

## 6. 这一版“刻意没讲”的两件事（下一阶段你再学）

你现在已经掌握了 poll 的主循环骨架：
- `pfds[]` 放监听/stdin/客户端
- `poll()` 返回后看 `revents`
- 新连接 `accept()` 后加入数组
- 断开/错误从数组移除

下一阶段（更工程化）再引入：

1) **非阻塞 I/O**（`fcntl(O_NONBLOCK)`）：
   - 读：循环 `recv` 直到 `EAGAIN`
   - accept：循环 `accept` 直到 `EAGAIN`

2) **写缓冲 + POLLOUT**：
   - send 可能短写/阻塞
   - 需要每个连接维护 outbuf
   - outbuf 非空时，给它加 `POLLOUT`，可写时继续 flush

---

## 7. 你现在该怎么练（最有效）

按顺序做 3 个练习：
1) step1：stdin 轮询 + timeout（把 timeout 改成 2000ms，打印“tick”）
2) step2：监听 listen_fd + stdin（理解两个 fd 同时被 poll）
3) step3：多客户端 echo（重点理解：数组增删与 i--）

如果你把 step3 跑起来了，下一步我可以带你把它升级到“非阻塞 + POLLOUT 写缓冲”的真正网络框架。

