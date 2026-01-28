### I/O函数
#### 一、`recv()` 是干什么的？（一句话）

> **`recv()` 用于从 socket 的接收缓冲区中读取数据，把数据从内核态拷贝到用户态缓冲区。**

📌 注意：

- 它**不是直接和网卡 I/O**
    
- 而是**读取内核 socket 接收缓冲区**
    

---

#### 二、函数原型
```C++
#include <sys/types.h>
#include <sys/socket.h>

ssize_t recv(int sockfd,
             void *buf,
             size_t len,
             int flags);
```
##### 1️⃣ `sockfd` —— 套接字描述符

- 必须是：
    
    - 已连接的 TCP socket
        
    - 或 UDP socket（语义略不同）
        
- 一般来自：

    ```C++
    accept();
    socket();
    ```
##### 2️⃣ `buf` —— 用户缓冲区

```C++
void* buf;
```

- 接收数据的目标地址
    
- 是普通用户内存
    
- 内核会把数据 **拷贝** 到这里
    

⚠️ **必须保证 buf 至少有 len 字节空间**
##### 3️⃣ `len` —— 最多接收的字节数

```C++
size_t len;
```

- `recv()` **最多**读取 `len` 字节
    
- 实际返回值：
    
    - 可能 **小于 len**
        
    - 这就是“不完全读”
##### 4️⃣ `flags` —— 接收控制标志（高级）

### 常见取值

|flag|作用|
|---|---|
|`0`|默认行为|
|`MSG_PEEK`|偷看数据，不移除|
|`MSG_WAITALL`|尽量读满|
|`MSG_DONTWAIT`|非阻塞接收|
|`MSG_OOB`|带外数据|
### `MSG_PEEK`（非常有用）

```C++
recv(fd, buf, len, MSG_PEEK);
```

- 读取数据
    
- **但不从接收缓冲区移除**
    
- 常用于：
    
    - 解析协议头
        
    - 查看长度字段
        

---

### 🔹 `MSG_WAITALL`（了解）

- 尽量等到 `len` 字节
    
- 但：
    
    - 对端关闭
        
    - 信号中断
        
    - 出错
        
- 都可能提前返回
    

⚠️ **不建议依赖**


--------
### 1) `readv()` 是干什么的？

> **一次系统调用把数据读到多个不连续的用户缓冲区里。**  
> 这叫 **分散读（scatter read）**。

对比：

- `read()`：读到 **一个** 连续缓冲区
    
- `readv()`：读到 **多个** 缓冲区（数组）
    

它常用于：**协议解析**（先读固定长度头，再读变长体）、**减少拷贝**、**减少系统调用次数**。
## 2) 函数原型与头文件

```C++
#include <sys/uio.h>
#include <unistd.h>

ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
```

- `fd`：文件描述符（文件、管道、socket 都可以）
    
- `iov`：`iovec` 数组，描述多个缓冲区
    
- `iovcnt`：数组元素个数
    

`iovec` 结构体：

```C++
struct iovec {
    void  *iov_base;  // 缓冲区起始地址
    size_t iov_len;   // 缓冲区长度
};
```
-----
#### 3) `readv()` 的行为规则（非常重要）

### 3.1 数据如何分配到多个缓冲区？

内核会把读到的数据按顺序填充：

- 先填满 `iov[0]`
    
- 再填 `iov[1]`
    
- …
    
- 直到数据用完或所有缓冲区填满
    

示意：

```C++
readv(fd, [bufA(4), bufB(8), bufC(16)])

读到 10 字节 →
bufA 填 4 字节
bufB 填 6 字节
bufC 不动
返回 10
```

### 3.2 返回值含义

`ssize_t ret = readv(...);`

- `ret > 0`：实际读到的总字节数（跨多个缓冲区的总和）
    
- `ret == 0`：EOF（对端关闭/文件结束）
    
- `ret == -1`：出错（看 `errno`）
    

### 3.3 会“不完全读”吗？

**会。**  
和 `read()`/`recv()` 一样：

- 阻塞 fd：只保证“至少有 1 字节/或 EOF/或错误”就返回
    
- 非阻塞 fd：没数据返回 `-1` 且 `errno=EAGAIN/EWOULDBLOCK`
    

**TCP 上尤其常见**：一次 `readv()` 不保证把你期望的“头+体”都读满。

---
## 4) 典型使用场景：协议“头部 + 正文”

例如协议格式：

`| 4字节长度(网络序) | N字节payload |`

你希望一次把头读到 `hdr`，正文读到 `payload` 的某一段或缓冲区里。
```C++
uint32_t netlen;
char body[4096];

struct iovec iov[2];
iov[0].iov_base = &netlen;
iov[0].iov_len  = sizeof(netlen);
iov[1].iov_base = body;
iov[1].iov_len  = sizeof(body);

ssize_t n = readv(fd, iov, 2);
```
- 如果 `n >= 4`：说明头已经拿到（可能 body 也拿到一部分）
    
- 如果 `n < 4`：头都不完整，必须继续读（这时你要维护“已读偏移”）

