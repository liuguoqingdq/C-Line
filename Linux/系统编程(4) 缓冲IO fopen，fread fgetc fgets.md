# C 标准库文件流（FILE*）详细教学

这份教程按“从零到能写能读、能处理二进制、知道什么时候该用 fdopen”的顺序讲：

- `fopen()`：通过路径打开文件，得到 `FILE*`
- `fdopen()`：把**已有文件描述符 fd** 关联成 `FILE*`
- 从 `FILE*` 读取：文本/二进制两套思维
- 错误处理与 EOF 判断
- 关闭：`fclose()` 的语义与注意事项

---

## 1. 先统一概念：fd vs FILE*

- **fd（file descriptor）**：内核对象的句柄，`open/read/write/close` 用它。
- **FILE\***：C 标准库在用户态封装的“文件流”，提供缓冲（buffering）与格式化 I/O（`fprintf/fscanf`）。

两者关系：
- `FILE*` 内部通常持有一个 fd（可用 `fileno(fp)` 取出来）
- `FILE*` 还有自己的用户态缓冲区（所以 `fwrite` 之后数据可能还在用户态缓冲里，未必立刻 `write`）

---

## 2. fopen：按路径打开文件

### 2.1 基本原型
```c
#include <stdio.h>

FILE *fopen(const char *path, const char *mode);
```
- 成功：返回 `FILE*`
- 失败：返回 `NULL`

### 2.2 mode 字符串怎么写（常用）

|mode|含义|文件不存在|是否截断|读写|
|---|---|---|---|---|
|`"r"`|只读|失败|否|读|
|`"w"`|只写|创建|是（清空）|写|
|`"a"`|追加写|创建|否|写（总在末尾）|
|`"r+"`|读写|失败|否|读+写|
|`"w+"`|读写|创建|是|读+写|
|`"a+"`|读写追加|创建|否|读+写（写总在末尾）|

**二进制模式**：在 mode 后加 `b`（Linux 上 `b` 基本无影响，但在 Windows 上会影响换行转换）。
- `"rb"`、`"wb"`、`"ab"`、`"r+b"` 等。

### 2.3 打开失败怎么查原因
```c
FILE *fp = fopen("data.txt", "r");
if (!fp) {
    perror("fopen");
    return 1;
}
```
- `perror()` 会结合 `errno` 打印原因。

---

## 3. 从 FILE* 读取文本数据（3 种常用方式）

### 3.1 按行读取：fgets（最推荐的文本读取方式）
```c
char line[1024];
while (fgets(line, sizeof(line), fp)) {
    // line 包含这一行（可能带 '\n'）
    fputs(line, stdout);
}

if (ferror(fp)) {
    perror("fgets");
}
```
优点：
- 不会溢出（给了缓冲大小）
- 容易处理包含空格的行
函数签名：
```C
char* fgets(char *buf,int size,FILE* stream);
```
该函数从stream中读取 size-1个字节的数据，并把结果保存到str中。 读完最后一个字节后，缓冲区中会写入空字符（\0）。当读到EOF或换行符时，会结束读。如果读到换行符，会把\n写入到str中。 
 fgets()成功时，返回str；失败时，返回NULL.

```C
if(!fgets(line,sizeof(line),fp)) //error...
```

### 3.2 按格式读取：fscanf（方便但坑多）
```c
int id; char name[64];
while (fscanf(fp, "%d %63s", &id, name) == 2) {
    printf("id=%d name=%s\n", id, name);
}
```
注意点：
- `%s` 默认以空白分隔，读不了带空格的字符串
- 失败时要检查返回值，别只靠 `feof`

### 3.3 按字符读取：fgetc（适合做词法/状态机）
```c
int ch;
while ((ch = fgetc(fp)) != EOF) {
    // ch 是 unsigned char 扩展后的 int
}
if (ferror(fp)) perror("fgetc");
```
函数签名：
```C
int fgetc(FILE* stream);
```

```C
int c = fgetc(fd);
if(c==EOF){
	perror("error");
}
printf("C is %c",(char)c);
```

