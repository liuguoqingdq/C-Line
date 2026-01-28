# 5.6 等待子进程终止（wait / waitpid / waitid）

本节目标：
1) 明确**为什么必须等待子进程**（避免僵尸）
2) 掌握 `wait()` / `waitpid()` 的 **pid 语义、options 语义、wstatus 解码**
3) 会写出生产可用的：
- 阻塞等待某个子进程
- 非阻塞批量回收（SIGCHLD handler / 主循环回收）
- shell/job-control 场景下等待“进程组”

---

## 5.6.1 为什么要 wait：僵尸进程（Zombie）

子进程退出后，内核不会立刻把它的所有痕迹都清掉，而是保留一个**最小的“退出记录”**（exit status、资源统计等），供父进程读取。
- 子进程此时进入 **Z（zombie）** 状态：
  - 不再运行
  - 但仍占用 PID 表项等内核资源

父进程如果长期不回收，会导致 zombie 堆积，严重时会耗尽 PID/表项，影响系统。

**回收方式**：父进程调用 `wait*()` 读取退出状态 → 内核释放退出记录。

---

## 5.6.2 `wait()`：等待任意一个子进程

```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *wstatus);
```

- 返回值：
  - 成功：返回“已退出子进程”的 PID
  - 失败：返回 -1，并设置 `errno`
- `wstatus`：输出参数，保存子进程状态编码；可为 `NULL` 表示不关心。

**等价关系**：
- `wait(&st)` 等价于 `waitpid(-1, &st, 0)`。

---

## 5.6.3 `waitpid()`：更灵活的等待

```c
pid_t waitpid(pid_t pid, int *wstatus, int options);
```

### (1) pid 参数：你要等谁（非常重要）
- `pid > 0`：等待 PID 等于 pid 的子进程
- `pid == -1`：等待任意子进程（类似 `wait`）
- `pid == 0`：等待**与调用者同一进程组**的任意子进程（job control 常用）
- `pid < -1`：等待**进程组 ID == |pid|** 的任意子进程（等待某个 job/管道）

### (2) options：你要等什么状态变化
- `0`：只等待“子进程终止”（退出或被信号杀死）
- `WNOHANG`：非阻塞：
  - 若没有子进程终止，立即返回 0（不是 -1）
- `WUNTRACED`：也报告“停止”（Ctrl+Z/`SIGSTOP` 等）
- `WCONTINUED`：也报告“继续”（被 `SIGCONT` 恢复）

> 服务器回收僵尸多用 `WNOHANG`；shell 作业控制常用 `WUNTRACED | WCONTINUED`。

### (3) 返回值：三态
- `>0`：某个子进程发生了你等待的状态变化，返回该子进程 PID
- `==0`：仅在 `WNOHANG` 且没有可回收子进程时出现
- `==-1`：出错（`errno` 解释原因）

常见 errno：
- `ECHILD`：没有子进程可等
- `EINTR`：等待被信号打断（常见，需要循环重试）

---

## 5.6.4 `wstatus` 到底是什么：如何解码

`wstatus` 不是“退出码”，而是一个**编码过的状态整数**。
用宏解码（不要自己位运算）：

- 正常退出：
  - `WIFEXITED(st)`：是否通过 `exit/_exit/return` 正常退出
  - `WEXITSTATUS(st)`：退出码（0~255），仅在 `WIFEXITED` 为真时使用
- 被信号杀死：
  - `WIFSIGNALED(st)`
  - `WTERMSIG(st)`：终止信号编号
  - `WCOREDUMP(st)`：是否产生 core（可选宏）
- 停止/继续：
  - `WIFSTOPPED(st)` / `WSTOPSIG(st)`
  - `WIFCONTINUED(st)`

---

## 5.6.5 生产级模板 1：等待指定子进程（处理 EINTR）

```c
#include <errno.h>
#include <sys/wait.h>

int wait_child(pid_t child) {
    int st;
    pid_t r;
    do {
        r = waitpid(child, &st, 0);
    } while (r == -1 && errno == EINTR);

    if (r == -1) return -1;

    if (WIFEXITED(st)) {
        return WEXITSTATUS(st);
    }
    if (WIFSIGNALED(st)) {
        // 常见做法：返回 128+signal 作为统一编码
        return 128 + WTERMSIG(st);
    }
    return -1;
}
```

要点：
- `EINTR` 是常态，别当异常。

---

## 5.6.6 生产级模板 2：非阻塞批量回收（主循环中扫僵尸）

