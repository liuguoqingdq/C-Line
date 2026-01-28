# C 文件流进阶：写入、定位、错误、缓冲、线程安全与文件锁（含示例）

本教程围绕 `FILE*`（stdio 文件流）中你点名的能力：

- **写入**：`fputc` / `fputs` / `fwrite`
- **定位**：`fseek` / `rewind`（并补充 `ftell`）
- **错误状态**：`ferror`（并补充 `clearerr` / `feof`）
- **控制缓冲**：`setvbuf` / `setbuf` / `fflush`
- **线程安全（同进程多线程）**：`flockfile` / `ftrylockfile` / `funlockfile` 与 `*_unlocked`
- **手动文件加锁（跨进程）**：`flock` 或 `fcntl` 记录锁
- **对流操作解锁**：`funlockfile`（流锁）/ `flock(LOCK_UN)`（文件锁）

> 重要提醒：
> - **流锁（flockfile）**解决的是**同一进程内多线程**对同一 `FILE*` 的并发互斥。
> - **文件锁（flock/fcntl）**解决的是**跨进程**的互斥/协作（通常是建议锁 advisory）。

---

## 1. 向流写数据

### 1.1 `fputc`：写一个字符

**函数签名**
```c
#include <stdio.h>
int fputc(int c, FILE *stream);
```

**参数意义**
- `c`：要写入的字符（以 `unsigned char` 的值写入，但参数类型是 `int`）
- `stream`：目标流

**返回值**
- 成功：返回写入的字符（转换成 `unsigned char` 后再转为 `int`）
- 失败：返回 `EOF`，并设置该流的错误标志（`ferror(stream)` 会变为真）

**用途**
- 做逐字符输出、简单状态机输出、生成文本时逐字符拼接。

**小例子**
```c
for (int i = 0; i < 10; ++i) {
    if (fputc('0' + i, fp) == EOF) { /* 处理错误 */ }
}
fputc('\n', fp);
```

---

### 1.2 `fputs`：写一个 C 字符串（不写结尾 `\0`）

**函数签名**
```c
#include <stdio.h>
int fputs(const char *s, FILE *stream);
```

**参数意义**
- `s`：以 `\0` 结尾的字符串
- `stream`：目标流

**返回值**
- 成功：返回非负值（具体值实现相关）
- 失败：返回 `EOF`

**用途与注意**
- 适合写一行或一段文本。
- **不会自动写换行**，要自己加 `\n`。
- **不会写入字符串末尾的 `\0`**。

**小例子**
```c
if (fputs("hello", fp) == EOF) { /* error */ }
if (fputs("\n", fp) == EOF) { /* error */ }
```

---

### 1.3 `fwrite`：写二进制块/数组（按“元素”写）

**函数签名**
```c
#include <stdio.h>
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
```

**参数意义**
- `ptr`：数据起始地址
- `size`：每个元素的字节数
- `nmemb`：元素个数
- `stream`：目标流

**返回值**
- 返回**成功写入的元素个数**（不是字节数）
- 若返回值 `< nmemb`：说明发生错误或无法继续写入
  - 这时应检查 `ferror(stream)`

**用途与注意**
- 适合写：二进制文件、结构化数据、图片、压缩数据、网络记录等。
- 写结构体要小心：padding、字节序、ABI、浮点格式等（跨平台文件格式通常不直接 fwrite 结构体）。

**小例子：写 1024 字节**
```c
unsigned char buf[1024];
size_t n = fwrite(buf,sizeof(buf),1,fp);
if (n != 1) {
    if (ferror(fp)) { /* error */ }
}
```

更常见、也更不容易写错的写法是把 `size=1`，让返回值等于“写入的字节数”：

```C
size_t n = fwrite(buf, 1, sizeof(buf), fp);
if (n != sizeof(buf)) {
    if (ferror(fp)) { /* error */ }
}
```

两种写法都对，只要**检查逻辑匹配返回值语义**即可。
#### 解决padding方案 C：序列化（最正规，推荐）