**该函数从stream中读取一个字符，并把该字符强制类型转换成unsigned int返回。强制类型转换是为了能够表示文件结束或错误：在这两种情况下都会返回EOF。fgetc()的返回值必须保存成int类型。把返回值保存为char类型是经常犯的危险错误，因为这么做会漏掉错误检查。**

**标准输入输出提供了一个函数可以把字符放回到流中。当读取流的最后一个字符，如果不需要该字符的话，可以把它放回流中.

函数签名：
```C
int ungetc(int c,FILE* stream);
```
*每次调用会把int c强制类型转换成unsigned char，并放回到stream中。成功时，返回c；失败时，返回EOF。stream的下一次读请求会返回c。如果有多个字符被放回流中，读取时会以相反的顺序返回——也就是说，最后放回的字符先返回。C标准指出，只保证一次放回成功，而且还必须中间没有读请求。因此，有些实现只支持一次放回。只要有足够的内存，Linux允许无限次放回。当然，一次放回总是成功。 

**如果在调用ungetc()之后，但在发起下一个读请求之前，发起了一次seek函数调用（见3.9节），会导致所有放回stream中的字符被丢弃。在单个进程的多线程场景中会发生这种情况，因为所有的线程共享一个缓冲区.

---

## 4. 读取二进制数据：fread

对于某些应用，每次读取一个字符或一行是不够的。在某些场景下，开发人员希望读写复杂的二进制数据，比如C结构体。为了解决这个entire，标准I/O库提供了fread()函数:
**函数签名：
```C
size_t fread(void* buf,size_t size,size_t nr,FILE* stream);
```
二进制读取的核心是：
- 文本读取是“字符/行/格式”，
- 二进制读取是“字节块/结构体/数组”。

**调用fread()会从stream中读取nr项数据，每项size个字节，并将数据保存到buf所指向的缓冲区。文件指针向前移动读出数据的字节数。 

 返回读到的数据项的个数（注意：不是读入字节的个数!）。如果读取失败或文件结束，fread()函数会返回一个比nr小的数。不幸的是，必须使用ferror()或feof()函数（见3.11节），才能确定是失败还是文件结束。 
 
 由于变量大小、对齐、填充、字节序这些因素的不同，由一个应用程序写入的二进制文件，另一个应用程序可能无法读取，甚至同一个应用程序在另一台机器上也无法读取.
### 4.1 读取一个固定长度块
```c
unsigned char buf[4096];
size_t n = fread(buf,sizeof(buf),1,fp);
if (n == 0) {
    if (feof(fp)) {
        // 正常 EOF
    } else if (ferror(fp)) {
        perror("fread");
    }
}
```

### 4.2 循环读取整个文件（二进制流式读取模板）
```c
unsigned char buf[4096];
for (;;) {
    size_t n = fread(buf,sizeof(buf),1,fp);
    if (n > 0) {
        // 处理 buf[0..n-1]
    }
    if (n < sizeof(buf)) {
        if (feof(fp)) break;
        if (ferror(fp)) { perror("fread"); break; }
    }
}
```

### 4.3 读结构体：可以，但要非常小心
```c
typedef struct {
    int32_t id;
    double  score;
} Record;

Record r;
if (fread(&r, sizeof(r), 1, fp) == 1) {
    // 读到了一个 Record
}
```
**注意（很重要）**：直接读结构体只有在以下条件都满足时才可靠：
- 生成文件和读取文件的程序：同编译器/同 ABI/同对齐规则
- 结构体无 padding（或你明确知道 padding 的布局）
- 浮点/整数大小一致
- 字节序一致（小端/大端）

更稳妥做法（跨平台文件格式）通常是：
- 按字段定义固定字节序（比如小端）
- 用 `uint32_t` 等固定宽度类型
- 逐字段 `fread` 或读入字节后手动解析

---

## 5. fdopen：把已有 fd 变成 FILE*