> 工程上：**`readv()` 常用于“先拿到头+尽可能多的 body”**，然后再用循环补齐剩余部分


--------
#### 使用send()函数
send()函数用于发送数据
##### 1) `send()` 是干什么的？

> **把用户缓冲区的数据拷贝到内核的 socket 发送缓冲区（send buffer），并触发 TCP/UDP 的发送流程。**

关键点：

- `send()` **不等于**“数据已经到对方”  
    它通常只表示：**数据已经进入内核发送队列**（或发送了一部分）。
    
- TCP 是**字节流**：一次 `send()` 不对应一次 `recv()`。

---
#### 2) 函数原型与头文件

```C++
#include <sys/types.h>
#include <sys/socket.h>

ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

参数：

- `sockfd`：socket 描述符（TCP 需已 `connect`，或 `accept` 得到的连接）
    
- `buf`：要发送的数据
    
- `len`：要发送的长度（最大尝试发送的字节数）
    
- `flags`：控制行为的标志位（常用为 0）
---
#### 3) 返回值含义（必须牢记）

`ssize_t n = send(...);`

- `n > 0`：本次**实际写入内核发送缓冲区**的字节数（可能 `< len`）
    
- `n == 0`：很少见；对 TCP 来说通常表示“发送 0 字节”（你传 `len=0`）
    
- `n == -1`：出错，看 `errno`
    

---

## 4) `send()` 会“不完全写（短写）”吗？

**会，而且很常见（尤其是非阻塞、高并发、发送大数据时）。**

### 为什么会短写？

- 内核 socket 发送缓冲区空间有限
    
- TCP 拥塞控制、对端窗口、链路状况等导致发送队列积压
    
- 非阻塞 socket 下，没空间就不会等你
    

### 典型现象

你 `send(..., len=10000)`，返回 `4096` 或 `8192`：

- 表示只写进去一部分
    
- 你必须**继续发送剩余数据**

------
#### 5) 阻塞 / 非阻塞下的行为差异

### 5.1 阻塞 socket（默认）

- 发送缓冲区满时：
    
    - `send()` 可能阻塞，直到有空间或出错
        
- 但仍可能短写（例如被信号中断、底层情况等）
    

### 5.2 非阻塞 socket（常配合 epoll）

- 发送缓冲区满时：
    
    - 立即返回 `-1`
        
    - `errno = EAGAIN` 或 `EWOULDBLOCK`
        
- 也可能短写：先写入一部分，然后没空间了

-----
## 6) `flags` 常用选项（重点）

多数情况下你写 `0` 就行，但下面几个很实用：

### `MSG_NOSIGNAL`（Linux 常用）

- 避免对端已关闭时触发 `SIGPIPE` 让进程直接退出
    
- 用法：
    

```C++
send(fd, buf, len, MSG_NOSIGNAL);
```

注意：macOS/BSD 没这个 flag，通常用 `setsockopt(SO_NOSIGPIPE)` 或忽略 SIGPIPE。

### `MSG_DONTWAIT`

- 本次发送使用非阻塞语义（即便 fd 本身是阻塞的）
    

```C++
send(fd, buf, len, MSG_DONTWAIT);
```
### `MSG_MORE`（Linux）

- 提示内核“后面还会有数据”，可能减少小包发送（与分段/聚合相关）
    
- 用于精细优化（不常用）
    

---

## 7) `send()` 常见错误码（高频）

- `EPIPE`：对端关闭连接，你还在发
    
    - 可能还伴随 `SIGPIPE`（默认会杀死进程）
        
- `ECONNRESET`：连接被对端复位
    
- `EAGAIN/EWOULDBLOCK`：非阻塞下发送缓冲区满
    
- `EINTR`：被信号中断（可重试）
    
- `ENOTCONN`：TCP socket 未连接就 send
    
- `EMSGSIZE`（UDP 常见）：报文太大
    

---

## 8) TCP vs UDP 的 `send()` 语义差异

### TCP（SOCK_STREAM）

- `send()` 发送的是**字节流**
    
- 可以分多次发送，接收端需要自己按协议“组帧”
    
- 短写/粘包/拆包是常态
    

### UDP（SOCK_DGRAM）

- `send()`（或 `sendto()`）对应一个**数据报**
    
- 要么整包进内核队列，要么失败（一般不会“写一半”）
    
- 超过 MTU 可能导致分片或失败（取决于系统/设置），常见 `EMSGSIZE`

---

#### writev()函数
# 一、`writev()` 是干嘛的？（一句话）

> **`writev()` 用来把“多个不连续的缓冲区”，一次性写到文件描述符中。**

专业术语叫：  
👉 **聚集写（gather write）**

---

# 二、为什么需要 `writev()`？

### 一、 `writev()` 是干什么的？

**一句话定义：** `writev()` 用于将**多个不连续的缓冲区**（Multiple Buffers）中的数据，通过**一次系统调用**连续地写入到文件描述符中。

- **专业术语：** **聚集写 (Gather Write)**
    

---

### 二、 核心价值：为什么不直接用 `write()`？

如果你需要发送 `[Header][Body]` 结构的数据：

|**方案**|**操作步骤**|**缺点**|
|---|---|---|
|**传统 write()**|`memcpy` 到临时大 buffer -> `write()`|**多一次内存拷贝、多一次内存分配**，性能开销大|
|**连续 write()**|`write(header)` -> `write(body)`|**两次系统调用**，在高并发下上下文切换开销翻倍|
|**writev()**|`writev(fd, {header, body}, 2)`|**❌ 不拼接、❌ 不拷贝、✅ 一次系统调用**|

> **结论：** `writev()` 的核心价值在于**减少系统调用次数**并实现**应用层零拷贝**。

---

### 三、 函数原型与核心数据结构

#### 1. 头文件与原型

```C++
#include <sys/uio.h>
#include <unistd.h>
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
```

#### 2. iovec 结构体（数组元素）

这是内核读取数据的“索引卡片”：
```C++
struct iovec {
    void  *iov_base;   // 缓冲区的起始地址
    size_t iov_len;    // 缓冲区的字节长度
};
```

---

### 四、 写入规则与行为特征

1. **逻辑顺序：** 内核严格按照索引顺序写入：`iov[0]` → `iov[1]` → `iov[2]`。
    
2. **原子性说明：** 在普通文件中，`writev()` 通常是原子的；但在 **Socket 编程** 中，多个进程同时写同一个 FD，数据可能会交织，且容易发生“短写”。
    
3. **返回值处理：**
    

|**返回值 (ret)**|**含义**|
|---|---|
|**ret > 0**|实际写入的总字节数（可能小于请求的总数）|
|**ret == 0**|写入长度为 0|
|**ret == -1**|出错，需检查 `errno`（如 `EAGAIN`）|

---

### 五、 工程难点：处理“不完全写”（Partial Write）

在非阻塞 IO（如 `epoll` 模式）下，由于发送缓冲区满，`writev()` 经常只写出一部分数据。此时必须**更新偏移量**并重试。

#### 核心逻辑：`iovec` 消耗函数

该函数用于动态调整 `iovec` 数组，使其指向剩余未发送的数据：
```C++
void consume_iov(struct iovec*& iov, int& iovcnt, size_t n) {
    while (iovcnt > 0 && n > 0) {
        if (n >= iov[0].iov_len) {
            // 当前这块 buffer 已写完，跳到下一块
            n -= iov[0].iov_len;
            iov++;
            iovcnt--;
        } else {
            // 当前 buffer 只写了一部分，更新起始地址和长度
            iov[0].iov_base = (char*)iov[0].iov_base + n;
            iov[0].iov_len -= n;
            n = 0;
        }
    }
}
```

**完全写writev_all**
```C++
size_t iov_total_bytes(const struct iovec* iov, int iovcnt) {
    size_t total = 0;
    for (int i = 0; i < iovcnt; i++) {
        total += iov[i].iov_len;
    }
    return total;
}//获取一共多少字节；