不要偷懒直接 `fwrite` 整个结构体，而是**逐个成员写入**。这是通用的“序列化”思路。

**做法：**


```C
// 写入
fwrite(&s.id, sizeof(s.id), 1, fp);
fwrite(&s.score, sizeof(s.score), 1, fp);

// 读取
fread(&s.id, sizeof(s.id), 1, fp);
fread(&s.score, sizeof(s.score), 1, fp);
```
---

## 2. 定位流：fseek / rewind（并补充 ftell）

> 定位函数对**可随机访问的流**有效（普通文件通常 OK）。对 pipe/socket 这种流一般不可定位。

### 2.1 `fseek`：移动文件位置指针

**函数签名**
```c
#include <stdio.h>
int fseek(FILE *stream, long offset, int whence);
```

**参数意义**
- `stream`：目标流
- `offset`：偏移量（字节）
- `whence`：基准位置
  - `SEEK_SET`：从文件开头
  - `SEEK_CUR`：从当前位置
  - `SEEK_END`：从文件末尾

**返回值**
- 成功：0
- 失败：非 0（并设置错误标志）

**用途**
- 随机读写、回填文件头、跳过某段数据、重读某段。

**重要规则（更新模式）**
当文件以更新模式打开（例如 `"r+"`, `"w+"`, `"a+"`），
- **在写后立刻读**，或 **在读后立刻写**，必须插入一次：
  - `fflush(stream)` 或 任意定位调用（`fseek/fsetpos/rewind`）
否则行为未定义。
##### 正确版本A:
```C
FILE *fp = fopen("t.txt", "w+");
fputs("ABC\n", fp);

fflush(fp);                        // ✅ 写->读切换：刷新输出缓冲
rewind(fp);                        // ✅ 再把位置移回开头（否则还在文件末尾）
char buf[16];
fgets(buf, sizeof(buf), fp);
printf("%s", buf);

fclose(fp);
```
##### 正确版本B：
```C
FILE *fp = fopen("t.txt", "w+");
fputs("ABC\n", fp);

fseek(fp, 0, SEEK_SET);            // ✅ 写->读切换：定位会同步（并把位置移到开头）
char buf[16];
fgets(buf, sizeof(buf), fp);
printf("%s", buf);

fclose(fp);

```

---

### 2.2 `rewind`：回到开头并清除错误/EOF 标志

**函数签名**
```c
#include <stdio.h>
void rewind(FILE *stream);
```

**行为**
- 等价于：`fseek(stream, 0L, SEEK_SET)` + `clearerr(stream)`

**用途**
- 重新从头读取文件（尤其是读到 EOF 后想再读一遍）。

---

### 2.3 `ftell`：获取当前位置（常和 fseek 配合）

**函数签名**
```c
#include <stdio.h>
long ftell(FILE *stream);
```

**返回值**
- 成功：当前位置（从开头算的字节偏移）
- 失败：-1L

> 更可移植/更大文件：POSIX 还有 `ftello/fseeko`（使用 `off_t`）。

---

## 3. 错误状态：ferror / feof / clearerr

### 3.1 `ferror`：检查该流是否发生 I/O 错误

**函数签名**
```c
#include <stdio.h>
int ferror(FILE *stream);
```

**返回值**
- 非 0：流的错误标志已置位（发生过读/写错误）
- 0：没有错误标志

**什么时候用**
- `fread/fwrite` 返回数量不符合预期时
- `fgets` 返回 NULL 时，需要区分 EOF 还是错误

### 3.2 `feof`：检查是否到 EOF

**函数签名**
```c
#include <stdio.h>
int feof(FILE *stream);
```

**重要概念**
- `feof` 只有在一次读取操作“尝试越过文件末尾”后才会置位。
- 不要用 `while(!feof(fp))` 作为读取循环条件（经典错误）。

实例：
```C
int c;
while ((c = fgetc(fp)) != EOF) { }          // 读到 EOF 或出错都会跳出

if (feof(fp))   puts("到文件末尾(EFO)");     // EOF：读完了
if (ferror(fp)) puts("发生读错误(ERROR)");   // ERROR：读失败了
```

