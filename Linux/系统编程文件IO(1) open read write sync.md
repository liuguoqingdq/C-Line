“最基本的文件访问方法是系统调用read()和write()。但是，在访问文件之前，必须先通过open()或creat()打开该文件。一旦完成文件读写，还应该调用系统调用close()关闭该文件。”

## 1) 文件 I/O（open/read/write/close/lseek）
```C
#include <unistd.h>    // read, write, close, lseek
#include <fcntl.h>     // open, O_RDONLY/O_CREAT/... flags
#include <sys/stat.h>  // mode_t, 权限位 S_IRUSR 等（创建文件常用）
#include <sys/types.h> // ssize_t, off_t 等（有时可省，现代系统常被间接包含）
```
### open()系统调用

**通过系统调用open()，可以打开文件并获取其文件描述符：**

```C
int open(const char *name, int flags);
int open(const char *name, int flags, mode_t mode);
```
它们的区别是 **第三个参数 `mode` 要不要传**：

- **2 个参数版**：只打开已有文件（或不需要创建权限参数时）
    
    `int fd = open("a.txt", O_RDONLY);`
    
- **3 个参数版**：当 `flags` 里包含 **`O_CREAT`（或某些情况下 `O_TMPFILE`）** 时，必须传 `mode`，用来指定“新建文件的权限位”
    
```C
int fd = open("a.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644); 
// 实际权限还会受 umask 影响：最终=0644 & ~umask
```

###### open()的flags参数 
 flags参数是由一个或多个标志位的按位或组合。它支持三种访问模式：O_RDONLY、O_WRONLY或O_RDWR，这三种模式分别表示以只读、只写或读写模式打开文件。 
 举个例子，以下代码以只读模式打开文件/home/kidd/madagascar：

```C
int fd;
fd = opne("/home/kidd/madagascar",ORDONLY);
if(fd==-1){
/* error */
}
```
不能对以只读模式打开的文件执行写操作，反之亦然。进程必须有足够的权限才能调用系统调用来打开文件。举个例子，假设用户对某个文件只有只读权限，该用户的进程只能以O_RDONLY模式打开文件，而不能以O_WRONLY或O_RDWR模式打开。 
 flags参数还可以和下面列出的这些值进行按位或运算，修改打开文件的行为：
### 1) 创建/覆盖类

- **`O_CREAT`**：文件不存在就创建（这时必须提供第三个参数 `mode`）
    
    ```C
    open("a.txt", O_WRONLY | O_CREAT, 0644);
    ```
    
- **`O_EXCL`**：配合 `O_CREAT` 使用；如果文件已存在则失败（避免覆盖）
     当和标志位O_CREAT一起使用时，如果参数name指定的文件已经存在，会导致open()调用失败。用于防止创建文件时出现竞争。如何没有和标志位O_CREAT一起使用，该标志位就没有任何含义。 
    
    `open("a.txt", O_WRONLY | O_CREAT | O_EXCL, 0644); // 存在则报 EEXIST`
    
- **`O_TRUNC`**：文件已存在且以写方式打开时，把文件长度截断为 0（清空重写）
    
    `open("a.txt", O_WRONLY | O_TRUNC);`
    

### 2) 追加/定位类

- **`O_APPEND`**：每次 `write` 都追加到文件末尾（内核保证追加定位的原子性）
    
    `open("log.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);`
    

### 3) 阻塞行为/同步类

- **`O_NONBLOCK`**：非阻塞 I/O（对 pipe/终端/socket 影响大，对普通磁盘文件通常意义不大）
    
    `open("/dev/tty", O_RDONLY | O_NONBLOCK);`
    
- **`O_SYNC`**：更强的同步写语义（write 返回前尽量保证数据落到存储设备，性能会下降）
    
- **`O_DSYNC`**：只保证数据（不强制元数据）同步（比 O_SYNC 轻一些）
- **`O_RSYNC`**: O_RSYNC标志位指定读请求和写请求之间的同步。该标志位必须和O_SYNC或O_DSYNC一起使用。
- O_LARGEFILE
 文件偏移使用64位整数表示，可以支持大于2GB的文件。64位操作系统中打开文件时，默认使用该参数。
```C
fd = open("a.txt",O_LARGEFILE | O_RDONLY);
```

### 4) 安全/继承类

