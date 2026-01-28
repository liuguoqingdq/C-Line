# Linux 系统编程：监视文件事件（File Event Monitoring）

在 Linux 上，“监视文件事件”通常指：当文件/目录发生 **创建、删除、修改、移动、属性变化** 等操作时，程序能收到通知并做处理。最常用的机制是 **inotify**；更偏安全/审计或拦截访问的机制是 **fanotify**。

> 先给结论：
>
>- **应用层监视（类似 `tail -f`、热加载、文件同步）**：用 **inotify**。
>- **安全/反病毒/访问审计、甚至想在打开/执行前拦截**：考虑 **fanotify**（权限要求更高）。

---

## 1. inotify：最常用的文件事件通知

### 1.1 能监视什么

inotify 对“监视点（watch）”触发事件，常见事件包括：

- `IN_CREATE`：目录中新建条目
- `IN_DELETE`：目录中删除条目
- `IN_MODIFY`：文件内容被修改
- `IN_ATTRIB`：元数据变化（权限/时间戳等）
- `IN_MOVED_FROM` / `IN_MOVED_TO`：移动/重命名（配对）
- `IN_DELETE_SELF`：被监视对象本身被删除
- `IN_MOVE_SELF`：被监视对象本身被移动
- `IN_CLOSE_WRITE`：写入后关闭（监视“写完一次”的好用事件）

### 1.2 核心 API（系统调用接口）

```c
#include <sys/inotify.h>

int inotify_init(void);                 // 旧接口
int inotify_init1(int flags);           // 推荐：可设非阻塞/close-on-exec

int inotify_add_watch(int fd, const char *pathname, uint32_t mask);
int inotify_rm_watch(int fd, int wd);
```

- `fd`：inotify 实例的文件描述符（像一个“事件队列”）
- `wd`：watch descriptor（一个整数，表示某个监视点）
- `mask`：你关心的事件集合（上面 IN_XXX 按位或）

返回值：
- `inotify_init1` 成功返回 `fd`，失败 `-1`
- `inotify_add_watch` 成功返回 `wd`，失败 `-1`

### 1.3 读事件：`read(fd, ...)` 读出 `struct inotify_event` 列表

inotify 事件是可变长结构，必须按 `len` 解析：

```c
struct inotify_event {
    int      wd;//监听点
    uint32_t mask;
    uint32_t cookie;
    uint32_t len;
    char     name[]; // len > 0 时存在（对目录事件常见：条目名）
};
```

- `cookie`：用于把 `IN_MOVED_FROM` 和 `IN_MOVED_TO` 配对
- `name`：如果监视的是目录，很多事件带子项名称。
## 1) `wd`：watch descriptor（监听点编号）

- `wd` 是 `inotify_add_watch(fd, pathname, mask)` 的返回值。
    
- 你可能对同一个 `inotify_fd` 加了多个 watch（多个目录/文件），**靠 wd 区分是哪一个 watch 触发的事件**。
    
- 常见用法：维护一个 `wd -> 被监视路径` 的 map，打印或处理事件时能反查回路径。
    

特别事件：

- 如果你 `inotify_rm_watch(fd, wd)`，内核会产生一个 `IN_IGNORED` 事件（mask 里含 `IN_IGNORED`），告知该 wd 被移除。
    

---

## 2) `mask`：事件类型 + 事件标志（位图）

`mask` 是一个 bitmask（按位或），**同时描述“发生了什么”以及一些附加信息**。

### 2.1 “发生了什么”：典型事件位

常见的有：

- `IN_CREATE`：创建（通常在目录 watch 上出现）
    
- `IN_DELETE`：删除（目录 watch）
    
- `IN_MODIFY`：内容修改
    
- `IN_ATTRIB`：属性变更（权限、时间戳等）
    
- `IN_MOVED_FROM`：从该目录移走/重命名走了（目录 watch）
    
- `IN_MOVED_TO`：移入该目录/重命名进来了（目录 watch）
    