### 3.3 `clearerr`：清除错误标志与 EOF 标志

**函数签名**
```c
#include <stdio.h>
void clearerr(FILE *stream);
```

---

## 4. 控制缓冲：setvbuf / setbuf / fflush

### 4.1 为什么要控制缓冲
- stdio 在用户态有缓冲区：减少系统调用，提高吞吐。
- 但交互式程序/实时日志有时需要更及时的输出。
标准I/O实现了三种类型的用户缓冲，并为开发者提供了接口，可以控制缓冲区类型和大小。以下不同类型的用户缓冲提供不同功能，并适用于不同的场景： 

- 无缓冲（Unbuffered） 
 不执行用户缓冲。数据直接提交给内核。因为这种无缓冲模式不支持用户缓冲（而用户缓冲一般会带来很多好处），通常很少使用，只有一个例外：标准错误默认是采用无缓冲模式。 
 - 行缓冲（Line-buffered） 
 缓冲是以行为单位执行。每遇到换行符，缓冲区就会被提交到内核。行缓冲对把流输出到屏幕时很有用，因为输出到屏幕的消息也是通过换行符分隔的。因此，行缓冲是终端的默认缓冲模式，比如标准输出。 
- 块缓冲（Block-buffered） 
 缓冲以块为单位执行，每个块是固定的字节数。本章一开始讨论的缓冲模式即块缓冲，它很适用于处理文件。默认情况下，和文件相关“的所有流都是块缓冲模式。标准I/O称块缓冲为“完全缓冲（full buffering）”。 
 
 **大部分情况下，默认的缓冲模式即对于特定场景是最高效的。但是，标准I/O还提供了一个接口，可以修改使用的缓冲模式：
### 4.2 `setvbuf`：最通用的缓冲控制

**函数签名**
```c
#include <stdio.h>
int setvbuf(FILE *stream, char *buf, int mode, size_t size);
```

**参数意义**
- `buf`：
  - 传 `NULL`：由库自动分配缓冲
  - 传非 NULL：你提供一块缓冲（注意生命周期）
- `mode`：
  - `_IONBF`：无缓冲（每次输出尽量立即下沉到内核）
  - `_IOLBF`：行缓冲（遇到 `\n` 刷新，常用于终端）
  - `_IOFBF`：全缓冲（攒满才刷，常用于普通文件）
- `size`：缓冲区大小（`buf!=NULL` 时通常需要 >0）

**返回值**
- 成功：0
- 失败：非 0

**注意**
- 必须在对该流进行 I/O **之前**调用（否则行为不可靠）。

---

### 4.3 `setbuf`：setvbuf 的简化版本

**函数签名**
```c
#include <stdio.h>
void setbuf(FILE *stream, char *buf);
```

**行为**
- `buf == NULL`：无缓冲
- `buf != NULL`：使用你提供的缓冲（大小通常为 `BUFSIZ`）

---

### 4.4 `fflush`：刷新用户态缓冲

**函数签名**
```c
#include <stdio.h>
int fflush(FILE *stream);
```

**返回值**
- 成功：0
- 失败：EOF

**行为**
- 对输出流：把用户态缓冲“冲到内核”（触发 write）
- 对输入流：不同实现略有差异（标准层面不建议用来“清空输入”）

**重要区别**
- `fflush` 不是落盘保证。
- 想更接近“真正落盘”：`fflush(fp)` 后再 `fsync(fileno(fp))`（POSIX）。

---

## 5. 线程安全：flockfile / funlockfile / *_unlocked

### 5.1 先说结论：stdio 大多是“函数级线程安全”
在 POSIX 系统中，典型实现会对每个 `FILE*` 内部加锁，因此：
- 单次 `fprintf/fwrite/fgets` 调用通常是线程安全的（不会把内部结构弄坏）。

但要注意：
- **“多次调用组成的操作序列”不是原子的**。
- 例如你想保证“打印两行必须连续不被别的线程插队”，就需要手动锁流。