- **`O_CLOEXEC`**：执行 `exec()` 时自动关闭该 fd，防止泄漏到子进程
    
    `open("a.txt", O_RDONLY | O_CLOEXEC);`
    

### 5) 其他你可能会见到的

- **`O_NOCTTY`**：打开终端设备时不把它变成进程的控制终端
    
- **`O_NOFOLLOW`**：不跟随符号链接（用于安全场景，避免被 symlink 攻击）
    

---

## 典型组合（背这几个就够用）

1. **只读打开**
    

`open("a.txt", O_RDONLY);`

2. **覆盖写（不存在就创建，存在就清空）**
    

`open("a.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);`

3. **追加写日志**
    

`open("log.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);`

4. **只允许“新建”，不允许覆盖**
    

`open("a.txt", O_WRONLY | O_CREAT | O_EXCL, 0644);`

##### 新建文件的权限 
 前面给出的两种open()系统调用方式都是合法的。除非创建了新文件，否则会忽略参数mode；如果给定O_CREAT参数，则需要该参数。在使用O_CREAT参数时如果没有提供参数mode，结果是未定义的，而且通常会很糟糕——所以千万不要忘记！ 
 当创建文件时，参数mode提供了新建文件的权限。对于新建的文件，打开文件时不会检查权限，因此可以执行与权限相反的操作，比如以只读模式打开文件，却在打开后执行写操作。 
 参数mode是常见的UNIX权限位集合，比如八进制数0644（文件所有者可以读写，其他人只能读）。从技术层面看，POSIX是根据具体实现确定值，支持不同的UNIX系统设置自己想要的权限位。但是，每个UNIX系统对权限位的实现都采用了相同的方式。因此，虽然技术上不可移植，但在任何系统上指定0644或0700效果都是一样的。 

S_IRWXU 
 文件所有者有读、写和执行的权限。 
 S_IRUSR 
 文件所有者有读权限。 
 S_IWUSR 
 文件所有者有写权限。 
 S_IXUSR 
 文件所有者有执行权限。 
 S_IRWXG 
 组用户有读、写和执行权限。 
 S_IRGRP 
 组用户有读权限。 
 S_IWGRP 
 组用户有写权限。 
 S_IXGRP 
 组用户有执行权限。 
 S_IRWXO 
 任何人都有读、写和执行的权限。 
 S_IROTH 
 任何人都有读权限。 
 S_IWOTH 
 任何人都有写权限。 
 S_IXOTH 
 任何人都有执行权限。

**为了代码的可读性，通常使用umask码：**
```C
int fd;
fd = open("a.txt",O_RDONLY | O_CREAT,0644)
```

### creat()系统调用函数
`create` 一般指的是 **`creat()`**（注意少了一个 `e`），它是早期 UNIX 的创建文件系统调用接口，现在仍然可用，但现代更推荐用 `open()` 代替。

## 1) `creat()` 函数原型

```C
#include <fcntl.h>  
int creat(const char *pathname, mode_t mode);
```

- 成功：返回文件描述符 `fd >= 0`
    
- 失败：返回 `-1`，并设置 `errno`
    

## 2) `creat()` 等价于什么？

```C
creat(pathname, mode) == open(pathname, O_WRONLY | O_CREAT | O_TRUNC, mode);
```

也就是：

- 只写方式打开
    
- 不存在就创建
    
- 存在就清空（截断为 0）
    

## 3) 一个例子

```C
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    int fd = creat("out.txt", 0644);
    if (fd < 0) { perror("creat"); return 1; }

    write(fd, "hello\n", 6);
    close(fd);
    return 0;
}

```

## 4) 你可能会问：为什么现在不太用 `creat()`？

因为它太“死”：只能 `O_WRONLY|O_CREAT|O_TRUNC` 这一种行为。想追加写、想读写、想 `O_EXCL` 防覆盖等，都得用 `open()`。

### 通过read()读文件
```C
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);
```

## 每个参数分别是什么

### 参数 1：`int fd`

- `fd` 是**文件描述符**，一般来自：
    
    - `open()` 打开文件
        
    - 或者标准输入 `0`（`STDIN_FILENO`）
        
- 内核用 `fd` 找到你打开的“文件对象 + 当前偏移量”等状态。
    

### 参数 2：`void *buf`

- 你提供的**缓冲区地址**，用来装读出来的数据。
    