- `IN_CLOSE_WRITE`：以写方式关闭（“写完一次”的好信号）
    
- `IN_OPEN`：被打开
    
- `IN_DELETE_SELF`：被监视对象本身被删
    
- `IN_MOVE_SELF`：被监视对象本身被移动/重命名
    

### 2.2 “附加标志”：常见 flag 位

- `IN_ISDIR`：这次事件对应的是**目录**（配合 `IN_CREATE/IN_MOVED_TO` 很常用：新建目录就给它 add_watch 实现递归）
    
- `IN_Q_OVERFLOW`：事件队列溢出（丢事件了，通常要做一次全量扫描重建状态）
    
- `IN_IGNORED`：某个 watch 被内核移除/失效（或你 rm_watch），该 wd 之后不再有效
    

> 一个事件里 mask 往往不止一个 bit，比如：`IN_CREATE | IN_ISDIR` 表示“创建了一个目录条目”。

---

## 3) `cookie`：配对 “from/to” 的移动事件（关键）

`cookie` 主要用于 **把 `IN_MOVED_FROM` 和 `IN_MOVED_TO` 配成一对**。

典型场景：同一文件系统内 rename/move 会产生两条事件：

- 源目录：`IN_MOVED_FROM`（带 cookie）
    
- 目标目录：`IN_MOVED_TO`（同一个 cookie）
    

你用 cookie 就能知道这是“同一次移动”的两端，从而推导：

- 原路径：`src_dir + oldname`
    
- 新路径：`dst_dir + newname`
    

注意点：

- **只有 move/rename 相关事件**才依赖 cookie（主要是 `IN_MOVED_FROM/IN_MOVED_TO`）
    
- 有时你只看到 `MOVED_FROM` 没看到 `MOVED_TO`（或反之），比如跨文件系统 move（底层是 copy+unlink）、或者目标目录没被 watch、或者队列溢出导致丢事件。工程上要允许“配对失败”。
    

---

## 4) `len`：`name` 字段的长度（包含 `\\0`，且可能有 padding）

- `len` 表示 `name[]` 实际占用的字节数。
    
- **如果 `len == 0`，说明这条事件没有 name**（常见于监视“文件本体”时的一些 self 事件：`IN_DELETE_SELF`、`IN_MOVE_SELF` 等）。
    
- 如果 `len > 0`：
    
    - `name` 是一个以 `\\0` 结尾的字符串（文件名/目录名），长度最多 `len`。
        
    - `len` **包含末尾的 `\\0`**，并且可能包含对齐填充（所以别用 `strlen` 来推进指针，推进要用 `sizeof(struct inotify_event)+len`）。
        

---

## 5) `name[]`：发生事件的“条目名”（相对于被 watch 的目录）

- **只有当你 watch 的是目录**时，`name` 才最有用：它是该目录下发生变化的条目名（如 `a.txt`）。
    
- 它是**相对名字**，不是全路径：你需要用 “watch 对应的目录路径 + '/' + name” 拼出完整路径。
    
- 若你 watch 的是单个文件：
    
    - 很多事件的 `name` 为空（`len=0`），因为事件本身已经指向那个文件了（由 wd 标识）。
        

---

## 6) 正确遍历事件缓冲区的方法（为什么 len 很重要）

你从 `read()` 得到的是多个事件拼在一起，必须这样走：
```C
for (char *p = buf; p < buf + n; ) {
    struct inotify_event *ev = (struct inotify_event*)p;

    // 使用 ev->wd / ev->mask / ev->cookie / ev->name

    p += sizeof(struct inotify_event) + ev->len;
}
```
因为你从 `read(inotify_fd, buf, ...)` 读到的不是“一个事件”，而是一整块缓冲区，里面**顺序拼接了多条 inotify 事件**。每条事件的大小是**可变的**：

- 事件头部固定：`sizeof(struct inotify_event)`
    
- 后面还跟着一个可选的 `name[]` 字段，长度由 `ev->len` 指定（**可能为 0**，也可能包含对齐 padding）
    

