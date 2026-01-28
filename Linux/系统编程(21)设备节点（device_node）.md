# Linux 系统编程：设备节点（Device Node）

你问的“设备节点”一般指 `/dev` 下的 **字符设备（character device）** 和 **块设备（block device）** 这两类 **特殊文件**。它们看起来像文件，但本质是内核为设备/驱动暴露出来的访问入口。

---

## 1. 设备节点是什么：不是数据文件，而是“设备的入口”

在 Linux 里，一个路径名对应一个 inode。普通文件的 inode 记录“数据块位置”；而**设备节点**的 inode 不指向普通数据块，而是记录一个设备号：

- **主设备号（major）**：通常用来定位 **哪一个驱动/设备类型**
- **次设备号（minor）**：通常用来区分 **同类设备的不同实例/分区/通道**

你打开一个设备节点（`open("/dev/sda", ...)`）时，内核根据它的 major/minor 把操作分发给相应驱动。

---

## 2. 字符设备 vs 块设备

### 2.1 字符设备（char device）

- 类型：`S_IFCHR`
- 特点：
  - 面向**字节流**（不一定可 seek）
  - 常见：终端 `/dev/tty`、串口 `/dev/ttyS0`、随机数 `/dev/urandom`、GPU/声卡等

### 2.2 块设备（block device）

- 类型：`S_IFBLK`
- 特点：
  - 面向**块 I/O**（通常支持随机访问/seek）
  - 常见：磁盘 `/dev/sda`、分区 `/dev/nvme0n1p1`

> 注意：块设备“通常可随机访问”不等于“你随便读写就安全”。很多块设备写错就直接毁文件系统。

---

## 3. 如何识别设备节点：`stat` 的 st_mode / st_rdev

你用 `stat/lstat/fstat` 得到 `struct stat st` 后：

- 文件类型：`S_ISCHR(st.st_mode)` / `S_ISBLK(st.st_mode)`
- 设备号：`st.st_rdev`（对设备节点有效）
  - `major(st.st_rdev)`
  - `minor(st.st_rdev)`

示例：

```c
#include <sys/stat.h>
#include <sys/sysmacros.h>
#include <stdio.h>

int main(){
    struct stat st;
    if (stat("/dev/null", &st) == -1) return 1;
    if (S_ISCHR(st.st_mode)) {
        printf("char device: major=%u minor=%u\n",
               major(st.st_rdev), minor(st.st_rdev));
    }
}
```

---

## 4. 创建设备节点：`mknod` / `mknodat`

### 4.1 `mknod` 做什么

`mknod` 用于创建特殊文件：

- 设备节点（字符/块设备）
- FIFO（命名管道）

```c
#include <sys/stat.h>
#include <sys/types.h>

int mknod(const char *pathname, mode_t mode, dev_t dev);
```

- **返回值**：成功 `0`；失败 `-1` 并设置 `errno`

### 4.2 参数含义

- `pathname`：要创建的路径
- `mode`：
  - 文件类型位：`S_IFCHR` 或 `S_IFBLK`（设备节点）或 `S_IFIFO`（FIFO）
  - 权限位：例如 `0660`
- `dev`：设备号（对字符/块设备才使用）
  - 用 `makedev(major, minor)` 构造

### 4.3 权限/能力要求（非常重要）

- 创建设备节点通常需要 **特权**：root 或具备 `CAP_MKNOD`。
- 非特权进程一般会 `EPERM`。

### 4.4 `mknodat`

```c
int mknodat(int dirfd, const char *pathname, mode_t mode, dev_t dev);
```

- 相对路径以 `dirfd` 为基准（或 `AT_FDCWD`）。

---

## 5. /dev 下的设备节点是谁创建的？（udev / devtmpfs）

现代 Linux 通常是这样的链条：

1. 内核把设备信息暴露到 sysfs（`/sys`）
2. udev（用户态守护）监听设备事件
3. udev 在 `/dev` 下创建/删除对应设备节点，并设置权限/属主/组（例如 `video`、`dialout`）