- 必须保证这块内存**可写**并且至少有 `count` 个字节容量。
    
- 例如：
    
    `char buf[4096]; read(fd, buf, sizeof(buf));`
    

### 参数 3：`size_t count`

- 你希望这次最多读取多少字节。
    
- 注意：这是“最多”，不是“必须读满”。


**每次调用read()函数，会从fd指向的文件的当前偏移开始读取len字节到buf所指向的内存中。执行成功时，返回写入buf中的字节数；出错时，返回-1，并设置errno值。fd的文件位置指针会向前移动，移动的长度由读取到的字节数决定。如果fd所指向的对象不支持seek操作（比如字符设备文件），则读操作总是从“当前”位置开始。 
 基本用法很简单。下面这个例子就是从文件描述符fd所指向的文件中读取数据并保存到word中。读取的字节数即unsigned long类型的大小，在Linux的32位系统上是4字节，在64位系统上是8字节。成功时，返回读取的字节数；出错时，返回-1：**
```C
usigned long world;
ssize_t nr
nr = read(fd,&world,sizeof(world));
if(nr==-1)......
```

## 返回值三种情况（必须背熟）

### 情况 A：返回值 `> 0`

- 表示**实际读取到的字节数**，比如返回 1000，就说明 `buf[0..999]` 有效。
    
- **不保证等于 `count`**：读文件也可能因为系统缓冲、信号等原因“读不满”。
    

### 情况 B：返回值 `== 0`

- 表示 **EOF（文件结束）**：已经读到文件末尾，再读就没有数据了。
    
- 对普通文件，这是“读完”的标志。
    

### 情况 C：返回值 `== -1`

- 表示出错，具体原因在全局变量 `errno` 里。
    
- 最常见的你需要认识两个：
    
    - `errno == EINTR`：被信号打断，通常**重试**即可
        
    - `errno == EAGAIN / EWOULDBLOCK`：非阻塞读时没数据（普通文件很少遇到）

### 读入所有字节

**诚如前面所描述的，由于调用read()会有很多不同情况，如果希望处理所有错误并且真正每次读入len个字节（至少读到EOF），那么之前简单“粗暴”的read()调用并不合理。要实现这一点，需要有个循环和一些条件语句，如下：**

```C
ssize_t ret;
size_t len=MAX_SIZE;//读入预定字节数
while(len!=0 && (ret=read(fd,buf,len))!=0 )//读到文件末尾
{
	if(ret==-1){
		if(error==EINTR) continue;
		perror("error");
		break;
	}
	len-=(size_t)ret;
	buf+=ret;
}
```

**这段代码判断处理了五种情况。循环从fd所指向的当前文件位置读入len个字节到buf中，一直读完所有len个字节或者EOF为止。如果读入的字节数大于0但小于len，就从len中减去已读字节数，buf增加相应的字节数，并重新调用read()。如果调用返回-1，并且errno值为EINTR，会重新调用且不更新参数。如果调用返回-1，并且errno设置为其他值，会调用perror()，向标准错误打印一条描述，循环结束。**

`read()` 读出来的是**原始字节**，它只返回“实际读了多少字节”，**不会自动在末尾补 `'\0'`**。

所以：

- 如果把 `read` 的结果当作 **C 字符串**来用，必须自己加终止符，并且要预留一个字节空间。
实例：
```C
char buf[1024];
ssize_t n = read(fd, buf, sizeof(buf) - 1); // 预留 1 字节给 '\0'
if (n > 0) {
    buf[n] = '\0';  // 手动补 '\0'
    printf("%s", buf);
}
```
补充两点常见坑：

- `read` 读到的内容里**可能本来就包含 `'\0'`**（比如二进制文件），这时即使你手动补了终止符，`printf("%s")` 也会在中间的 `'\0'` 提前截断。
    
- 如果 `n == sizeof(buf)-1`，上面这种写法仍然安全，因为你预留了 1 字节。

