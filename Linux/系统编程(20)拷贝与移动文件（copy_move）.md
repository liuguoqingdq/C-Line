# Linux 系统编程：拷贝与移动文件（copy / move）

这份讲解从**系统调用语义**出发，解释在 Linux 系统编程里如何“拷贝文件”和“移动文件”，以及它们与 `cp/mv` 在底层做的事分别是什么。

> 先给结论：
>
>- **移动（move）**：优先用 `rename/renameat/renameat2`，同一文件系统内通常是**原子操作**、几乎 O(1)。跨文件系统会 `EXDEV`，只能做“拷贝 + 删除源”。
>- **拷贝（copy）**：本质是“创建目标 + 复制内容 + 复制元数据”。实现路径可用：`read/write`（最通用）或 `copy_file_range`（内核态文件到文件拷贝），以及 reflink（克隆）等优化。

---

## 1. 拷贝与移动到底“拷贝/移动”的是什么？

一个文件不只是内容（data），还包括：

- **数据内容**：文件字节流、空洞（sparse holes）
- **元数据**：权限 mode、属主 uid/gid、时间戳 atime/mtime/ctime、硬链接数、xattr（扩展属性）、ACL、SELinux label 等

系统编程里你需要明确：你的“拷贝”是否要**保留元数据**（类似 `cp -p` / `cp -a`），是否要处理符号链接、硬链接、设备文件、FIFO、目录等特殊类型。

---

## 2. 文件移动（move）：`rename` / `renameat` / `renameat2`

### 2.1 `rename` 做什么

`rename` 的语义是：把旧路径名 `oldpath` 变成新路径名 `newpath`（也可能是覆盖替换）。

```c
#include <stdio.h>

int rename(const char *oldpath, const char *newpath);
```

- **返回值**：成功返回 `0`；失败返回 `-1` 并设置 `errno`

### 2.2 最关键语义（面试/笔试高频）

- **同一文件系统内**：`rename` 通常是**原子**的“目录项更新”（不会出现中间态），速度快。
- **可覆盖**：若 `newpath` 已存在且是非目录，通常会被替换（具体规则与目录/非目录有关）。
- **不能跨文件系统**：跨挂载点会失败，`errno = EXDEV`。

### 2.3 `renameat`（目录 fd + 相对路径）

```c
#include <stdio.h>

int renameat(int olddirfd, const char *oldpath,
             int newdirfd, const char *newpath);
```

- `olddirfd/newdirfd`：解释相对路径的“目录 fd”，可传 `AT_FDCWD` 表示当前工作目录。

### 2.4 `renameat2`（Linux 特有：更强语义）

```c
#define _GNU_SOURCE
#include <stdio.h>

int renameat2(int olddirfd, const char *oldpath,
              int newdirfd, const char *newpath,
              unsigned int flags);
```

常用 `flags`：

- `RENAME_NOREPLACE`：若 `newpath` 已存在则失败（避免覆盖）
- `RENAME_EXCHANGE`：交换两个路径名（双向交换）
- `RENAME_WHITEOUT`：和 overlay/union FS 相关（一般应用少用）

> 工程建议：你要做“安全地移动且不覆盖”，优先 `renameat2(..., RENAME_NOREPLACE)`。

### 2.5 跨文件系统的 move：为什么要“copy + unlink”

因为 `rename` 只能在同一文件系统修改目录项；跨文件系统没有一个 inode 能被“搬过去”，只能：

1. 拷贝源文件到目标文件系统（copy）
2. 确认拷贝成功后删除源（unlink）

这也解释了 `mv` 在跨盘移动时会慢很多。

---

## 3. 文件拷贝（copy）：几条实现路线

### 3.1 最通用：`open + read + write` 循环

**优点**：可移植、语义清晰、你可完全掌控（校验、限速、加密等）。

**缺点**：需要用户态缓冲，CPU/拷贝开销大；处理稀疏文件/元数据比较麻烦。

核心系统调用：

- `open/openat` 打开源/创建目标
- `read` 读取
- `write` 写入（注意短写）
- `fsync`（可选：需要落盘保证时）

### 3.2 内核态文件到文件拷贝：`copy_file_range`

```c
#define _GNU_SOURCE
#include <unistd.h>

ssize_t copy_file_range(int fd_in, off_t *off_in,
                        int fd_out, off_t *off_out,
                        size_t len, unsigned int flags);
```

- **返回值**：
  - `>0`：实际拷贝字节数
  - `0`：到达 EOF
  - `-1`：失败并设置 `errno`

- `off_in/off_out`：可为 `NULL`，表示使用/更新当前文件偏移；非 `NULL` 则使用该偏移并更新它（类似 `pread/pwrite` 的效果）。
- `flags`：目前一般要求为 `0`。