所以一条事件在内存里的真实布局是：

```CSS
[ 固定头 struct inotify_event ] [ name 字节 len 个（含 '\0' + padding） ]
```

别用 `sizeof(struct inotify_event)` 固定步长，也别用 `strlen(ev->name)` 推进。
如果wd标识的描述符是个目录，且该目录下的文件发生监视事件，数组name[]就保存和路径相关的文件名。在这种情况下，len值不为零。需要注意的是，len的长度和字符串name的长度不一样，name可以包含多个null字符进行填充，以确保后续的inotify_event能够正确对齐。因此，在计算数组中下一个inotify_event结构体的偏移时，必须使用len，而不能使用strlen()。




------
# 获取事件队列大小

未处理事件队列大小可以通过在inotify实例文件描述符上执行ioctl（参数为FIONREAD）来获取。请求的第一个参数获得以无符号整数表示的队列的字节长度： 

```C
unsigned int queue_len;
int ret;
ret = ioctl(fd,FIONREAD,&queue_len);
if(ret<0){
	perror("ioctl");
}else{
	printf("队列大小%d\n",queue_len);
}
```

# 销毁监听

```C
int ret = close(fd);
if (ret == -1) {
    perror("close");
}
fd = -1;   // 常见习惯：避免后续误用/二次 close

```




---

## 2. 最小可跑示例：监视目录的创建/删除/修改/重命名

> 这个 demo 监视一个目录（比如 `./watched`），打印所有事件。

```c
// inotify_demo.c
#define _GNU_SOURCE
#include <sys/inotify.h>
#include <unistd.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static void show_mask(uint32_t m) {
    struct { uint32_t bit; const char* name; } items[] = {
        {IN_CREATE, "CREATE"}, {IN_DELETE, "DELETE"}, {IN_MODIFY, "MODIFY"},
        {IN_ATTRIB, "ATTRIB"}, {IN_MOVED_FROM, "MOVED_FROM"}, {IN_MOVED_TO, "MOVED_TO"},
        {IN_DELETE_SELF, "DELETE_SELF"}, {IN_MOVE_SELF, "MOVE_SELF"},
        {IN_CLOSE_WRITE, "CLOSE_WRITE"}, {IN_OPEN, "OPEN"}, {IN_CLOSE_NOWRITE, "CLOSE_NOWRITE"},
        {IN_IGNORED, "IGNORED"}, {IN_Q_OVERFLOW, "Q_OVERFLOW"}
    };
    for (size_t i=0;i<sizeof(items)/sizeof(items[0]);i++) {
        if (m & items[i].bit) printf("%s|", items[i].name);
    }
}

int main(int argc, char **argv) {
    const char *path = (argc > 1) ? argv[1] : "./watched";

    int fd = inotify_init1(IN_NONBLOCK | IN_CLOEXEC);
    if (fd < 0) { perror("inotify_init1"); return 1; }

    uint32_t mask = IN_CREATE|IN_DELETE|IN_MODIFY|IN_ATTRIB|
                    IN_MOVED_FROM|IN_MOVED_TO|IN_DELETE_SELF|IN_MOVE_SELF|
                    IN_CLOSE_WRITE;

    int wd = inotify_add_watch(fd, path, mask);
    if (wd < 0) { perror("inotify_add_watch"); return 1; }

    printf("Watching: %s (wd=%d)\n", path, wd);

    char buf[4096] __attribute__((aligned(__alignof__(struct inotify_event))));

    for (;;) {
        ssize_t n = read(fd, buf, sizeof(buf));
        if (n < 0) {
            if (errno == EAGAIN || errno == EINTR) {
                usleep(50 * 1000);
                continue;
            }
            perror("read");
            break;
        }

        for (char *p = buf; p < buf + n; ) {
            struct inotify_event *ev = (struct inotify_event*)p;
            printf("wd=%d mask=0x%x cookie=%u ", ev->wd, ev->mask, ev->cookie);
            show_mask(ev->mask);
            if (ev->len > 0) printf(" name=%s", ev->name);
            putchar('\n');

            p += sizeof(struct inotify_event) + ev->len;
        }
    }

    inotify_rm_watch(fd, wd);
    close(fd);
    return 0;
}
```

