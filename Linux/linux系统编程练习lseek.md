## 练习 1：回到文件开头再读

### 题目

打开一个文本文件，先读取 10 个字节并打印，然后**使用 `lseek()` 回到文件开头**，再读取 10 个字节并打印。

### 要求

- 使用 `open / read / lseek / close`
    
- 输出两次读取的内容
    

### 考察点

- `SEEK_SET`
    
- file offset 会随 `read()` 移动
    
```C++
//先简单默认不会短读
int fd=open("a.txt",O_RDONLY);
if(fd==-1){
	perror("open");
}
char buf[4096];
char*p=buf;//需要一个指针来移动buf缓冲，数组不能直接移动
ssize_t len=read(fd,p,10);
p+=10;
lseek(fd,0,SEEK_SET);
len=read(fd,p,10);
```


---

## 练习 2：跳过前 N 个字节读取

### 题目

从文件中**跳过前 20 个字节**，然后读取接下来的 15 个字节并打印。

### 要求

- 使用 `lseek(fd, 20, SEEK_SET)`
    
- 不允许用 `read()` 丢弃数据
    

### 考察点

- `SEEK_SET` 的绝对定位含义
    
```C++
//不考虑短读情况
int fd=open("a.txt",O_RDONLY);
lseek(fd,20,SEEK_SET);
char buf[4096];
char* p=buf;
ssize_t len = read(fd,p,15);
```

---

## 练习 3：当前位置相对移动读取

### 题目

先读取 10 个字节，然后再 **从当前位置向前跳 5 个字节**，再读取 10 个字节。

### 要求

- 第二次定位必须使用 `SEEK_CUR`
    
- 打印两次读取的结果
    

### 考察点

- `SEEK_CUR` 是相对“当前 offset”
    
```C++
//补考虑短读
int fd = open("a.txt",O_RDONLY);
char buf[4096];
char*p=buf;
ssize_t len = read(fd,p,10);
p+=10;
lseek(fd,-5,SEEK_CUR);
len=read(fd,p,10);
```

---

## 练习 4：文件尾部倒着读

### 题目

读取文件**最后 20 个字节**并打印。

### 要求

- 使用 `lseek(fd, -20, SEEK_END)`
    
- 不允许读取整个文件再截取
    

### 考察点

- `SEEK_END`
    
- offset 可以为负
    
```C++
int fd = open("a.txt",O_RDONLY);
lseek(fd,-20,SEEK_END);
char buf[1024];
char*p=buf;
ssize_t len = read(fd,buf,20);
```

---

## 练习 5：获取文件大小（经典）

### 题目

使用 `lseek()` 获取文件大小，并打印（单位：字节）。

### 要求

- 使用 `lseek(fd, 0, SEEK_END)`
    
- **不能使用 `stat()`**
    

### 提示

`off_t size = lseek(fd, 0, SEEK_END);`

### 考察点

- `lseek` 返回值的意义
    
```C++
int fd=open("a.txt",O_RDONLY);
off_t len=lseek(fd,0,SEEK_END);
lseek(fd,0,SEEK_SET);
```

---

## 练习 6：读文件中间 10 个字节

### 题目

读取文件**正中间**的 10 个字节。

### 要求

1. 先用 `lseek` 获取文件大小
    
2. 计算中点
    
3. 再 `lseek` 到中点读取
    

### 考察点

- `off_t` 运算
    
- 多次 `lseek` 组合使用
    
```C++
int fd = open("a.txt",O_RDONLY);
off_t len=lseek(fd,0,SEEK_END);
off_t mid=len/2;
lseek(fd,mid,SEEK_SET);
char buf[1024];
char*p=buf;
ssize_t n=read(fd,p,10);
```

---
## 练习 8：定位写（覆盖写）

### 题目

向文件中**第 5 个字节处**写入字符串 `"HELLO"`。

### 要求

- 使用 `lseek` 定位
    
- 不允许重新创建文件
    
- 覆盖原内容（不是插入）
    

### 考察点

- 写操作同样受 offset 影响
    
```C++
int fd=open("a.txt",O_WRONLY);
lseek(fd,5,SEEK_SET);
ssize_t n=write(fd,"HELLO",5);
```

---

## 练习 9：制造“文件空洞”（sparse file）

### 题目

新建一个文件，执行：

1. `lseek(fd, 1024, SEEK_SET)`
    
2. 写入字符串 `"END"`
    

然后观察文件大小。

### 要求

- 用 `ls -lh` 或 `stat` 验证文件大小
    

### 考察点

- 稀疏文件（hole）
    
- `lseek` 可以跳过未写区域
    
```C++
int fd = open("a.txt",O_WRONLY | O_TRUNC |O_CREAT,0644);
lseek(fd,1024,SEEK_SET);
ssize_t len = write(fd,"END",3);
struct stat st;
fstat(fd,&st);
std::cout<<st.st_size<<"\n";
```

---

## 练习 10：验证不可 seek 的文件

### 题目

尝试对 **标准输入（stdin）** 调用 `lseek()`，并打印错误信息。

### 要求

- `fd = 0`
    
- 判断返回值并打印 `errno`
    

### 考察点

- 管道 / 终端 / socket 不支持 seek
    
- 错误处理

```C++
off_t len=lseek(STDIN_FILENO,10,SEEK_SET);
if(len<0){
	perror("lseek");
}
```