**特点**：在内核里完成“fd 到 fd”的复制，避免数据来回搬运。某些文件系统/内核路径还可能更优化（例如更少数据拷贝）。

**注意**：不同文件系统/不同类型 fd 组合可能不支持，可能报 `EINVAL` 或 `EXDEV` 等，需要回退方案。

### 3.3 “克隆/写时拷贝”（reflink clone）：几乎 O(1) 的“拷贝”

在支持 reflink 的文件系统（如 btrfs、xfs(部分配置)、某些 overlay 情况）里，可以用 ioctl 进行“克隆”：

- 目标文件与源文件共享数据块，修改时才写拷贝（COW），类似“轻量快照”。

典型接口（了解即可，工程里常见）：

- `ioctl(fd_out, FICLONE, fd_in)`

如果你要做高性能备份/版本化，reflink 是很香的路线，但它**依赖文件系统能力**。

### 3.4 `sendfile` 能不能用来文件到文件拷贝？

在 Linux 上 `sendfile` 经典语义是 **file → socket** 的零拷贝发送（常用于静态文件服务器）。

```c
#include <sys/sendfile.h>

ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

对“文件到文件”的拷贝：更常用 `copy_file_range` 或 `splice`/管道组合。

---

## 4. 拷贝时必须考虑的细节

### 4.1 权限与 umask

创建目标文件常用：

```c
int out = open(dst, O_WRONLY|O_CREAT|O_TRUNC, mode);
```

这里的 `mode` 会被进程的 `umask` 掩掉，所以如果你要“完全复制权限”，通常流程是：

1. `open(..., 0600)` 先创建（安全默认）
2. 拷贝完内容后 `fchmod(out, src_mode)` 修正权限
3. 如果需要：`fchown(out, src_uid, src_gid)`（需要权限）

### 4.2 时间戳

拷贝后用：

- `futimens(out, times)` 设置 atime/mtime（纳秒级）

### 4.3 稀疏文件（sparse file）

有些文件包含“空洞”（看起来很大，但实际磁盘占用小）。

- 简单的 `read/write` 会把空洞“实写”成零，导致占用变大。
- `copy_file_range` 或 reflink 可能更容易保留空洞（依赖内核/FS实现）。
- 若你要严格保留稀疏结构，可用 `lseek(fd, ..., SEEK_DATA/SEEK_HOLE)` 分段复制。

### 4.4 符号链接 / 硬链接怎么处理

- 如果源是 symlink：
  - **复制链接本身**：用 `readlink` 读出 target，再 `symlink` 创建（类似 `cp -a`）
  - **复制它指向的文件内容**：跟随链接后当普通文件复制（类似默认 `cp` 行为，视参数）

- 硬链接：如果你要保持“多个文件名共享同一 inode”的关系（`cp -a` 也不一定默认做到），你需要在拷贝整个目录树时维护“inode -> 新路径”的映射。

### 4.5 安全：避免被 symlink 攻击

当目标路径位于不可信目录（例如 `/tmp`）时，攻击者可用 symlink 把你写入重定向到敏感文件。

常用对策：

- 用 `openat(dirfd, name, O_NOFOLLOW | O_CREAT | O_EXCL, ...)`
- 或先 `lstat` 再 `open` 并验证 inode
- 尽量使用 *at 系列（`openat/renameat/unlinkat`）+ 目录 fd 固定住路径解析根

---

## 5. 一个实战级实现：优先 `copy_file_range`，失败回退 `read/write`

> 下面代码只处理“普通文件 → 普通文件”的拷贝（不含目录、设备文件、symlink），但结构是工程里常用的。

```c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <time.h>
#include <unistd.h>

static int xwrite(int fd, const void *buf, size_t n) {
    const char *p = (const char*)buf;
    while (n > 0) {
        ssize_t w = write(fd, p, n);
        if (w < 0) {
            if (errno == EINTR) continue;
            return -1;
        }
        p += (size_t)w;
        n -= (size_t)w;
    }
    return 0;
}

static int copy_data_fallback_rw(int in, int out) {
    char buf[1 << 20]; // 1MB
    for (;;) {
        ssize_t r = read(in, buf, sizeof(buf));
        if (r == 0) return 0;          // EOF
        if (r < 0) {
            if (errno == EINTR) continue;
            return -1;
        }
        if (xwrite(out, buf, (size_t)r) < 0) return -1;
    }
}

