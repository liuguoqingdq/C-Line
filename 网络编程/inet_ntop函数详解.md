# inet_ntop 函数详解

## 1. 作用概述

`inet_ntop` 的作用是：

> **把二进制形式（网络字节序）的 IP 地址转换成人类可读的字符串形式。**

- `n` = network（网络）
- `t` = to（到）
- `p` = presentation（呈现/展示）

所以 `inet_ntop` 就是 **Network → Presentation** 的转换函数。

典型用途：

- 你从 `sockaddr_in` / `sockaddr_in6` 结构体里拿到了 IP（二进制形式），想打印到日志里；
- 调试时打印客户端 / 服务器端的 IP 地址；
- 把 `accept()` 拿到的对端地址转换成字符串。

---

## 2. 函数原型与参数

```c
#include <arpa/inet.h>

const char *inet_ntop(int af,
                      const void *src,
                      char *dst,
                      socklen_t size);
```

### 2.1 参数含义

1. `af` —— 地址族（address family）

   - `AF_INET`  ：IPv4
   - `AF_INET6` ：IPv6

2. `src` —— 源地址（二进制形式）

   - `AF_INET`  时：`src` 指向 `struct in_addr`，通常是 `&addr.sin_addr`
   - `AF_INET6` 时：`src` 指向 `struct in6_addr`，通常是 `&addr6.sin6_addr`

   注意：这里存放的是 **网络字节序** 的 IP 地址。

3. `dst` —— 目标缓冲区（字符串）

   - 你提供的 `char` 数组，用来存放转换后的 IP 字符串
   - 例如：

     ```c
     char ipstr[INET_ADDRSTRLEN];   // IPv4
     char ip6str[INET6_ADDRSTRLEN]; // IPv6
     ```

4. `size` —— `dst` 缓冲区的大小

   - 对 IPv4：应至少为 `INET_ADDRSTRLEN`（一般为 16）
   - 对 IPv6：应至少为 `INET6_ADDRSTRLEN`（一般为 46）

### 2.2 返回值

- 成功：返回 `dst`（同一个指针，便于链式使用）
- 失败：返回 `NULL`，并设置 `errno`，如：
  - `EAFNOSUPPORT`：`af` 不是支持的地址族
  - `ENOSPC`：缓冲区 `dst` 太小

---

## 3. IPv4 使用示例

```c
#include <stdio.h>
#include <arpa/inet.h>

int main(void) {
    struct sockaddr_in addr;

    addr.sin_family = AF_INET;
    // 将字符串形式的 IP 转成二进制形式，存入 addr.sin_addr
    inet_pton(AF_INET, "192.168.1.100", &addr.sin_addr);

    char ipstr[INET_ADDRSTRLEN];

    const char *ret = inet_ntop(AF_INET, &addr.sin_addr,
                                ipstr, sizeof(ipstr));
    if (ret == NULL) {
        perror("inet_ntop");
        return 1;
    }

    printf("IP: %s\n", ipstr);
    return 0;
}
```

要点：

- `af` 使用 `AF_INET`；
- `src` 是 `&addr.sin_addr`，类型是 `struct in_addr *`；
- `dst` 是字符数组首地址 `ipstr`；
- `size` 用 `sizeof(ipstr)` 或 `INET_ADDRSTRLEN`。

---

## 4. 在实际网络编程中的典型用法

### 4.1 打印 `accept()` 得到的客户端 IP

```c
int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
// ... bind、listen 省略

struct sockaddr_in cliaddr;
socklen_t clilen = sizeof(cliaddr);

int connfd = accept(listen_fd, (struct sockaddr *)&cliaddr, &clilen);

char ip[INET_ADDRSTRLEN];
if (inet_ntop(AF_INET, &cliaddr.sin_addr, ip, sizeof(ip)) == NULL) {
    perror("inet_ntop");
} else {
    printf("new connection from %s:%u\n", ip, ntohs(cliaddr.sin_port));
}
```

- `cliaddr.sin_addr` 是客户端 IP（二进制）；
- 用 `inet_ntop` 转成字符串方便打印；
- 端口打印时记得 `ntohs(cliaddr.sin_port)` 转回主机字节序。

### 4.2 日志与调试

- 记录每一次连接的 IP；
- debug：确认服务是否只收到了预期网段的访问；
- 结合 `getsockname` / `getpeername` 打印本端和对端地址。

---

## 5. 与 inet_pton / inet_ntoa 的关系

### 5.1 `inet_pton` —— 配套的“反向函数”

```c
int inet_pton(int af, const char *src, void *dst);
```

- `p` = presentation → `n` = network
- 即：**字符串（如 "127.0.0.1"） → 二进制（网络字节序）**

通常成对使用：

- 用 `inet_pton` 解析配置文件或命令行传入的 IP；
- 用 `inet_ntop` 把从 socket 中得到的 IP 打印/记录出去。

### 5.2 `inet_ntoa` —— 旧接口（不推荐）

```c
char *inet_ntoa(struct in_addr in);
```

缺点：

1. 只支持 IPv4；
2. 返回的是指向 **内部静态缓冲区** 的指针：
   - 多线程下不安全；
   - 多次调用会覆盖之前的结果；
3. 不可指定缓冲区大小，易出问题。

因此，在新代码里应优先使用：

- `inet_pton` / `inet_ntop` 组合

---

## 6. 常见错误与坑

### 6.1 传错 `src` 指针

错误示例：

```c
inet_ntop(AF_INET, &addr, ip, sizeof(ip)); // ❌ src 应该是 &addr.sin_addr
```

正确做法：

```c
inet_ntop(AF_INET, &addr.sin_addr, ip, sizeof(ip)); // ✅
```

### 6.2 忘记检查返回值

```c
inet_ntop(AF_INET, &addr.sin_addr, ip, sizeof(ip));
printf("%s\n", ip); // ❌ 若转换失败，ip 内容是不确定的
```

正确：

```c
if (inet_ntop(AF_INET, &addr.sin_addr, ip, sizeof(ip)) == NULL) {
    perror("inet_ntop");
} else {
    printf("%s\n", ip);
}
```

### 6.3 缓冲区过小

- IPv4 至少 `INET_ADDRSTRLEN`
- IPv6 至少 `INET6_ADDRSTRLEN`

缓冲区太小时：

- `inet_ntop` 返回 `NULL`，`errno = ENOSPC`

### 6.4 `af` 与地址类型不匹配

```c
struct sockaddr_in6 addr6;
// ... addr6 填的是 IPv6
inet_ntop(AF_INET, &addr6.sin6_addr, ip, sizeof(ip)); // ❌ af 错了
```

必须保证：

- 地址族 `af` 与 `src` 指向的结构体类型匹配。

---

## 7. 小练习

1. 写一个程序：
   - 从命令行读取一个 IP 字符串（IPv4 或 IPv6）；
   - 用 `inet_pton` 转成二进制；
   - 再用 `inet_ntop` 转回字符串打印；
   - 比较输入和输出是否一致。

2. 在你的 TCP 服务器中：
   - 使用 `accept` 得到客户端地址；
   - 用 `inet_ntop` + `ntohs` 把 `IP:port` 打印出来；
   - 多开几个客户端连接测试，观察输出。

通过这些练习，你会把 `inet_ntop` 和整体“地址结构处理流程”串得非常熟。