### 5.2 `flockfile`：给一个 `FILE*` 加流锁

**函数签名（POSIX）**
```c
#include <stdio.h>
void flockfile(FILE *stream);
```

**作用**
- 对该 `FILE*` 加锁。
- 同一进程内其他线程对该流的 stdio 操作会阻塞，直到解锁。

### 5.3 `ftrylockfile`：尝试加锁（不阻塞）

**签名**
```c
#include <stdio.h>
int ftrylockfile(FILE *stream);
```

**返回值**
- 0：成功拿到锁
- 非 0：没拿到（锁正被其他线程持有）

### 5.4 `funlockfile`：解锁流

**签名**
```c
#include <stdio.h>
void funlockfile(FILE *stream);
```

### 5.5 “对流操作解锁”还可能指 *_unlocked
POSIX 还提供一些不加内部锁的版本（更快但需要你自己保证互斥）：
- `getc_unlocked`, `putc_unlocked`, `fgetc_unlocked`, `fputc_unlocked` 等

典型用法是：
- `flockfile(fp)`
- 期间用 `*_unlocked` 做大量字符级 I/O
- `funlockfile(fp)`
### 1) 默认版本为什么慢一点

比如 `fgetc(fp)` / `fputc(fp)`：  
在多线程里，为了防止两个线程同时操作同一个 `FILE*` 把缓冲区弄乱，库内部通常会**每次调用都加锁/解锁**。

### 2) `*_unlocked` 是什么

`fgetc_unlocked(fp)` / `fputc_unlocked(fp)`：  
功能几乎一样，但**不会做内部加锁**，所以：

- 单线程：更快一点
    
- 多线程：如果两个线程同时用同一个 `FILE*`，就可能乱/崩/数据错
    

### 3) `flockfile / funlockfile` 的意义（你看到的“对流操作解锁”）

它们是 POSIX 提供的**“把锁交给你管理”**的方式：

- `flockfile(fp)`：你手动把这个 `FILE*` 的内部锁**锁住**（一次）
    
- 然后你在这段期间用大量 `*_unlocked` 做 I/O：**不会每次都锁/解锁**，因此更快
    
- `funlockfile(fp)`：你手动把锁**解开**

---

## 6. 手动文件加锁（跨进程）：flock 或 fcntl

> 这类锁一般是 **建议锁（advisory）**：只有遵守同一约定的进程才会配合。

### 6.1 `flock`（简单，常用于整文件锁）

**签名（POSIX/BSD/Linux 常见）**
```c
#include <sys/file.h>
int flock(int fd, int operation);
```

**operation**
- `LOCK_SH`：共享锁（读锁）
- `LOCK_EX`：独占锁（写锁）
- `LOCK_UN`：解锁
- `LOCK_NB`：非阻塞（与 SH/EX 组合）

**返回值**
- 成功：0
- 失败：-1（`errno` 说明原因，如 `EWOULDBLOCK`）

> 与 stdio 配合：你可以 `int fd = fileno(fp); flock(fd, LOCK_EX); ... flock(fd, LOCK_UN);`

---

### 6.2 `fcntl` 记录锁（更复杂，可锁定文件区间）

**签名（核心调用）**
```c
#include <fcntl.h>
int fcntl(int fd, int cmd, ...);
```

常用 `cmd`：
- `F_SETLK`：设置锁（非阻塞）
- `F_SETLKW`：设置锁（阻塞等待）

配合 `struct flock`：
```c
struct flock lk;
memset(&lk, 0, sizeof(lk));
lk.l_type   = F_WRLCK;         // F_RDLCK / F_WRLCK / F_UNLCK
lk.l_whence = SEEK_SET;
lk.l_start  = 0;
lk.l_len    = 0;              // 0 表示锁到 EOF（整文件）
fcntl(fd, F_SETLKW, &lk);
```

---

## 7. 一个“把所有点串起来”的简单示例