```c
#include <errno.h>
#include <sys/wait.h>

void reap_children_nonblock(void) {
    while (1) {
        int st;
        pid_t r = waitpid(-1, &st, WNOHANG);
        if (r > 0) {
            // 回收了一个子进程，继续看看还有没有
            continue;
        }
        if (r == 0) {
            // 还有子进程，但当前没有退出的
            break;
        }
        // r < 0
        if (errno == ECHILD) break; // 没有子进程
        if (errno == EINTR) continue;
        break;
    }
}
```

典型使用位置：
- 事件循环每轮末尾
- 或在 `SIGCHLD` handler 里只设标志，主循环里调用该函数


-----

## 模板3

```C
// waitpid_blocking_demo.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <errno.h>

int main(void) {
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        // 子进程：模拟做点事
        printf("child: pid=%d, doing work...\n", (int)getpid());
        sleep(2);
        printf("child: done, exit(42)\n");
        _exit(42);  // 子进程退出（推荐用 _exit）
    }

    // 父进程：阻塞等待这个子进程结束
    printf("parent: pid=%d, waiting child=%d ...\n", (int)getpid(), (int)pid);

    int status = 0;
    pid_t r;
    do {
        r = waitpid(pid, &status, 0);   // options=0 => 阻塞等待
    } while (r == -1 && errno == EINTR); // 如果被信号打断，就重试

    if (r == -1) {
        perror("waitpid");
        return 1;
    }

    // 解析子进程状态
    if (WIFEXITED(status)) {
        printf("parent: child exited normally, code=%d\n", WEXITSTATUS(status));
    } else if (WIFSIGNALED(status)) {
        printf("parent: child killed by signal=%d\n", WTERMSIG(status));
    } else {
        printf("parent: child ended with other status\n");
    }

    return 0;
}

```



---

## 5.6.7 `SIGCHLD` 与“避免僵尸”

- 子进程状态变化会触发 `SIGCHLD`。
- 推荐做法：
  - handler 里**不要做复杂事**，只置一个 `sig_atomic_t` 标志
  - 主循环里调用 `reap_children_nonblock()`

> 也可在 handler 里直接 `waitpid(-1, ..., WNOHANG)` 循环回收，但需注意信号处理函数的限制。

---

## 5.6.8 扩展：`waitid()`（更精细的接口）

`waitid()` 可返回更结构化的信息（`siginfo_t`），适合做更精细的状态处理：
```c
#include <sys/wait.h>
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
```

一般课程/面试以 `waitpid` 为主。

---

# 5.7 会话（session）和进程组（process group）

本节目标：
1) 搞清层级：**会话 SID → 进程组 PGID → 进程 PID**
2) 解释它们为什么存在：**作业控制（job control）与控制终端（controlling terminal）**
3) 学会 `setpgid/setsid/tcsetpgrp` 的典型组合

---

## 5.7.1 基本定义

### 进程组（Process Group）
- 一组相关进程的集合，用 **PGID** 标识。
- 目的：
  - 对“整组”发信号（Ctrl+C、kill -pgid）
  - 作为 job-control 的单位（前台/后台 job）

### 会话（Session）
- 会话是一组进程组的集合，用 **SID** 标识。
- 通常对应：一次登录的 shell 环境。
- 会话里有：
  - 一个或多个进程组
  - 在任意时刻最多一个“前台进程组”（foreground pgrp）

---

## 5.7.2 控制终端（Controlling Terminal）与前台/后台

当你在终端里运行程序：
- 终端属于某个会话的控制终端
- 终端只允许“前台进程组”读输入
- Ctrl+C/ Ctrl+Z 等由终端驱动产生信号，发往前台进程组

典型现象：
- 后台程序如果试图读终端，会收到 `SIGTTIN` 被暂停

---

## 5.7.3 相关系统调用/函数

### (1) 获取 PGID / SID
```c
pid_t getpgrp(void);        // 当前进程组
pid_t getpgid(pid_t pid);   // pid==0 表示当前
pid_t getsid(pid_t pid);    // pid==0 表示当前
```

### (2) 设置进程组：`setpgid`
```c
int setpgid(pid_t pid, pid_t pgid);
```
- `pid==0`：当前进程
- `pgid==0`：让 PGID=pid（创建新进程组的常见写法）

用途：shell 创建 job/管道时，把多个子进程放进同一 PGID。

### (3) 创建新会话：`setsid`
```c
pid_t setsid(void);
```
成功后：
- 当前进程成为会话首领（SID=自己的 PID）
- 同时成为新进程组组长（PGID=自己的 PID）
- 脱离原控制终端（没有 controlling terminal）

失败条件（重要）：
- 调用者如果已经是“进程组组长”，`setsid()` 失败（EPERM）。

---

## 5.7.4 “组长”到底有什么用

进程组组长：满足 `PID == PGID` 的那个进程。
- 不是管理者，没有特权。
- 作用：作为 PGID 的锚点；同时影响 `setsid()` 的可调用性（组长不能直接 setsid）。