编译运行：

```bash
gcc -Wall -Wextra -O2 inotify_demo.c -o inotify_demo
mkdir -p watched
./inotify_demo watched
```

另一个终端做一些操作：

```bash
touch watched/a.txt
echo hi >> watched/a.txt
mv watched/a.txt watched/b.txt
rm watched/b.txt
```

---

## 3. 与 epoll 结合：统一事件循环

inotify 的 `fd` 本质就是一个可读 fd，所以可直接加到 `epoll`：

- `epoll` 监视 `inotify_fd` 的 `EPOLLIN`
- 一旦可读，就 `read()` 拉取所有 `inotify_event`

这种写法适合：网络服务器 + 文件热更新（配置变更自动加载）这种场景。

---

## 4. inotify 的坑点与工程应对

### 4.1 监视目录 ≠ 递归监视子目录

inotify **不自动递归**：

- 你监视 `root/` 只能看到 `root/` 直接子项的事件
- 对 `root/sub/` 必须再单独 add_watch

工程方案：
- 启动时遍历目录树，为每个目录 add_watch
- 监听 `IN_CREATE` 且 `IN_ISDIR`，新建目录后再 add_watch

### 4.2 事件可能丢：`IN_Q_OVERFLOW`

事件队列满了会出现 `IN_Q_OVERFLOW`。应对：

- 及时读（非阻塞 + epoll）
- 溢出后做一次“全量扫描”重建状态

### 4.3 文件被替换（原子写）导致 watch 失效

很多程序更新配置文件用的是：

- 写到临时文件
- `rename(temp, config)` 原子替换

如果你只监视 `config` 本体，可能会遇到：

- 原文件 inode 消失（`IN_DELETE_SELF/IN_MOVE_SELF`）
- 新文件是另一个 inode，需要重新 add_watch

工程建议：
- **监视父目录**，关注 `IN_MOVED_TO`/`IN_CLOSE_WRITE` 对目标文件名
- 或监视文件本体同时处理 self 事件并重建 watch

### 4.4 不能“跨主机/跨网络文件系统可靠”

某些网络文件系统对 inotify 支持不完整或语义不同（取决于 FS/挂载）。如果是 NFS/SMB 场景，要测试确认。

---

## 5. fanotify：更偏安全/审计/拦截

fanotify 更强：可以监视 **文件被 open/读/写/执行** 等访问事件，甚至在某些模式下可阻塞等待用户态决策。

特点：

- 常用于：反病毒、DLP、安全代理
- 通常需要更高权限（CAP_SYS_ADMIN 等）
- 接口更复杂：事件返回的是 fd，需要你读元信息、再 `close()`

如果你只是做“文件变更通知”，优先 inotify。

---

## 6. 你真正想监视的“文件事件”是哪类？（对照表）

- 想知道“配置文件改了，自动 reload”：
  - 监视父目录 + 关注 `IN_CLOSE_WRITE`/`IN_MOVED_TO`
- 想实现类似 `tail -f`：
  - 监视 `IN_MODIFY` 或 `IN_CLOSE_WRITE`，然后再 `read` 新增内容
- 想递归监视项目目录：
  - 遍历 add_watch + 动态补 watch
- 想知道“谁打开了这个文件”：
  - 可能需要 fanotify / audit

---

## 7. 常用调试命令

- 看事件：`inotifywait`（inotify-tools）

```bash
inotifywait -m watched
```

- 看某文件类型：

```bash
stat watched/a.txt
```

---

如果你告诉我你的具体目标（比如“监视某个配置文件变化并热重载”，还是“监视整个目录树并递归同步”），我可以把上面 demo 直接改成 **最贴合你场景的版本**（包含：递归监视、溢出恢复、rename 替换处理、以及 epoll 主循环整合）。