**有时，开发人员不希望read()调用在没有数据可读时阻塞在那里。相反地，他们希望调用立即返回，表示没有数据可读。这种方式称为非阻塞I/O，它支持应用以非阻塞模式执行I/O操作，因而如果是读取多个文件，以防错过其他文件中的可用数据。 
 因此，需要额外检查errno值是否为EAGAIN。正如前面所讨论的，如果文件描述符以非阻塞模式打开（即open()调用中指定参数为O_NONBLOCK，参见2.1.1节中的“open()调用的参数”），并且没有数据可读，read()调用会返回-1，并设置errno值为EAGAIN，而不是阻塞模式。当以非阻塞模式读文件时，必须检查EAGAIN，否则可能因为丢失数据导致严重错误。你可能会用到如下代码：**
 ```C
 char buf[MAX_SIZE];
 usigned int len;
 start:
 ssize_t ret = read(fd,buf,len);
 if (ret==-1){
	 if(error==ENTER) goto satrt;
	else if(error==EAGAIN)....;
	else...;
 }
 ```
### 怎么打开非阻塞设置

**方法 A：`open` 时带 `O_NONBLOCK**`
```C
int fd = open("a.txt",O_RDONLY | O_NONBLOCK); 
```

**方法 B：对已有 fd 用 `fcntl` 打开非阻塞**
```C
#include <fcntl.h>

int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

## 非阻塞读的正确姿势：配合 select/poll/epoll

非阻塞不是让你疯狂 `while(read...)` 空转（那会 CPU 100%），而是：

1）先等“可读事件”（有数据了再读）  
2）再 `read()`，并且通常要**一直读到 EAGAIN** 才停（把缓冲区读空）

```C
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <stdio.h>
#include <poll.h>

int main() {
    int fd = STDIN_FILENO;

    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);

    char buf[1024];

    while (1) {
        struct pollfd pfd = { .fd = fd, .events = POLLIN };
        int r = poll(&pfd, 1, 5000); // 等 5 秒
        if (r == 0) { puts("timeout..."); continue; }
        if (r < 0) { perror("poll"); break; }

        // 可读了：把当前能读的都读出来（直到 EAGAIN）
        while (1) {
            ssize_t n = read(fd, buf, sizeof(buf));
            if (n > 0) {
                write(STDOUT_FILENO, buf, (size_t)n);
                continue;
            }
            if (n == 0) return 0; // EOF
            if (errno == EINTR) continue;
            if (errno == EAGAIN || errno == EWOULDBLOCK) break;
            perror("read");
            return 1;
        }
    }
    return 0;
}


```

#### 其他错误码 
 **其他错误码指的是编程错误或（对EIO而言）底层问题。read()调用执行失败后，可能的errno值包括： 
 EBADF 
 给定的文件描述符非法或不是以可读模式打开。 
 EFAULT 
 buf指针不在调用进程的地址空间内。 
 EINVAL 
 文件描述符所指向的对象不允许读。 
 EIO 
 底层I/O错误。 

#### read()调用的大小限制 
 **类型size_t和ssize_t是由POSIX确定的。类型size_t保存字节大小，类型ssize_t是有符号的size_t（负值用于表示错误）。在32位系统上，对应的C类型通常是unsigned int和int。因为这两种类型常常一起使用，ssize_t的范围更小，往往限制了size_t的范围。 
 size_t的最大值是SIZE_MAX，ssize_t的最大值是SSIZE_MAX。如果len值大于SSIZE_MAX，read()调用的结果是未定义的。在大多数Linux系统上，SSIZE_MAX的值是LONG_MAX，在32位系统上这个值是2 147 483 647。这个数值对于一次读操作而言已经很大了，但还是需要留心它。如果使用之前的读循环作为通用的读方式，可能需要给它增加以下代码：

```C
if(len>SSIZE_MAX) len=SSIZE_MAX;
```
**“调用read()时如果len参数为0，会立即返回，且返回值为0。”

### write写系统调用

**“写文件，最基础最常见的系统调用是write()。和read()一样，write()也是在POSIX.1中定义的：”

## 1) 函数签名（原型）

```C
#include <unistd.h>
ssize_t write(int fd, const void *buf, size_t count);
```

- `ssize_t`：有符号类型，方便用 `-1` 表示错误。
    
- `write` 会把 `buf` 指向的 `count` 个字节写到 `fd` 对应的对象（文件/终端/管道/socket…）。
    

---

## 2) 参数解释

### 参数 1：`int fd`

- 文件描述符：来自 `open()`/`creat()`/`socket()` 等。
    
- 常见的：
    
    - `STDOUT_FILENO`（1）：标准输出
        
    - `STDERR_FILENO`（2）：标准错误
        

### 参数 2：`const void *buf`

- 要写出的数据在内存中的起始地址。
    
- `const` 表示写不会修改你的缓冲区内容。
    

### 参数 3：`size_t count`

- 希望写出的字节数（最多写这么多）。
    
- 注意：**不保证一次写完**（尤其是管道/socket/非阻塞 fd）。
    

---

## 3) 返回值（必须背）

`ssize_t n = write(fd, buf, count);`

- `n >= 0`：本次**实际写入的字节数**（可能小于 `count`，也可能等于）
    
- `n == -1`：出错，查看 `errno`
    

### 常见 errno

- `EINTR`：被信号打断（通常重试）
    
- `EAGAIN / EWOULDBLOCK`：非阻塞写时“现在写不进去”
    
- `EBADF`：fd 不可写（比如用 `O_RDONLY` 打开的 fd 去 write）
    
- `EPIPE`：写到已关闭读端的管道/已关闭连接的 socket（通常还会触发 `SIGPIPE`）
    

---

## 4) 关键语义：为什么 write 可能写不完？

对普通磁盘文件，很多时候看起来“像是一次写完”，但对以下对象很常见写不完：

- **管道 pipe**
    
- **socket**
    
- **设置了 `O_NONBLOCK` 的 fd**
    
- 内核缓冲区空间不够时
    

所以健壮代码要考虑 **partial write**（部分写）。

## 5) 正确姿势：写满指定字节数（write_all）

这是最常用的封装：确保把 `buf` 的 `count` 字节全部写出去，除非遇到不可恢复错误。
```C
#include <unistd.h>
#include <errno.h>