---

## 5.7.5 典型场景：shell 如何管理一条管道

命令：`cat file | grep foo | wc -l`

shell 的常见做法：
1) fork 出多个子进程
2) 把它们 `setpgid(child, pgid)` 放进同一个进程组
3) 将该 pgid 设为终端前台进程组（`tcsetpgrp`，在 `<termios.h>`/`unistd.h`）
4) Ctrl+C 触发 SIGINT → 发给前台进程组（整条管道一起中断）
5) shell 用 `waitpid(-pgid, ...)` 等待整组结束/停止

> 这就是会话+进程组存在的核心价值：把“job”当成一个单位。

---

# 5.8 守护进程（daemon）

本节目标：
1) 守护进程的特征：后台运行、脱离终端、长期服务
2) 经典 daemonize 步骤背后的原因（为什么要 double-fork）
3) 生产实践：日志、工作目录、umask、文件描述符、信号与退出

---

## 5.8.1 守护进程是什么

守护进程通常具备：
- 不依赖控制终端（终端关闭不影响）
- 后台长期运行，提供服务
- 自己管理日志/配置/生命周期

例如：sshd、cron、systemd 管理的服务等。
[[linux系统编程守护进程]]

---

## 5.8.2 经典 daemonize 过程（逐步解释“为什么”）

### 步骤 1：fork，父进程退出
目的：
- 让 shell 认为命令已结束
- 子进程不再是进程组组长的概率更高（为 setsid 做铺垫）

### 步骤 2：子进程调用 `setsid()`
目的：
- 创建新会话，脱离原会话
- 断开控制终端（没有 controlling terminal）

### 步骤 3：再次 fork（double-fork），让第一子进程退出
目的：
- 防止守护进程重新获得控制终端
- 因为“会话首领”如果打开终端设备可能会重新获得 controlling terminal
- 第二子进程不再是会话首领，更安全

### 步骤 4：设置工作目录、umask
- `chdir("/")` 或项目目录：避免占用挂载点导致无法卸载
- `umask(0)` 或合适权限掩码：让创建文件权限可控

### 步骤 5：关闭/重定向标准 fd
- 关闭 stdin/stdout/stderr 或重定向到 `/dev/null` 或日志文件

### 步骤 6：处理信号与 PID 文件（可选）
- `SIGTERM`：优雅退出
- 记录 pidfile 便于管理（现代 systemd 不一定需要）

---

## 5.8.3 生产级 daemonize 示例（最常见模板）

```c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

static void daemonize(void) {
    pid_t pid;

    // 1) fork
    pid = fork();
    if (pid < 0) exit(1);
    if (pid > 0) exit(0); // 父进程退出

    // 2) setsid
    if (setsid() < 0) exit(1);

    // 3) second fork
    pid = fork();
    if (pid < 0) exit(1);
    if (pid > 0) exit(0);

    // 4) umask & chdir
    umask(0);
    if (chdir("/") < 0) exit(1);

    // 5) 关闭并重定向标准 fd
    int fd = open("/dev/null", O_RDWR);
    if (fd >= 0) {
        dup2(fd, STDIN_FILENO);
        dup2(fd, STDOUT_FILENO);
        dup2(fd, STDERR_FILENO);
        if (fd > 2) close(fd);
    }
}

int main(void) {
    daemonize();

    // 你的服务主循环
    while (1) {
        sleep(10);
    }

    return 0;
}
```

关键点解释：
- 两次 `fork`：防止重新获得控制终端
- `setsid`：创建新会话、脱离终端
- 重定向 0/1/2：避免写到已关闭的终端导致错误

---

## 5.8.4 守护进程的优雅退出（SIGTERM）

生产守护进程必须支持：
- `SIGTERM` 到来后停止接收新请求
- flush 缓冲/关闭 fd
- 等待子线程退出

推荐模式：handler 只置标志，主循环收尾。

---

## 5.8.5 systemd 时代的补充

现代 Linux 中大多数服务由 systemd 管理：
- 不一定需要自己 double-fork（Type=simple/forking 的差异）
- 日志可以交给 journald

但掌握传统 daemonize 仍然有价值：理解会话/终端/信号的底层逻辑。

---

# 本节速记

- `wait/waitpid`：父进程必须回收子进程，避免僵尸
- `waitpid` 的 pid 语义：
  - `-1` 任意子进程
  - `0` 同进程组
  - `-pgid` 指定进程组
- 会话（SID）是进程组集合；前台进程组决定 Ctrl+C 等信号的接收者
- 守护进程核心：`fork → setsid → fork → chdir/umask → 重定向 fd → 信号收尾`