因此：

- 你通常 **不需要手写 `mknod`** 去创建常见设备节点
- 程序一般是“使用”设备节点（open/ioctl/read/write）

---

## 6. 设备节点能当普通文件 copy/move 吗？

这是系统编程里很关键的点：

### 6.1 “拷贝内容”通常没有意义甚至危险

如果你对设备节点做 `open + read + write`：

- 读出来的数据是“设备返回的数据”（例如 `/dev/urandom`）
- 写进去是“对设备的写操作”（对块设备写会直接改磁盘扇区）

所以像 `cp` 那样“复制设备节点内容”多数时候是错误的。

### 6.2 正确做法：复制“节点本身”（保留设备号）

当你实现 `cp -a` 类工具要支持设备节点时，正确语义通常是：

- `stat` 源路径，若 `S_ISCHR/S_ISBLK`：
  - 在目标创建一个同类型设备节点
  - `mode` 复制权限
  - `dev` 复制 `st_rdev`

伪代码：

```c
if (S_ISCHR(st.st_mode) || S_ISBLK(st.st_mode)) {
    mknod(dst, (st.st_mode & 07777) | (S_ISCHR? S_IFCHR : S_IFBLK), st.st_rdev);
}
```

### 6.3 移动（rename）对设备节点如何？

- **同一文件系统内 rename**：只是改目录项名字，设备节点本身（inode、st_rdev）不变，安全。
- **跨文件系统 move（copy+unlink）**：
  - 普通文件要复制内容
  - 设备节点应该用 `mknod` 在目标“重建节点”，然后删源节点

---

## 7. 设备节点常见错误与 errno

- `EPERM`：没有权限创建设备节点（缺 CAP_MKNOD）
- `EEXIST`：目标已存在
- `ENOENT`：父目录不存在
- `ENOSPC`：空间不足/ inode 不足（创建节点也需要 inode）

---

## 8. 使用设备节点：open/read/write/ioctl（概览）

设备节点的访问模式通常是：

1. `open("/dev/xxx", flags)`
2. `read/write`（可选）
3. `ioctl(fd, request, ...)` 与驱动进行控制/配置

不同设备的语义完全不同（`ioctl` 的 request 码和参数由驱动定义）。

---

## 9. 一个小演示：显示设备节点 major/minor

```c
#include <sys/stat.h>
#include <sys/sysmacros.h>
#include <stdio.h>
#include <errno.h>

static void show(const char *p){
    struct stat st;
    if (lstat(p, &st) == -1) {
        perror(p);
        return;
    }
    if (S_ISCHR(st.st_mode) || S_ISBLK(st.st_mode)) {
        printf("%s: %s device major=%u minor=%u\n",
               p,
               S_ISCHR(st.st_mode) ? "char" : "block",
               major(st.st_rdev),
               minor(st.st_rdev));
    } else {
        printf("%s: not a device node\n", p);
    }
}

int main(){
    show("/dev/null");
    show("/dev/zero");
    show("/dev/sda");
    return 0;
}
```

---

## 10. 你在“拷贝/移动工具实现”里怎么处理设备节点（实用规则）

- `S_ISREG`：复制内容
- `S_ISLNK`：复制链接本身（`readlink + symlink`）或按需求跟随
- `S_ISDIR`：递归创建目录
- **`S_ISCHR/S_ISBLK`：用 `mknod(dst, ..., st_rdev)` 重建节点**（需要权限）
- `S_ISFIFO`：`mkfifo`/`mknod(...S_IFIFO...)`
- `S_ISSOCK`：一般不复制（Unix 域 socket 文件通常是运行时对象）

如果你想，我可以把你之前的跨文件系统 `mv` demo 扩展成：

- 对普通文件走 copy+unlink
- 对字符/块设备走 mknod 重建 + unlink
- 对 symlink 走 readlink+symlink

这样就更接近真实 `mv/cp -a` 的处理策略。