ssize_t write_all(int fd, const void *buf, size_t count) {
	const char *p = (const char*)buf;
	size_t len = count;
	while(len>0){
		ssize_t ret = write(fd,p,len);
		if(ret>0){
		len-=(size_t)ret;
		p+=ret;
		continue;
		}
		if(ret==-1 && errno==ENTER){
		continue;
		}
		return-1;
	}
	return ssize_t(count);
}
```

### write()大小限制 
 **如果count值大于SSIZE_MAX，调用write()的结果是未定义的。 
 调用write()时，如果count值为零，会立即返回，且返回值为0.

```C
if(count>SSIZE_MAX) count=SSIZE_MAX;
```

### write的行为
**当write()调用返回时，内核已经把数据从提供的缓冲区中拷贝到内核缓冲区中，但不保证数据已经写到目的地。实际上，write调用执行非常快，因此不可能保证数据已经写到目的地。处理器和硬盘之间的性能差异使得这种情况非常明显。 
 
 **相反，当用户空间发起write()系统调用时，Linux内核会做几项检查，然后直接把数据拷贝到缓冲区中。然后，在后台，内核收集所有这样的“脏”缓冲区（即存储的数据比磁盘上的数据新），进行排序优化，然后把这些缓冲区写到磁盘上（这个过程称为回写writeback）。通过这种方式，write()可以频繁调用并立即返回。这种方式还支持内核把写操作推迟到系统空闲时期，批处理很多写操作。 

**延迟写并没有改变POSIX语义。举个例子，假设要对一份刚写到缓冲区但还没写到磁盘的数据执行读操作，请求响应时会直接读取缓冲区的数据，而不是读取磁盘上的“陈旧”数据。这种方式进一步提高了效率，因为对于这个读请求，是从内存缓冲区而不是从硬盘中读的。如期望的那样，读写请求相互交织，而结果也和预期一致——当然，前提是在数据写到磁盘之前，系统没有崩溃！虽然应用可能认为写操作已经成功了，在系统崩溃情况下，数据却没有写入到磁盘。

**延迟写的另一个问题在于无法强制“顺序写（write ordering）”。虽然应用可能会考虑对写请求进行排序，按特定顺序写入磁盘；而内核主要是出于性能考虑，按照合适的方式对写请求重新排序。只有当系统出现崩溃时，延迟写才会有问题，因为最终所有的缓冲区都会写回，而且文件的最终状态和预期的一致。实际上绝大多数


## 同步IO
**虽然同步I/O是个很重要的主题，但不必过于担心延迟写的问题。写缓冲带来了极大的性能提升，因此，任何操作系统，甚至是那些“半吊子”的操作系统，都因支持缓冲区实现了延迟写而可以称为“现代”操作系统。然而，有时应用希望能够控制何时把数据写到磁盘。在这种场景下，Linux内核提供了一些选择，可以牺牲性能换来同步操作。

## 1）同步 vs 异步：说的是“调用返回时，I/O 是否已经完成”

### 同步 I/O（Synchronous I/O）

- `read/write` **返回时**，这次 I/O 请求在语义上已经“完成”了：
    
    - `read`：数据已经拷贝到你的 `buf`
        
    - `write`：数据已经从你的 `buf` 拷贝到内核（至少进入内核缓冲/页缓存）
        

> 注意：这不等于“数据已经落盘”（下面第 2 点会讲）。

### 异步 I/O（Asynchronous I/O）

- 发起 I/O 后**立即返回**，真正完成在“以后”，你通过回调/事件/完成队列拿到结果。
    
- 典型：POSIX AIO、Linux `io_uring` 等。
    

---

## 2）阻塞 vs 非阻塞：说的是“没准备好时会不会卡住等”

这和“同步/异步”是**另一条维度**：

- **阻塞（blocking）**：没数据可读/不可写时，`read/write` 可能睡眠等待。
    
- **非阻塞（non-blocking, O_NONBLOCK）**：没数据时立即返回 `-1`，`errno=EAGAIN/EWOULDBLOCK`，你自己用 `select/poll/epoll` 再来一次。
    

所以你会看到三种常见组合：

- **同步 + 阻塞**：最常见的 `read()`（默认）
    
- **同步 + 非阻塞**：`O_NONBLOCK` + `read()`（读不到就 EAGAIN）
    
- **异步**：`io_uring`/AIO（完成后通知你）
    

---

## 3）文件写入里的“同步”经常特指：落盘语义（durability）

对**普通文件**，即使是同步调用 `write()`，它也可能只是写进了**页缓存（page cache）**，并没有立刻写到磁盘。

如果你想要更强的“写完就落盘/更接近落盘”，常见手段：

### A. `open` 时加 `O_SYNC` / `O_DSYNC`

```C
int fd = open("log.txt", O_WRONLY|O_CREAT|O_APPEND|O_SYNC, 0644);
write(fd, buf, len);
```

- `O_SYNC`：数据+元数据更强同步（更慢）
    
- `O_DSYNC`：主要保证数据同步（通常比 O_SYNC 快点）
    

### B. 写完后调用 `fsync()` / `fdatasync()`

```C
write(fd, buf, len);
fsync(fd);        // 更强：刷数据+必要元数据
// 或 fdatasync(fd); // 通常只刷数据
```

这才是你在“日志/数据库/关键配置写入”里常说的“同步写”。

---

## 4）一句话总结你该怎么记

- **同步 I/O（调用语义）**：返回时这次 I/O 已经完成到“规定阶段”（read 已拷到用户态；write 已交给内核）。
    
- **阻塞/非阻塞（等待行为）**：没准备好时等不等。
    
- **落盘同步（耐久性）**：要靠 `O_SYNC/O_DSYNC` 或 `fsync/fdatasync`。

```C
int ret;
ret=fsync(fd);
if(ret==-1){//errno}
ret=fdatasync(fd);
if(ret==-1){//erron}
```

**fdatasync()的功能和fsync()类似，其区别在于fdatasync()只会写入数据以及以后要访问文件所需要的元数据。例如，调用fdatasync()会写文件的大小，因为以后要读该文件需要文件大小这个属性。fdatasync()不保证非基础的元数据也写到磁盘上，因此一般而言，它执行更快。对于大多数使用场景，除了最基本的事务外，不会考虑元数据如文件修改时间戳，因此fdatasync()就能够满足需求，而且执行更快。 
 fsync()通常会涉及至少两个I/O操作：一是回写修改的数据，二是更新索引节点的修改时间戳。因为索引节点和文件数据在磁盘上可能不是紧挨着——因而会带来代价很高的seek操作——在很多场景下，关注正确的事务顺序，但不包括那些对于以后访问文件无关紧要的元数据（比如修改时间戳），使用fdatasync()是提高性能的简单方式。

**这两个函数都不保证任何已经更新的包含该文件的目录项会同步到磁盘上。这意味着如果文件链接最近刚更新，文件数据可能会成功写入磁盘，但是却没有更新到相关的目录中，导致文件不可用。为了保证对目录项的更新也都同步到磁盘上，必须对文件目录也调用fsync()进行同步。

**成功时，两个调用都返回0。失败时，都返回-1，并设置errno值为以下三个值之一： 
 EBADF 
 给定文件描述符不是以写方式打开的合法描述符。 
 EINVAL 
 给定文件描述符所指向的对象不支持同步。 
 EIO 
 在同步时底层I/O出现错误。这表示真正的I/O错误，经常在发生错误处被捕获。 

### sync()系统调用
`sync()` 是 Linux/Unix 里的**系统调用**（也有同名的 libc 封装函数），作用是把内核页缓存/缓冲区里“脏”的数据尽快刷回磁盘，让文件系统状态更接近“落盘”。
## 1) 它到底做了什么

- 把**所有挂载文件系统**中尚未写回的：
    
    - 文件数据（page cache 里的脏页）
        
    - 元数据（inode、目录项、superblock 等）
        
- 触发写回（writeback），让内核把这些脏数据提交到块设备。
    

> 关键点：在 Linux 上，`sync()` **通常是“发起/调度写回”后就返回**，并不保证你返回那一刻数据已经全部写完（语义上更像“kick off writeback”）。


## 2) 函数原型与返回
```C
#include <unistd.h>
void sync(void);
```
返回类型是 `void`，也就是说你**拿不到错误码**（出错通常会在后续写回/设备 I/O 中体现）。

## 3) 和 fsync / fdatasync / syncfs 的区别（非常重要）

- `fsync(fd)`：只针对**某一个文件描述符**对应的文件，把该文件的**数据+必要的元数据**刷到稳定存储（通常会等待完成）。
    
- `fdatasync(fd)`：更偏向只保证**数据**落盘，尽量少刷元数据（性能可能更好，但语义更窄）。
    
- `syncfs(fd)`：针对 **`fd` 所在的那个文件系统** 做同步（范围比 `sync()` 小），通常会等待完成。
    
- `sync()`：范围最大（全系统所有文件系统），开销最大，而且在 Linux 上经常只是调度写回。
    

**结论：想保证“某个关键文件真正持久化”，优先用 `fsync/fdatasync`，而不是 `sync()`。**

## 4) 典型使用场景

- 关机/重启前（系统工具会做，不建议业务程序频繁调用）。
    
- 做系统级“全盘落盘”操作（例如运维脚本），但要接受它很重、影响整机 IO。

## 5) 代码示例：写文件并保证落盘
```C
int fd = open("data.log", O_WRONLY|O_CREAT|O_TRUNC, 0644);
write(fd, buf, len);

/* 保证 data.log 内容尽量落盘 */
if (fsync(fd) < 0) perror("fsync");

/* 如果你创建/重命名了文件，严格持久化还需要 fsync 目录 */
int dfd = open(".", O_RDONLY);
if (dfd >= 0) { fsync(dfd); close(dfd); }

close(fd);
```

#### 那我fsync(dfd)，能不能把dfd目录下修改的文件一并落盘？
一般 **不行**。

`fsync(dfd)`（`dfd` 是目录的 fd）主要保证的是这个**目录自身的元数据**落盘——典型就是：

- 目录项更新：`create`/`unlink`/`rename` 让“文件名 ↔ inode”的映射变化
    
- 目录 inode / 链接数 / 时间戳等
    

它**不会**替你把“这个目录下面被修改过的普通文件的数据页”一并强制落盘。也就是说：

- 你只是 `write()` 改了 `a.txt` 的内容（目录项没变）：`fsync(目录fd)`基本没用，必须 `fsync(a_fd)`。
    
- 你做了“安全写入”流程：写 `tmp` → `fsync(tmp_fd)` → `rename(tmp, a.txt)` → **`fsync(父目录fd)`**  
    这里最后一步的目录 fsync 是为了保证 `rename` 这个目录项变化也落盘。
    

如果你想“把某个文件系统上的东西都同步一下”，可用：

- `syncfs(fd)`：同步 **fd 所在的文件系统**（范围比 `sync()` 小），但这仍然是“文件系统级”，不是“只同步某个目录下所有文件”。
    

最可靠的做法仍是：**对关键文件逐个 `fsync`/`fdatasync`，必要时再 `fsync` 父目录**（涉及创建/重命名/删除时）。

