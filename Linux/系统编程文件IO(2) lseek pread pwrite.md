##  1)直接IO
**和其他现代操作系统内核一样，Linux内核实现了复杂的缓存、缓冲以及设备和应用之间的I/O管理的层次结构。高性能的应用可能希望越过这个复杂的层次结构，进行独立的I/O管理。

**但是，创建一个自己的I/O系统往往会事倍功半，实际上，操作系统层的工具往往比应用层的工具有更好的性能。此外，数据库系统往往倾向于使用自己的缓存，以尽可能减少操作系统带来的开销。 

 **在open()中指定O_DIRECT标志位会使得内核对I/O管理的影响最小化。如果提供O_DIRECT标志位，I/O操作会忽略页缓存机制，直接对用户空间缓冲区和设备进行初始化。所有的I/O操作都是同步的，操作在完成之前不会返回。 

 **使用直接I/O时，请求长度、缓冲区对齐以及文件偏移都必须是底层设备扇区大小（通常是512字节）的整数倍。在Linux内核2.6以前，这项要求更加严格：在Linux内核2.4中，所有的操作都必须和文件“系统的逻辑块大小对齐（一般是4KB）。为了保持兼容性，应用需要对齐到更大（而且操作更难）的逻辑块大小。


## 2)关闭文件
当程序完成对某个文件的操作后，可以通过系统调用close()取消文件描述符到对应文件的映射：

```C
#include <unistd.h>
int close(int fd);
```

**系统调用close()会取消当前进程的文件描述符fd与其关联的文件之间的映射。调用后，先前给定的文件描述符fd不再有效，内核可以随时重用它，当后续有open()调用或creat()调用时，重新把它作为返回值。close()调用在成功时返回0，出错时返回-1，并相应设置errno值。close()的用法很简单：

```C
#include <unistd.h>
if(close(fd)<0){
	perror("close");
}
```

## 3) lseek系统调用与定位读写

### 1) `lseek()` 是什么

`lseek()` 用来改变一个文件描述符 `fd` 的当前读写位置（**file offset**）。

- 之后你再 `read(fd, ...)` / `write(fd, ...)`，都会从这个 offset 开始读/写。
    
- **注意：** 它只改“指针”，不读也不写数据。
    

---

### 2) 函数原型与返回值


```C
#include <unistd.h>
#include <sys/types.h>

off_t lseek(int fd, off_t offset, int whence);
```

- **成功：** 返回新的文件位置（相对于文件开头的字节偏移量）。
    
- **失败：** 返回 `-1`，并设置 `errno`。
    
- **类型：** `off_t` 是“文件偏移/文件大小”类型（通常是 64-bit，具体取决于编译选项和系统位数）。
    

---

### 3) `whence` 三种模式（最核心）

`whence` 参数决定了 `offset` 是基于哪里开始计算的。

**A. SEEK_SET (从文件开头算)**


```C
lseek(fd, 0, SEEK_SET);      // 回到文件开头
lseek(fd, 100, SEEK_SET);    // 定位到第 100 字节处
```

**B. SEEK_CUR (从当前位置算)**


```C
lseek(fd, 10, SEEK_CUR);     // 向前跳 10 字节
lseek(fd, -5, SEEK_CUR);     // 向后退 5 字节
```

**C. SEEK_END (从文件末尾算)**

```C
lseek(fd, 0, SEEK_END);      // 跳到文件末尾（EOF 位置）
lseek(fd, -20, SEEK_END);    // 跳到“倒数 20 字节”处（前提：文件必须足够大）
```

[[linux系统编程练习lseek]]

---

### 4) 最常用的 4 个“教学级”例子

**例1：获取文件大小** 利用 `SEEK_END` 返回值的特性：

C

```C
off_t cur = lseek(fd, 0, SEEK_CUR);  // 1. 记住当前位置
off_t end = lseek(fd, 0, SEEK_END);  // 2. end 的返回值就是文件大小
lseek(fd, cur, SEEK_SET);            // 3. 恢复原位置（可选）
```

**例2：跳到末尾追加写（手动方式）**

```C
lseek(fd, 0, SEEK_END);
write(fd, "hello\n", 6);
```

> **注意：** 并发环境下，这种方式会导致竞态条件（Race Condition），推荐使用 `open(..., O_APPEND)`。

**例3：读取某一段（随机访问）**


```C
lseek(fd, 1024, SEEK_SET);
read(fd, buf, 128);  // 从 1024 处读 128 字节
```

**例4：制造“空洞文件”（Sparse File）**


