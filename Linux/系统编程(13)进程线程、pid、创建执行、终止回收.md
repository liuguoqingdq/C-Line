# 5.1 程序、进程和线程

本节目标：把“磁盘上的程序”如何变成“内存里的进程”，以及进程内如何并发出“多个线程”，用**操作系统视角 + 系统调用视角**讲清楚。

---

## 5.1.1 三者的定义与区别

### 1）程序（Program）
- **静态概念**：磁盘上的可执行文件（如 ELF）或脚本文件。
- 典型内容：机器指令、只读数据、全局变量初值、动态链接信息等。
- 关键点：程序本身**不会运行**；只有被装载到内存并被内核调度，才会“跑起来”。

### 2）进程（Process）
- **动态概念**：程序的一次运行实例。
- 进程至少包含：
  - **独立的虚拟地址空间**（代码段、数据段、堆、栈、映射区）
  - **进程资源**：打开的文件描述符（fd）、信号处理方式、当前工作目录、环境变量等
  - **执行上下文**：寄存器状态（PC/IP、SP 等）、调度信息

> 直观类比：程序是“菜谱”，进程是“正在做的一锅菜（含锅、火、食材状态）”。

### 3）线程（Thread）
- 线程是进程内的执行流。
- **线程共享进程资源**：地址空间、打开的 fd、全局变量等。
- **线程私有**：寄存器上下文、线程栈、线程局部存储（TLS）等。

> 直观类比：进程是一间厨房；线程是厨房里多个厨师共享食材/工具并同时做事。

#### 分配进程ID 
 缺省情况下，内核将进程ID的最大值设置为32 768，这是为了和老的UNIX系统兼容，因为这些系统使用了有符号16位数来表示进程ID。系统管理员可以通过修改/proc/sys/kernel/pid_max把这个值设置成更大的值，但是会牺牲一些兼容性。 
 内核分配进程ID是以严格的线性方式执行的。如果当前pid的的最大值是17，那么分配给新进程的pid值就为18，即使当新进程开始运行时，pid为17的进程已经不再运行了。内核分配的pid值达到了/proc/sys/kernel/pid_max之后，才会重用以前已经分配过的pid值。因此，尽管内核不保证长时间的进程ID的唯一性，但这种分配方式至少可以保证pid在短时间内是稳定且唯一的。

---

## 5.1.2 Linux 视角：进程 = 任务（task），线程 = 轻量级进程

Linux 内核里统一用 `task_struct` 表示可调度实体。
- **同一进程的多个线程**：属于同一个线程组（thread group）。
- 用户态常见现象：
  - `getpid()` 对同一进程内的所有线程返回相同值（线程组 ID，TGID）
  - 每个线程有自己的 TID（线程 ID，Linux 特有），可通过 `gettid`（系统调用）查看

---

## 5.1.3 进程的“资源视图”：你必须熟悉的 4 个核心点

### 1）虚拟内存布局（典型）
```
高地址
+--------------------+
|        栈 stack     |
+--------------------+
|   映射区 mmap       |  (共享库、mmap 文件、匿名映射)
+--------------------+
|        堆 heap      |
+--------------------+
|  数据段 data/bss    |
+--------------------+
|  代码段 text/rodata |
+--------------------+
低地址
```

### 2）文件描述符（fd）
- 每个进程有一张 fd 表（0/1/2 默认：stdin/stdout/stderr）。
- fork 后：子进程继承 fd（指向同一“打开文件表项”，共享文件偏移）。
- exec 后：默认 fd 仍保留，除非设置了 `FD_CLOEXEC`（close-on-exec）。

### 3）信号（signal）
- 异步事件机制（如 `SIGINT`、`SIGTERM`、`SIGCHLD`）。
- 终止进程与子进程回收（5.4）高度相关。

### 4）调度与上下文切换
- 进程/线程是内核调度单位。
- CPU 时间片切换时保存/恢复寄存器上下文。

---

## 5.1.4 常见并发模型（考试/面试常问）

### A. 多进程模型（fork 多份）
- 优点：隔离强，崩一个不崩全。
- 缺点：进程间通信（IPC）成本更高。

