# Linux 分散/聚集 I/O（readv/writev）由浅入深

> 你可以把它当成：**一次系统调用读/写多段内存**。  
> - **分散（scatter）读**：`readv` 把数据从一个 `fd` 读出来，**分散地**放到多个 buffer。  
> - **聚集（gather）写**：`writev` 把多个 buffer 的数据 **聚集起来**，一次写到 `fd`。

---

## 0. 先从你已经会的 `read/write` 说起

平时你可能这样写网络协议或文件格式：

- 先读固定长度的 header（比如 8 字节）
- 再根据 header 里的长度读 body

用 `read` 可能会写成两次系统调用：

```c
read(fd, hdr, 8);
read(fd, body, body_len);
```

这会带来两个现实问题：

1) **系统调用次数多**：一次系统调用要从用户态切到内核态，有开销。

2) **拼包/拆包要手动处理**：`read` 可能短读，你不得不写循环和状态机。

`readv/writev` 的核心价值是：

- **一次系统调用**就把多段 buffer 读满/写出（注意：仍可能短读/短写，但能减少 syscall）。

---

## 1) API 速记：`struct iovec`

`readv/writev` 的参数不是一个 buffer，而是一组 buffer：

```c
#include <sys/uio.h>

struct iovec {
    void  *iov_base;  // 指向缓冲区起始地址
    size_t iov_len;   // 该缓冲区长度
};
```

函数签名：

```c
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
```

- 返回值：成功返回 **总共读/写的字节数**；失败返回 -1 并设置 `errno`。
- `iovcnt`：数组元素个数。

---

## 2) 最小可懂示例：聚集写 `writev`

场景：你要把 `"LEN="` + 数字 + `"\n"` 写入文件。

不用 `writev`：你可能 write 三次。

用 `writev`：一次写完。

```c
#include <sys/uio.h>
#include <unistd.h>
#include <string.h>

ssize_t write_len_line(int fd, int len) {
    char num[32];
    int n = snprintf(num, sizeof num, "%d", len);

    const char *p1 = "LEN=";
    const char *p3 = "\n";

    struct iovec iov[3] = {
        { .iov_base = (void*)p1, .iov_len = strlen(p1) },
        { .iov_base = (void*)num, .iov_len = (size_t)n },
        { .iov_base = (void*)p3, .iov_len = 1 },
    };

    return writev(fd, iov, 3);
}
```

你现在脑中应该形成一个画面：

- `iov[0]` 指向 `"LEN="`
- `iov[1]` 指向数字字符串
- `iov[2]` 指向换行

内核按顺序把这三段拼起来写出去。

---

## 3) 分散读 `readv`：一次把“头”和“体”分别读到不同 buffer

场景：协议 header 固定 8 字节（比如 magic + body_len），body 紧随其后。

```c
#include <sys/uio.h>
#include <unistd.h>

// 读：8字节头 + body_len字节体
ssize_t read_packet(int fd, void *hdr8, void *body, size_t body_len) {
    struct iovec iov[2] = {
        { .iov_base = hdr8, .iov_len = 8 },
        { .iov_base = body, .iov_len = body_len },
    };
    return readv(fd, iov, 2);
}
```

好处：

- 内核把前 8 字节放进 `hdr8`
- 后面的数据放进 `body`
- 你不需要先读进一个大缓冲再 memcpy 拆开。

---

## 4) 关键难点：`readv/writev` 仍可能短读/短写（必须会处理）

很多人卡在这里：

> “我一次调用 `readv`，是不是保证把所有 iov 都填满？”

**不保证。**

原因和 `read` 一样：

- 文件/管道/socket 都可能返回更少字节（比如非阻塞、信号中断、网络缓冲不足）。

所以正确姿势是：**写一个 `readv_full / writev_full`**（或者至少懂其思路）。

### 4.1 `writev` 的短写处理思路（重点）

`writev` 返回 `n` 表示总共写出 `n` 字节。

如果 `n` 小于所有 iov_len 之和，你要：

- 从 iov[0] 开始扣掉 `n`
- 扣完一个 iov 就换下一个
- 在中间某个 iov 里停下，把该 iov 的 base 前移、len 缩短
- 然后继续 writev

下面给你一个**可直接复用**的版本（逻辑写全，但仍保持可读性）：

```c
#include <sys/uio.h>
#include <unistd.h>
#include <errno.h>

static size_t iov_total(const struct iovec *iov, int cnt) {
    size_t t = 0;
    for (int i = 0; i < cnt; i++) t += iov[i].iov_len;
    return t;
}

// 尝试把 iov 全部写完；成功返回总字节数，失败返回 -1
ssize_t writev_full(int fd, struct iovec *iov, int cnt) {
    size_t total = iov_total(iov, cnt);
    size_t done = 0;

    while (done < total) {
        ssize_t n = writev(fd, iov, cnt);
        if (n > 0) {
            done += (size_t)n;

            // 消耗掉前面的 iov
            ssize_t left = n;
            while (cnt > 0 && left >= (ssize_t)iov[0].iov_len) {
                left -= (ssize_t)iov[0].iov_len;
                iov++; cnt--; // 整个 iov[0] 已写完，丢掉它
            }
            if (cnt > 0 && left > 0) {
                // 在当前 iov 内部部分写完：前移 base，缩短 len
                iov[0].iov_base = (char*)iov[0].iov_base + left;
                iov[0].iov_len  -= (size_t)left;
            }
            continue;
        }
        if (n < 0 && errno == EINTR) continue; // 被信号打断，重试
        return -1;
    }
    return (ssize_t)done;
}
```