ssize_t writev_all(int fd,struct iovec* iov,int count){
	size_t total=iov_total_bytes(iov,count);
	while(total>0 && count>0){
		ssize_t n = writev(fd,iov,count);
		if(n<0){
		return -1;
		}
		if(n==0){
		break;
		}
		while(count>0 && (size_t)n>=iov[0].iov_len){
		total=total-iov[0].iov_len;
		n=(size_t)n-iov[0].iov_len;
		count--;
		iov++;
		}
		if(count!=0 && n<iov[0].iov_len){
			iov[0].iov_len=iov[0].iov_len-(size_t)n;
			iov[0].iov_base=(char*)iov[0].iov_base+(size_t)n;
			total=total-(size_t)n;
			}
	}
	return total;
}
```


---

### 六、 典型使用场景

- **场景 1：HTTP 响应（Header 与 Body 分离）**
    
    - 静态资源服务器：Header 在内存中，Body 在磁盘（由 `mmap` 映射）。
        
- **场景 2：日志系统**
    
    - 同时写入 `[时间戳] [日志等级] [内容] [\n]`，无需拼装字符串。
        

---

### 七、 相关函数横向对比

|**函数**|**特点**|**适用场景**|
|---|---|---|
|**write**|单缓冲区写入|简单数据写入|
|**writev**|多缓冲区聚集写|协议头与负载分离（如 HTTP/RPC）|
|**send**|带 flags 的 socket 写入|需要控制发送行为（如 `MSG_NOSIGNAL`）|
|**sendmsg**|**最强接口**：多缓冲区 + flags + 辅助数据|复杂的套接字通信、传递文件描述符|
|**sendfile**|磁盘到网卡的零拷贝|大文件传输|

---

### 八、 总结（面试/避坑金句）

> **“`writev()` 并不是万能药。它虽然减少了 `memcpy`，但在网络编程中，你必须像对待 `write()` 一样，通过循环和偏移量维护来处理‘短写’问题。永远不要假设一次 `writev()` 能把所有 `iovec` 里的东西都发完。”**