```C
lseek(fd, 1024*1024, SEEK_SET);  // 跳到 1MB 位置
write(fd, "X", 1);               // 中间没写过的部分不会真实占用磁盘块
```

---

### 5) 必须知道的坑（考试/面试重点）

- **坑1：不是所有 fd 都能 lseek**
    
    - **管道 (pipe) / socket / 终端 (terminal)**：不能 seek，调用会失败，`errno` 设置为 `ESPIPE`。
        
    - 只有“可随机访问”的对象（如普通文件、块设备）才行。
        
- **坑2：SEEK_END 往回跳要注意负数和越界**
    
    
    
    ```C
    off_t pos = lseek(fd, -20, SEEK_END);
    if (pos == (off_t)-1) perror("lseek");  // 可能 EINVAL（比如文件本来就很小）
    ```
    
- **坑3：并发/多线程共享同一个 fd 时，offset 是共享状态**
    
    - 如果两个线程同时对同一个 `fd` 读写，会互相干扰 offset。
        
    - **解决思路：**
        
        1. 用 `pread` / `pwrite`（不改变共享 offset）。
            
        2. 每个线程独立 `open` 一个 fd。
            
        3. 加锁保护 `lseek` + `read/write` 组合。
            
- **坑4：追加写别用 `lseek(END) + write` 来保证原子性**
    
    - 在多进程/多线程并发时会发生：A seek 到 end -> B seek 到 end -> A write -> B write（导致覆盖/交叉）。
        
    - **正确方式：** 用 `O_APPEND` 打开文件：
        
        
        
        ```C
        int fd = open("log.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);
        write(fd, msg, len);   // 内核保证每次 write 都原子性地追加到当时的 EOF
        ```
        

---

### 6) 完整可运行的小程序（演示 seek / size / 随机读）


```C
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>

int main() {
    // 打开文件，如果不存在则创建，如果存在则清空
    int fd = open("demo.bin", O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) { perror("open"); return 1; }

    // 写入初始数据
    write(fd, "ABCDEFGH", 8);

    // 获取文件大小
    off_t size = lseek(fd, 0, SEEK_END);
    printf("size = %lld\n", (long long)size);  // 输出: 8

    // 随机读取演示
    lseek(fd, 3, SEEK_SET);                    // 指到 'D' (下标3)
    char c;
    read(fd, &c, 1);
    printf("byte@3 = %c\n", c);                 // 输出: D

    // 相对位移演示
    lseek(fd, 2, SEEK_CUR);                    // 从当前位置(读取过D后的位置)再跳 2
    read(fd, &c, 1);
    printf("after +2, byte = %c\n", c);         // 'D'之后是'E'，再跳2是'G'

    close(fd);
    return 0;
}
```

---

### 7) 什么时候优先用 `pread` / `pwrite`

如果你要“定位到某处读/写”，但又**不想影响 fd 的当前位置**（尤其在并发环境下），请使用：


```C
ssize_t pread(int fd, void *buf, size_t count, off_t offset);
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);
```

它们等价于“带 offset 的 read/write”，但**不会改变共享的文件偏移量**，是线程安全的读写方式。

## 1) 原型与语义

```C
ssize_t pread(int fd, void *buf, size_t count, off_t offset);
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);
```

- 成功：返回实际读/写的字节数（可能 **< count**）
    
- 失败：返回 `-1`，设置 `errno`
    
- `offset`：从文件开头算的字节偏移（`off_t`）
    

### 关键性质

- **不改变** `fd` 的 file offset
    
- 对普通文件（regular file），在并发场景更安全：多个线程/进程可以对同一个 `fd` 用不同 offset 操作，互不干扰 “当前位置”。

**写满循环模板**
```C
ssize_t write_full_pwrite(int fd, const void* buf, size_t n, off_t off) {
	const char* p=(const char*)buf;
	size_t len=n;
	while(len>0){
		ssize_t ret = pwrite(fd,p,len,off);
		if(ret<0){
			if(errno==EINTR)continue;
			perror("error");
			return -1;
		}
		len-=(size_t)ret;
		p+=ret;
		off+=ret;
	}
	return (ssize_t)n;
}
```


```C++
ssize_t pread_full(int fd,void* buf,size_t n,off_t off){
	char*p=(char*)buf;
	size_t len=n;
	while(len>0){
		ssize_t ret=pread(fd,p,len,off);
		if(ret<0){
			if(errno==EINTR) continue;
			perror("pread");
			return -1;
		}
		if(ret==0)break;
		len-=(size_t)ret;
		p+=ret;
		off+=ret;
	}
	return (ssize_t)(n-len);//实际读取的字节数
}
```