> 读：`readv` 的“短读”处理也类似，但要注意 `0` 表示 EOF（对 socket 可能表示对端关闭）。

处理readv短读：
```C
static size_t iov_total(const struct iovec *iov, int cnt) {
    size_t t = 0;
    for (int i = 0; i < cnt; i++) t += iov[i].iov_len;
    return t;
}
ssize_t readv_full(int fd,struct iovec * iov,int cnt){
	size_t total=iov_total(iov,cnt);
	size_t done=0;
	while(done<total){
		ssize_t n = readv(fd,iov,cnt);
		if(n>0){
			done+=(size_t)n;
			ssize_t left = n;
			while(cnt>0 && left>=(szie_t)iov[0].iov_len){
				--cnt;
				left-=(ssize_t)iov[0].iov_len;
				iov++;
			}
			if(cnt>0 && left>0){
				iov[0].iov_len -= (size_t)left;
				iov[0].iov_base = (char*)iov[0].iov_base+(size_t)left;
				/*__`iov_base` 是 `void*`，不能直接 `+=`（标准 C 不允许 void_ 指针算术）_*  要转成 `char*` 做字节偏移。*/
			}
			continue;
		}
		if(n<0 && errno==EINTR) continue;
		if (n == 0) {
            // EOF：文件读到末尾；对 TCP 来说是对端关闭连接
            break;
        }
		return -1;
	}
	
	return (ssize_t)done;
}
```


## 优化count值 
 在向量I/O操作中，Linux内核必须分配内部数据结构来表示每个段（segment）。一般来说，是基于count的大小动态分配进行的。然而，为了优化，如果count值足够小，内核会在栈上创建一个很小的段数组，通过避免动态分配段内存，从而获得性能上的一些提升。count的阈值一般设置为8，因此如果count值小于或等于8时，向量I/O操作会以一种高效的方式，在进程的内核栈中运行。 
 大多数情况下，无法选择在指定的向量I/O 操作中一次同时传递多少个段。当你认为可以试用一个较小值时，选择8或更小的值肯定会得到性能的提升.
 
---

## 5) `readv/writev` 和 `recvmsg/sendmsg` 的关系

`readv/writev` 本质上是 `recvmsg/sendmsg` 的简化形式：

- `sendmsg`/`recvmsg` 除了 iovec，还能带：
  - 控制信息（ancillary data），比如**传递文件描述符**（`SCM_RIGHTS`）
  - 获取对端地址（UDP）
  - flags（如 `MSG_DONTWAIT`）

所以：

- 只需要多 buffer：`readv/writev`
- 需要更强能力：`recvmsg/sendmsg`

---

## 6) 性能视角：它到底快在哪？

### 6.1 少 syscalls（最直接）

- `write` 3 次 → 3 次 user↔kernel 切换
- `writev` 1 次 → 1 次切换

当你输出很多小块数据（header/body/多个字段）时，收益明显。

### 6.2 少 memcpy（取决于你原来的写法）

如果你以前做法是：

- 把 header 和 body 先 memcpy 拼成一个大 buffer
- 再 write 一次

那么 `writev` 可以让你：

- 直接指向原来的 header/body buffer
- 不用额外拼接

（注意：内核仍会把数据拷贝到内核缓冲/页缓存/网卡缓冲，`writev` 不是“零拷贝”。它省的是你用户态的拼接拷贝。）

---

## 7) 常见坑（非常实用）

1) **短读/短写没处理**：网络 I/O 下经常出错。

2) **iov 指针生命周期**：`writev` 不会异步保存你的指针；调用返回前必须有效（通常栈上没问题）。

3) **iovec 数量上限**：由 `IOV_MAX` 限制（`sysconf(_SC_IOV_MAX)` 可查）。

4) **非阻塞 fd**：`EAGAIN/EWOULDBLOCK` 要配合 epoll/select 处理。

5) **TCP 不是消息边界协议**：`readv` 读到的字节数不等于“一整包”，仍要靠协议长度字段组包。

---

## 8) 什么时候最该用？（你一眼能选型）

- **网络协议栈/网关/代理**：header+body 多段拼接
- **日志系统**：时间戳/级别/正文分段输出
- **高性能文件格式写入**：多个字段块一次落盘
- **RPC/序列化框架**：避免每次都拼 buffer

不太适合：

- 你本来就只有一段连续 buffer：直接 `write` 就够了。

---

## 9) 给你一张“心智模型”总结

- `write(fd, buf, len)`：一次写一个 buf
- `writev(fd, [buf1, buf2, buf3])`：一次写多个 buf（按顺序拼起来写）
- `read(fd, buf, len)`：一次读到一个 buf
- `readv(fd, [buf1, buf2])`：一次读，先填 buf1，再填 buf2

核心注意：**返回值永远是“总字节数”，而不是“填满了几个 iov”。**

---

## 10) 你下一步该学什么（承上启下）

如果你想把这块真正用到“工程级别”，下一步通常是：

1) `readv/writev + epoll`：非阻塞高并发
2) `sendmsg/recvmsg`：FD 传递、UDP 元信息
3) `splice/sendfile`：文件到 socket 的更少拷贝路径
4) `io_uring`：更现代的异步 I/O（把 syscall 进一步“批处理”）

---

如果你告诉我你要用的场景是：
- **文件 I/O**（写日志/写数据文件）
- 还是 **网络 I/O**（TCP 组包/拆包）

我可以基于你的场景给一个“最小可运行”的 demo（带 Makefile 和测试方法），让你跑起来立刻看到短读/短写的现象和正确处理方式。