### 5.1 为什么需要 fdopen
当你已经有一个 fd（例如来自 `open()`、`socket()`、`pipe()`、`dup()`），但你想用 stdio 的：
- 缓冲
- `fprintf/fgets` 等便利接口
就可以用 `fdopen()` 把 fd 关联成 `FILE*`。

### 5.2 原型
```c
#include <stdio.h>

FILE *fdopen(int fd, const char *mode);
```

### 5.3 示例：open + fdopen
```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int fd = open("data.txt", O_RDONLY);
if (fd < 0) { perror("open"); return 1; }

FILE *fp = fdopen(fd, "r");
if (!fp) {
    perror("fdopen");
    close(fd); // fdopen 失败你要自己 close
    return 1;
}

char line[256];
while (fgets(line, sizeof(line), fp)) {
    fputs(line, stdout);
}

fclose(fp); // 关键：fclose 会关闭底层 fd
```

### 5.4 **所有权规则（必须记住）**
- `fdopen(fd, ...)` 成功后：`FILE*` **接管**这个 fd。!!!!!!
- 以后应该 `fclose(fp)`，而不是 `close(fd)`。
- **不要同时 fclose 和 close 同一个底层 fd**，会导致二次关闭。

如果你想让 `FILE*` 不“吃掉”原 fd：
- 先 `dup()` 一份，再 `fdopen(dupfd, ...)`。
```c
int dupfd = dup(fd);
FILE *fp = fdopen(dupfd, "r");
// fclose(fp) 只会关 dupfd，不会关原 fd
```

### 5.5 mode 必须兼容 fd 的打开方式
例如：
- fd 是 `O_RDONLY` 打开的，用 `fdopen(fd, "w")` 会失败。

---

## 6. 从 FILE* 拿回 fd：fileno

```c
#include <stdio.h>
#include <unistd.h>

int fd = fileno(fp);
// 然后你可以对这个 fd 调用 fsync/pread 等
```

常见用途：
- `fflush(fp); fsync(fileno(fp));`（想更接近落盘）

---

## 7. 关闭文件流：fclose（以及 flush 语义）

### 7.1 fclose 做了什么
- 刷新用户态缓冲（相当于隐含 `fflush`）
- 释放 `FILE*` 相关资源
- **关闭底层 fd**

```c
if (fclose(fp) != 0) {
    perror("fclose");
}
```

### 7.2 fflush vs fsync（非常常见的误解）
- `fflush(fp)`：把 **用户态缓冲** 冲到内核（触发 write），不保证落盘
- `fsync(fileno(fp))`：请求内核把数据/元数据刷到稳定存储（更接近落盘）

典型“更稳写入”顺序：
```c
fwrite(buf, 1, len, fp);
fflush(fp);
fsync(fileno(fp));
```

---

## 8. 一个“完整小练习”：读文本 + 读二进制（两套模板）

### 8.1 读文本按行打印
```c
FILE *fp = fopen("a.txt", "r");
if (!fp) { perror("fopen"); return 1; }

char line[1024];
while (fgets(line, sizeof(line), fp)) {
    fputs(line, stdout);
}
if (ferror(fp)) perror("fgets");

fclose(fp);
```

### 8.2 读二进制文件并统计字节数
```c
FILE *fp = fopen("a.bin", "rb");
if (!fp) { perror("fopen"); return 1; }

unsigned char buf[4096];
unsigned long long total = 0;
for (;;) {
    size_t n = fread(buf, 1, sizeof(buf), fp);
    total += n;
    if (n < sizeof(buf)) {
        if (feof(fp)) break;
        if (ferror(fp)) { perror("fread"); break; }
    }
}

printf("total=%llu\n", total);

fclose(fp);
```

---

## 9. 你可以继续问我的两个方向（建议）

1) **写二进制文件格式**：如何定义 header/length/checksum，如何避免结构体 padding 与字节序问题
2) **把 socket/pipe 用 fdopen 变成流**：例如对管道用 `fgets`，对 socket 用 `fprintf`，以及为何要小心缓冲（交互式协议常要 `fflush`）