目标：
1) 写入一个二进制文件，文件头先占位 8 字节（记录总长度）
2) 写入文本行与二进制块
3) 回头用 `fseek` 回填头部长度
4) 使用：
   - **跨进程文件锁**（flock）保证只有一个进程写
   - **同进程流锁**（flockfile）保证多线程写入序列不插队
   - 检查 `ferror`

> 示例偏教学：真实工程写缓冲/短写/崩溃一致性需要更严谨设计。

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
#include <sys/file.h>

static int write_u64_le(FILE *fp, uint64_t x) {
    unsigned char b[8];
    for (int i = 0; i < 8; i++) b[i] = (unsigned char)((x >> (8*i)) & 0xFF);
    return (fwrite(b, 1, 8, fp) == 8) ? 0 : -1;
}

int main() {
    FILE *fp = fopen("out.bin", "wb+");
    if (!fp) { perror("fopen"); return 1; }

    // 1) 跨进程整文件独占锁（写锁）
    int fd = fileno(fp);
    if (flock(fd, LOCK_EX) < 0) { perror("flock"); fclose(fp); return 1; }

    // 2) 同进程多线程：把一段写入序列变成原子
    flockfile(fp);

    // 2.1 预留 8 字节头（总长度占位）
    if (write_u64_le(fp, 0) < 0) goto io_fail;

    // 2.2 写一行文本
    if (fputs("hello\n", fp) == EOF) goto io_fail;

    // 2.3 写一个字符
    if (fputc('X', fp) == EOF) goto io_fail;
    if (fputc('\n', fp) == EOF) goto io_fail;

    // 2.4 写二进制块
    unsigned char buf[4] = {0xDE, 0xAD, 0xBE, 0xEF};
    if (fwrite(buf, 1, sizeof(buf), fp) != sizeof(buf)) goto io_fail;

    // 3) 计算当前文件长度
    long end = ftell(fp);
    if (end < 0) goto io_fail;

    // 4) 回到开头回填长度
    if (fseek(fp, 0L, SEEK_SET) != 0) goto io_fail;
    if (write_u64_le(fp, (uint64_t)end) < 0) goto io_fail;

    // 5) 刷新用户态缓冲
    if (fflush(fp) == EOF) goto io_fail;

    funlockfile(fp);

    // 6) 解跨进程锁
    if (flock(fd, LOCK_UN) < 0) { perror("flock unlock"); }

    if (fclose(fp) != 0) { perror("fclose"); return 1; }

    return 0;

io_fail:
    // 出错：检查 ferror
    if (ferror(fp)) {
        fprintf(stderr, "I/O error: %s\n", strerror(errno));
    }
    funlockfile(fp);
    flock(fd, LOCK_UN);
    fclose(fp);
    return 1;
}
```

你可以重点观察：
- `fwrite` 的返回是“元素个数”，这里用 `size=1` 把它退化成“字节数”。
- 回填头部：先 `ftell` 取末尾位置，再 `fseek` 回到 0。
- `rewind(fp)` 也能回到开头并清除错误/EOF，但这里需要回填并不想清标志时，用 `fseek` 更显式。
- **流锁**与**文件锁**是两种层级：
  - `flockfile/funlockfile`：线程
  - `flock(LOCK_EX/UN)`：进程

---

## 8. 常见误区速查

1) `fputs` 会写 `\0`：**不会**。
2) `fwrite` 返回字节数：**不是**，返回写入“元素个数”。
3) `fflush` 就等于落盘：**不是**，它只把用户态缓冲冲到内核。
4) 多线程下多次 `fprintf` 组合是原子的：**不是**，要 `flockfile`。
5) `flockfile` 能跨进程：**不能**，跨进程用 `flock/fcntl`。

---

如果你希望“示例更贴近工程”，下一步通常是：
- 把回显/日志写入做成函数，展示 `ftrylockfile` 的非阻塞写法
- 讲清 `a+` 模式下 `fseek` 的限制与追加语义
- 加入 `fsync(fileno(fp))` 演示落盘语义（POSIX）