static int copy_data_try_copy_file_range(int in, int out) {
    // 用 stat 拿到大小，便于循环拷贝
    struct stat st;
    if (fstat(in, &st) < 0) return -1;
    if (!S_ISREG(st.st_mode)) {
        errno = EINVAL;
        return -1;
    }

    off_t off_in = 0, off_out = 0;
    off_t remain = st.st_size;

    while (remain > 0) {
        size_t chunk = (remain > (off_t)(1 << 30)) ? (1U << 30) : (size_t)remain; // <=1GB
        ssize_t n = copy_file_range(in, &off_in, out, &off_out, chunk, 0);
        if (n == 0) return 0; // EOF
        if (n < 0) {
            if (errno == EINTR) continue;
            return -1;
        }
        remain -= n;
    }
    return 0;
}

int copy_file(const char *src, const char *dst, int preserve_meta) {
    int in = -1, out = -1;
    struct stat st;

    in = open(src, O_RDONLY);
    if (in < 0) return -1;
    if (fstat(in, &st) < 0) goto fail;
    if (!S_ISREG(st.st_mode)) { errno = EINVAL; goto fail; }

    // 先用安全权限创建，避免受 umask/权限细节影响
    out = open(dst, O_WRONLY | O_CREAT | O_TRUNC, 0600);
    if (out < 0) goto fail;

    // 先尝试内核态拷贝
    if (copy_data_try_copy_file_range(in, out) < 0) {
        // 某些组合/文件系统不支持则回退
        int saved = errno;
        if (saved == EXDEV || saved == EINVAL || saved == ENOSYS || saved == EPERM) {
            // 回到文件开头
            if (lseek(in, 0, SEEK_SET) < 0) goto fail;
            if (lseek(out, 0, SEEK_SET) < 0) goto fail;
            if (ftruncate(out, 0) < 0) goto fail;
            if (copy_data_fallback_rw(in, out) < 0) goto fail;
        } else {
            errno = saved;
            goto fail;
        }
    }

    // 需要“确保持久化”时可以 fsync（注意性能）
    // if (fsync(out) < 0) goto fail;

    if (preserve_meta) {
        // 权限
        if (fchmod(out, st.st_mode & 07777) < 0) goto fail;

        // 属主（可能需要 root）
        // if (fchown(out, st.st_uid, st.st_gid) < 0) goto fail;

        // 时间戳（atime/mtime）
        struct timespec ts[2];
        ts[0] = st.st_atim;
        ts[1] = st.st_mtim;
        if (futimens(out, ts) < 0) goto fail;
    }

    close(in);
    close(out);
    return 0;

fail:
    {
        int e = errno;
        if (in >= 0) close(in);
        if (out >= 0) close(out);
        errno = e;
        return -1;
    }
}

int move_file(const char *src, const char *dst) {
    if (rename(src, dst) == 0) return 0;
    if (errno != EXDEV) return -1; // 不是跨文件系统就直接失败返回

    // 跨文件系统：copy + unlink
    if (copy_file(src, dst, 1) < 0) return -1;
    if (unlink(src) < 0) return -1;
    return 0;
}

int main(int argc, char **argv) {
    if (argc < 4) {
        fprintf(stderr, "Usage: %s [copy|move] SRC DST\n", argv[0]);
        return 2;
    }

    if (strcmp(argv[1], "copy") == 0) {
        if (copy_file(argv[2], argv[3], 1) < 0) {
            fprintf(stderr, "copy failed: %s\n", strerror(errno));
            return 1;
        }
    } else if (strcmp(argv[1], "move") == 0) {
        if (move_file(argv[2], argv[3]) < 0) {
            fprintf(stderr, "move failed: %s\n", strerror(errno));
            return 1;
        }
    } else {
        fprintf(stderr, "Unknown op\n");
        return 2;
    }

    return 0;
}
```

编译：

```bash
gcc -Wall -Wextra -O2 file_copy_move.c -o fcm
```

---

## 6. 你在笔试里可以这样答（高分要点）

- **move**：同 FS 内用 `rename`，原子、快；跨 FS 返回 `EXDEV`，只能 copy+unlink。
- **copy**：创建目标 + 复制数据 + 复制元数据。数据复制可用 `read/write`（通用）或 `copy_file_range`（内核态优化），reflink 可接近 O(1)（依赖 FS）。
- 注意：权限与 `umask`、短写、`EINTR`、稀疏文件、symlink 安全、以及保留时间戳/属主/xattr。

---

如果你还想把这部分扩展成“拷贝/移动整个目录树”（递归），我可以继续补：

- `nftw`/`fts` 的遍历方式
- 目录创建与权限策略
- 保留 symlink、硬链接关系
- 原子替换（临时文件 + `renameat2(NOREPLACE)`）与崩溃一致性（`fsync`/`fdatasync`/目录 `fsync`）

