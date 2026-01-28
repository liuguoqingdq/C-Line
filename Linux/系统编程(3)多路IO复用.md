
## 1)文件的截断
## 1) 原型

```C
#include <unistd.h>

int truncate(const char *path, off_t length);
int ftruncate(int fd, off_t length);
```
- 成功返回 `0`
    
- 失败返回 `-1` 并设置 `errno`
    

---

## 2) 截断/扩展时到底会发生什么

### 把文件变小（截断）

- 文件新长度变为 `length`
    
- 超过 `length` 的那部分数据**被丢弃**（不可恢复）
    

### 把文件变大（扩展）

- 文件长度变为 `length`
    
- 新增长的区域读取时表现为 **`\0`（全 0）**
    
- 在很多文件系统上可能形成 **稀疏文件（sparse hole）**：逻辑上有那么大，但未必真的分配了磁盘块（取决于 FS）
    

---

## 3) `truncate` vs `ftruncate` 的核心区别

### `truncate(path, len)`（按路径）

- 你给一个 **文件路径**，内核按路径找到文件并改大小
    
- 典型用途：你手里没有 fd，只想“把某个文件改成某个大小”
    

### `ftruncate(fd, len)`（按 fd）

- 你先 `open()` 拿到 **文件描述符 fd**，再改大小
    
- 典型用途：文件可能被 **rename/unlink**，路径不可靠，但 fd 仍指向同一个 inode；或者你想避免“路径被替换”的竞态
    

**一句话：`ftruncate` 更“稳”（避免 TOCTOU），也能对“已打开但已改名/已删除”的文件生效；`truncate` 更“方便”（只靠路径）。**

---

## 4) 权限/前置条件（非常常见的坑）

- `truncate()`：通常要求对该文件有**写权限**（以及路径可达）
    
- `ftruncate()`：通常要求 `fd` 以**可写方式打开**（`O_WRONLY` 或 `O_RDWR`）
    

---

## 5) 细节与坑点

- **文件当前 offset 不会被自动改**。如果你原来的 offset 在新 EOF 之后，之后 `read` 会直接返回 0（EOF）。
    
- 对 mmap 的区域做 truncate 可能导致访问出错（如 SIGBUS），需要小心同步设计。
    
- 目录一般不能被 truncate（会失败）。
    

---

## 6) 最常见用法示例

### 清空日志文件（按路径）

`if (truncate("/var/log/app.log", 0) < 0) perror("truncate");`

### 把已打开文件缩到 0（更安全）

```C
int fd = open("app.log", O_WRONLY);
ftruncate(fd, 0);
lseek(fd, 0, SEEK_SET);   // 常见：把 offset 也归零
```


# 2)多路复用IO
[[C++网络编程select详解]]
[[epoll()系统调用]]
[[Linux多路复用：poll()]]