### B. 多线程模型（pthread）
- 优点：共享内存，通信成本低。
- 缺点：数据竞争、锁、死锁、崩溃影响整个进程。

### C. 事件驱动 + 单线程（epoll 等）
- 优点：减少锁，吞吐可高。
- 缺点：业务逻辑复杂、阻塞点必须控制。

---

# 5.2 进程 ID（PID）

本节目标：搞清 PID / PPID / PGID / SID / TID 的含义与常用系统调用。

---

## 5.2.1 pid_t 与 PID 的基本概念
- PID 类型：`pid_t`（一般是有符号整数类型）。
- PID 在系统范围内唯一（在 PID 命名空间下是“命名空间内唯一”）。
- PID 会复用：某进程退出后，其 PID 未来可能再次分配给新进程。

---

## 5.2.2 常用获取 ID 的系统调用

## 什么是进程组 PGID

**进程组**就是把一批相关进程“捆在一起”，方便：

- **一把发信号**（例如 Ctrl+C 让整条管道都停掉）
    
- **前台/后台控制**（谁能读终端、谁是前台任务）
    

POSIX 也明确说 `setpgid()` 的目的就是把进程分组，用于**信号、前后台、作业控制**。 [man7.org](https://man7.org/linux/man-pages/man3/setpgid.3p.html?utm_source=chatgpt.com)


**进程组组长（process group leader）**就是：**一个进程组里，PID 恰好等于该进程组的 PGID 的那个进程**。

- **进程组 ID（PGID）**是一个数字，用来标识一个进程组。
    
- **组长**不是“权限更大/能管别人”的意思，它更像是“这个组的命名者/创建者”：通常创建新进程组时，会让某个进程的 **PID 作为 PGID**，于是它就成了组长。
    

### 如何产生进程组组长

最常见两种方式：

1. **默认继承**  
    子进程 fork 出来后，默认和父进程在同一个进程组（所以通常不是新组长）。
    
2. **创建新进程组**（常见于 shell 做作业控制）
    

`setpgid(0, 0);`

这句的意思是：把“当前进程(pid=0)”放到“PGID=0（等价于用自己的 PID 当 PGID）”的组里。结果就是：

- PGID = PID
    
- 该进程成为**进程组组长**
    

### 它有什么“实际意义”？

主要是一些规则会用到它的身份，最典型的是 **`setsid()` 的限制**：

- `setsid()` 用来创建新会话（session）。
    
- 但如果调用者已经是**进程组组长**，`setsid()` 会失败（返回 -1，`errno=EPERM`）。
    

所以守护进程经常先 `fork()` 一次，让子进程 **不是**进程组组长，再在子进程里调用 `setsid()` 创建新会话。

---

## 什么是会话 SID

**会话**是一组进程组的集合，通常对应“一次登录/一个交互式 shell 环境”。会话与**控制终端（controlling terminal）**绑定，用来实现：

- 会话里**只能有一个前台进程组**（foreground process group）
    
- 终端产生的信号（例如 Ctrl+C → SIGINT）会发给**前台进程组**
    
- **只有前台进程组**可以从终端 `read()`；后台进程组去读会收到 `SIGTTIN` 而被挂起 [man7.org](https://man7.org/linux/man-pages/man2/getpgrp.2.html)
    

控制终端通常由**会话首领**首次打开某个终端设备时建立（除非 `open()` 时指定 `O_NOCTTY`），并且一个终端最多只能是一个会话的控制终端。 [man.he.net](https://man.he.net/man7/credentials?utm_source=chatgpt.com)

---

## 这几个函数分别干什么用

### `pid_t getpgrp(void);`

- 返回**当前进程**所在的**进程组 ID（PGID）**（POSIX 版本无参数）。 [man7.org](https://man7.org/linux/man-pages/man2/getpgrp.2.html)
    
- 常用：打印/调试、判断自己在哪个 job 里。
    

### `pid_t getpgid(pid_t pid);`

- 返回 **pid 指定进程**的 PGID；`pid==0` 表示“当前进程”。 [man7.org](https://man7.org/linux/man-pages/man2/getpgrp.2.html)
    
- 适用：你需要查询**别的进程**属于哪个组（但多数时候用 `getpgrp()` 就够了）。 [manpages.debian.org](https://manpages.debian.org/testing/manpages-dev/getpgid.2.en.html)
    

### `pid_t getsid(pid_t pid);`

- 返回 **pid 指定进程**的**会话 ID（SID）**；用于判断某进程属于哪个会话。 [man7.org](https://man7.org/linux/man-pages/man7/credentials.7.html?utm_source=chatgpt.com)
    

### `int setpgid(pid_t pid, pid_t pgid);`

- 把 `pid` 这个进程放进 `pgid` 这个进程组：
    
    - `pid==0`：操作当前进程
        
    - `pgid==0`：让 PGID 变成 `pid`（常用来**创建一个新进程组**） [man7.org](https://man7.org/linux/man-pages/man2/getpgrp.2.html)
        
- 典型用途（**shell 必备**）：创建管道 `a | b | c` 时，让 a/b/c 都进入同一个 PGID，这样 Ctrl+C / Ctrl+Z 能“一次作用整条管道”。man page 也明确点名 `bash` 用 `setpgid()/getpgrp()` 实现作业控制。 [man7.org](https://man7.org/linux/man-pages/man2/getpgrp.2.html)
    
- 常见限制/坑：
    
    - 不能把进程移到**不同会话**的进程组里（否则 `EPERM`）。 [man7.org](https://man7.org/linux/man-pages/man2/getpgrp.2.html)
        
    - 父进程给子进程 `setpgid` 这事一般要在**子进程 `execve()` 之前**完成；如果子进程已经 `execve()`，可能 `EACCES`。 [man7.org](https://man7.org/linux/man-pages/man2/getpgrp.2.html)
        

### `pid_t setsid(void);`

- **创建新会话**（前提：调用者**不能**已经是进程组组长，否则失败）。 [man7.org+1](https://man7.org/linux/man-pages/man2/setsid.2.html?utm_source=chatgpt.com)
    
- 成功后：
    
    - 调用进程成为**会话首领**，SID = 自己 PID
        
    - 同时也成为一个**新进程组**的组长，PGID = 自己 PID [man7.org](https://man7.org/linux/man-pages/man2/setsid.2.html?utm_source=chatgpt.com)
        
    - 并且**不再有控制终端**（这就是做守护进程“脱离终端”的关键一步）。 [pubs.opengroup.org+1](https://pubs.opengroup.org/onlinepubs/000095399/functions/setsid.html?utm_source=chatgpt.com)
        
- 守护进程常见流程里用它，是为了避免终端 hangup 等影响；比如终端挂断时可能引发 `SIGHUP` 相关行为。 [Ubuntu Manpages](https://manpages.ubuntu.com/manpages//bionic/en/man2/setsid.2.html?utm_source=chatgpt.com)

### 1）获取本进程 PID、父进程 PID
```c
#include <sys/types.h>
#include <unistd.h>

pid_t getpid(void);
pid_t getppid(void);//父进程
```
- 返回值：成功返回 PID/PPID，失败一般不会（几乎总成功）。

### 2）进程组与会话（作业控制相关）
```c
#include <sys/types.h>
#include <unistd.h>

pid_t getpgid(pid_t pid); // pid==0 表示当前进程
pid_t getpgrp(void);      // 当前进程组 ID
pid_t getsid(pid_t pid);  // 获取会话 ID

int setpgid(pid_t pid, pid_t pgid);
pid_t setsid(void);       // 创建新会话（守护进程常用）
```
- 进程组（PGID）：shell 作业控制（前台/后台）基本单位。
- 会话（SID）：一个会话可包含多个进程组。

### 3）线程 ID（Linux 特有）
- POSIX 线程 ID：`pthread_t pthread_self(void);`（不是整数 PID）。
- Linux 内核 TID：`gettid`（不是 POSIX 标准）：
```c
#include <sys/syscall.h>
#include <unistd.h>
pid_t tid = (pid_t)syscall(SYS_gettid);
```
> 常见考点：**同一进程不同线程** `getpid()` 一样，但 `gettid()` 不同。

---

## 5.2.3 /proc 视角理解 PID
- `/proc/<pid>/`：进程信息（cmdline、status、fd、maps 等）。
- `/proc/<pid>/task/<tid>/`：线程级信息。

---

## 5.2.4 示例：打印 PID/PPID/TID
```c
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>
#include <sys/syscall.h>

static pid_t gettid_linux(void) {
    return (pid_t)syscall(SYS_gettid);
}

int main(void) {
    printf("PID=%d PPID=%d TID=%d\n", (int)getpid(), (int)getppid(), (int)gettid_linux());
    return 0;
}
```
编译：
```bash
gcc -Wall -Wextra -O2 -g pid_demo.c -o pid_demo
```

---

# 5.3 运行新进程（创建 / 执行）

本节目标：掌握“**创建一个子进程**”与“**在子进程里执行新程序**”的标准套路，并理解 fork/exec/wait 的配合。

---

## 5.3.1 总览：经典流程图

```
父进程
  |
  | fork()
  v
子进程（得到父进程的地址空间副本，COW）
  |
  | exec*(...)
  v
子进程变身为新程序（地址空间被新程序替换）

父进程
  |
  | wait/waitpid
  v
回收子进程退出状态（避免僵尸）
```

---

## 5.3.2 创建进程：fork / vfork / clone

### 1）fork：最常用
```c
#include <sys/types.h>
#include <unistd.h>

pid_t fork(void);
```
- 返回值：
  - 父进程：返回子进程 PID（>0）
  - 子进程：返回 0
  - 失败：返回 -1，`errno` 设置
- 核心机制：
  - **写时复制（Copy-On-Write, COW）**：初始不复制物理页，写入时才复制。
- fork 后父子差异：
  - PID 不同
  - 计时器、待处理信号等状态不同
  - 内存内容起点相同（COW 视角）
  - fd 表继承（共享同一打开文件表项）

> 重要注意（实务+考点）：fork 后子进程里如果不 exec，要非常小心缓冲区与锁（尤其多线程程序）。

### 2）vfork：更“激进”，谨慎使用
- 子进程与父进程暂时共享地址空间；父进程会被挂起，直到子进程 exec 或 _exit。
- 适合 “立刻 exec” 的极端性能场景；现代系统一般建议用 fork（COW 已很快）。

### 3）clone：Linux 底层原语（线程/容器相关）
- `pthread_create` 最终会用 `clone` 创建线程。
- `clone` 可以选择共享地址空间、fd、信号处理等（由 flags 控制）。

---

## 5.3.3 执行新程序：exec 家族

### 1）exec 的本质
- **不创建新进程**，而是把当前进程的用户态地址空间替换成新程序。
- 成功后：原代码不会再继续执行。

### 2）常见 exec 族函数（理解命名规则）
- `l`：参数以列表形式传入（list）
- `v`：参数以数组形式传入（vector）
- `p`：在 `PATH` 中搜索可执行文件
- `e`：显式提供环境变量 `envp`

最常用：
```c
#include <unistd.h>

int execl(const char *path, const char *arg0, ..., (char *)NULL);
int execv(const char *path, char *const argv[]);
int execle(const char *path, const char *arg0, ..., (char *)NULL, char *const envp[]);
int execve(const char *path, char *const argv[], char *const envp[]); // 系统调用语义核心

int execlp(const char *file, const char *arg0, ..., (char *)NULL);
int execvp(const char *file, char *const argv[]);
```
- 返回值：
  - 成功：**不返回**
  - 失败：返回 -1，`errno` 设置
[[linux系统编程exec族函数]]

---

## 5.3.4 等待子进程：wait / waitpid

### 1）为什么要 wait
- 子进程退出后，内核要保留一小段“退出信息”（exit status）供父进程读取。
- 父进程若不回收，这个记录会一直占用，形成**僵尸进程（Zombie）**。

### 2）函数原型
```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *wstatus);
pid_t waitpid(pid_t pid, int *wstatus, int options);
```
[[linux系统编程wait与waittpid]]
- `wait`：等待任意一个子进程。
- `waitpid`：可指定等待某个 PID，或使用特殊值：
  - `pid > 0`：等特定子进程
  - `pid == -1`：等任意子进程（类似 wait）
  - `pid == 0`：等与当前进程同进程组的子进程
- `options` 常见：
  - `WNOHANG`：非阻塞轮询
  - `WUNTRACED`：也返回停止（stop）状态
  - `WCONTINUED`：也返回继续（continue）状态

### 3）解析退出状态
```c
if (WIFEXITED(status)) {
    int code = WEXITSTATUS(status); // exit(code)
}
if (WIFSIGNALED(status)) {
    int sig = WTERMSIG(status);     // 被信号终止
}
```
`WTERMSIG(status)` 不是函数，而是一个宏。它用在你 `wait()` / `waitpid()` 得到的 `status`（也就是你说的 `wstatus`）上，**提取“子进程是被哪个信号终止的”那个信号编号**。

### 正确用法（必须先判断）

只有当子进程确实是“被信号杀死”时才有意义，所以要先判断：

```C
int status;
pid_t p = waitpid(child, &status, 0);

if (p > 0) {
    if (WIFSIGNALED(status)) {
        int sig = WTERMSIG(status);
        printf("child killed by signal %d\n", sig);
    }
}
```

- `WIFSIGNALED(status)`：判断是否因信号终止。
    
- `WTERMSIG(status)`：取出终止信号号（如 `SIGKILL=9`、`SIGSEGV=11` 等，具体值依平台）。

---

## 5.3.5 示例：最小“启动并等待”模板（强烈建议背下来）

```c
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        // 子进程：执行 /bin/ls -l
        char *argv[] = {"ls", "-l", NULL};
        execvp("ls", argv);

        // exec 失败才会走到这里
        perror("execvp");
        _exit(127); // 注意：子进程用 _exit，避免重复刷新父进程的 stdio 缓冲
    }

    // 父进程：等待子进程
    int status = 0;
    pid_t w = waitpid(pid, &status, 0);
    if (w < 0) {
        perror("waitpid");
        return 1;
    }

    if (WIFEXITED(status)) {
        printf("child exit code=%d\n", WEXITSTATUS(status));
    } else if (WIFSIGNALED(status)) {
        printf("child killed by signal=%d\n", WTERMSIG(status));
    }

    return 0;
}
```

**关键点总结（考试常考）**：
- `fork` 后用 `pid==0` 判断子进程。
- `exec*` 成功不返回，失败返回 -1。
- 子进程 exec 失败后：用 `_exit(127)`。
- 父进程必须 `wait/waitpid` 回收。

---

## 5.3.6 进阶：posix_spawn（更适合多线程程序）

在多线程程序里，`fork()` 之后只允许调用 async-signal-safe 函数，容易踩坑。此时可考虑：
```c
#include <spawn.h>
extern char **environ;

int posix_spawn(pid_t *pid, const char *path,
                const posix_spawn_file_actions_t *file_actions,
                const posix_spawnattr_t *attrp,
                char *const argv[], char *const envp[]);
```
- 常用于：稳定地“创建并执行”新进程，避免 fork+exec 在多线程场景的锁问题。

---

# 5.4 终止进程（退出、信号、回收）

本节目标：理解进程结束的路径（正常/异常/被杀），掌握 exit/_exit/kill/wait 的组合，以及僵尸/孤儿问题。

---

## 5.4.1 进程终止的三条主路径

### 1）正常终止（Normal termination）
- `main` 返回
- 调用 `exit(status)`

### 2）立即终止（Immediate termination）
- `_exit(status)` / `_Exit(status)`：直接进入内核退出，不执行用户态清理。

### 3）异常终止（Abnormal termination）
- `abort()`（通常触发 `SIGABRT`）
- 被信号终止（如 `SIGKILL`、`SIGSEGV`）

---

## 5.4.2 exit / _exit / _Exit 的差异（必考）

### 1）exit
```c
#include <stdlib.h>
void exit(int status);
```
- 会做用户态清理：
  - 调用 `atexit` 注册的函数
  - 刷新并关闭标准 I/O 缓冲（`stdout` 等）

### 2）_exit
```c
#include <unistd.h>
void _exit(int status);
```
- 不做上述用户态清理，直接退出。
- **fork 后的子进程**如果要退出，推荐 `_exit`，避免重复刷新父进程缓冲造成“双重输出”。

### 3）_Exit
```c
#include <stdlib.h>
void _Exit(int status);
```
- C 标准库提供的“立即退出”，语义类似 `_exit`。

---

## 5.4.3 退出码（Exit status）
- `exit(code)` 的 code 通常只有低 8 位可被父进程通过 `wait` 读取（0~255）。
- 常用约定：
  - `0`：成功
  - `127`：常用作“exec 失败/命令不存在”（很多 shell 采用）
[[linux系统退出码]]
---

## 5.4.4 信号终止：kill / raise
[[linux系统编程信号终止]]
### 1）向进程发送信号：kill
```c
#include <signal.h>

int kill(pid_t pid, int sig);
```
- `pid > 0`：发给指定 PID
- `pid == 0`：发给当前进程组
- `pid < 0`：发给某进程组（`-pgid`）或更大范围（需权限）

常用信号：
- `SIGTERM`：请求优雅退出（可被捕获/处理）
- `SIGKILL`：强制立即杀死（不可捕获、不可忽略）
- `SIGINT`：Ctrl+C

### 2）给自己发信号：raise
```c
#include <signal.h>
int raise(int sig);
```

---

## 5.4.5 僵尸进程与孤儿进程

### 1）僵尸（Zombie）
- 子进程已退出，但父进程没 wait。
- 表现：`ps` 里状态 `Z`。
- 解决：父进程 `wait/waitpid`；或处理 `SIGCHLD` 并循环回收。

### 2）孤儿（Orphan）
- 父进程先退出，子进程还在。
- 子进程会被 1 号进程（init/systemd）接管，最终会被回收。

---

## 5.4.6 示例：SIGCHLD 回收（避免产生僵尸）

> 适用于“父进程持续运行、会创建很多子进程”的场景（如服务器）。

```c
#define _GNU_SOURCE
#include <errno.h>
#include <signal.h>
#include <stdio.h>
#include <sys/wait.h>
#include <unistd.h>

static void on_sigchld(int sig) {
    (void)sig;
    int saved = errno;

    // 循环回收所有已退出的子进程（非阻塞）
    while (1) {
        int status;
        pid_t pid = waitpid(-1, &status, WNOHANG);
        if (pid > 0) {
            // 这里不要做复杂 I/O（信号处理函数限制较多）
            continue;
        }
        if (pid == 0) break;          // 没有更多已退出子进程
        if (pid < 0 && errno == ECHILD) break; // 没有子进程
        break;
    }

    errno = saved;
}

int main(void) {
    struct sigaction sa;
    sa.sa_handler = on_sigchld;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;
    sigaction(SIGCHLD, &sa, NULL);

    // 示例：创建子进程并立即退出
    pid_t pid = fork();
    if (pid == 0) {
        _exit(0);
    }

    // 父进程继续做事，不阻塞 wait
    sleep(1);
    puts("parent still running");
    return 0;
}
```

---

# 本章速记清单（建议考前背）

1. **程序 vs 进程 vs 线程**：程序静态；进程有独立地址空间与资源；线程共享进程资源、各有栈与寄存器。
2. **fork 返回值三态**：父>0、子=0、错=-1。
3. **exec 成功不返回**：失败才返回 -1。
4. **子进程退出优先 _exit**：避免 stdio 缓冲重复刷新。
5. **父进程必须 wait**：否则僵尸。
6. **kill(SIGTERM) 优雅；SIGKILL 强制**。
7. **getpid 同进程线程一致；gettid(系统调用) 线程各不相同**。

---

# 练习题（自测）

1. fork 后父子进程的 fd 指向是否相同？文件偏移如何变化？
2. exec 之后 PID 会变吗？为什么？
3. 为什么在子进程里 exec 失败后常用 `_exit(127)`？
4. 僵尸与孤儿的区别是什么？各自如何解决？
5. 多线程程序中 fork 有什么风险？有什么替代方案？（提示：posix_spawn）

