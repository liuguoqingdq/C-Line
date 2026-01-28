# Linux 信号：`kill()` 与 `raise()` 详细讲解

本节把你贴的两段内容讲透：
- **信号是什么**（它不是“杀进程专用”）
- `kill(pid, sig)` 的 **pid 取值语义、权限规则、返回值与 errno**
- `raise(sig)` 与 `kill()` 的关系（单线程/多线程差异）
- **生产中**怎么用 `SIGTERM` 做优雅退出、怎么避免僵尸、怎么写安全的信号处理函数

---

## 0. 信号到底是什么？

信号（signal）是 UNIX/Linux 的一种**异步通知机制**：内核或进程向目标进程（更准确是“进程/线程”）投递一个事件。

- “异步”的意思：信号可能在任何时刻到来，打断你正在执行的逻辑。
- 信号的结果：
  1) 采用默认行为（终止、停止、忽略、产生 core dump 等）
  2) 被用户自定义处理函数捕获（handler）
  3) 被阻塞（暂存为 pending，等解除阻塞后再递送）

> 注意：`kill` 这个名字历史上很误导，它更像是 **send_signal()**。

---

## 1. `kill()`：向进程/进程组发送信号

### 1.1 函数原型
```c
#include <signal.h>
int kill(pid_t pid, int sig);
```

### 1.2 `sig` 参数：你要发送什么信号
- `sig` 是一个信号编号，例如：`SIGTERM`、`SIGKILL`、`SIGINT`。
- **特殊值：`sig == 0`**
  - 不会真的发送信号
  - 只做“权限/存在性检查”（常用技巧：检测某 PID 是否存在、自己是否有权限向它发信号）

> 这也是为什么 `kill()` 经常用于“探测进程是否活着”，而不是真的杀它。

---

## 1.3 `pid` 参数：决定信号的投递范围（核心考点）

### A) `pid > 0`：发给指定 PID 的那个进程
- 典型用法：父进程管理子进程、监控程序终止 worker。

### B) `pid == 0`：发给**当前进程组**（current process group）
- “当前进程组”指调用者所属的 PGID。
- 典型用法：shell/作业控制（job control），一次性把信号发给同一 job 的所有进程（例如一条管道）。

### C) `pid < 0`：发给某个进程组（或更大范围）
- `pid == -pgid`：发给 **PGID 为 pgid 的那个进程组**。
- `pid == -1`：发给“你有权限发送的**所有进程**”（通常强权限才用；一般程序不要碰）。

> 实务建议：如果你的目标是进程组，优先用 `kill(-pgid, sig)` 或 `killpg(pgid, sig)`（后者是库函数封装）。

---

## 1.4 权限规则：为什么会 `EPERM`

并不是你想发就能发。
- 你需要对目标进程有权限：通常要求**同一用户**（同 UID），或具备特权（root / CAP_KILL）。
- 没权限会返回 `-1`，并设置 `errno=EPERM`。

> 这也解释了：你对系统上陌生进程 `kill(pid, SIGTERM)` 可能失败。

---

## 1.5 返回值与常见 errno（面试/排错必会）

- 成功：返回 0。
- 失败：返回 -1，`errno` 常见包括：
  - `ESRCH`：目标不存在（PID/PGID 没有对应进程）
  - `EPERM`：存在但无权限
  - `EINVAL`：`sig` 非法

配合 `sig==0` 可以区分“存在但没权限” vs “根本不存在”。

---

## 1.6 常用信号的语义（与生产实践强相关）

### `SIGTERM`：请求优雅退出（推荐）
- **可捕获**：你可以在 handler 里设置“退出标志”，让主循环收尾：
  - 停止接收新请求
  - flush 缓冲/落盘
  - 关闭 socket
  - 等待子线程/子进程结束

### `SIGKILL`：强制立即终止
- **不可捕获、不可忽略、不可阻塞**。
- 适用于：进程卡死/无法响应（例如死锁），不得不强杀。

### `SIGINT`：Ctrl+C
- 终端驱动发给前台进程组的典型信号。
- 交互式程序常用它作为“用户中断”。

---

## 2. `raise()`：给“自己”发信号

### 2.1 原型
```c
#include <signal.h>
int raise(int sig);
```

### 2.2 它等价于什么？
在**单线程/传统语境**下，可以近似理解为：
- `raise(sig)` ≈ `kill(getpid(), sig)`

但在**多线程（pthread）**环境中要更严谨：
- 信号是“进程级”的概念，但递送是“线程级”的：
  - **进程定向信号**（比如 `kill(pid, sig)`、`raise(sig)` 这类）会递送给进程中某个**未阻塞该信号**的线程（具体由实现选择）。
  - 如果你想指定某个线程接收，用 `pthread_kill(thread, sig)`。

> 实战要点：别假设 `raise()` 一定在“当前线程”执行 handler；你应当把 handler 设计成线程安全、可重入限制下的最小逻辑。

---

## 3. 用法对比：什么时候用 `kill`，什么时候用 `raise`

- 用 `kill`：
  - 管理子进程（父→子发 SIGTERM/SIGKILL）
  - job control（对 PGID 发信号）
  - 探测进程是否存在：`kill(pid, 0)`

- 用 `raise`：
  - 程序内部触发“自我中断/自我终止”流程（例如：检测到严重错误时触发 `SIGABRT`）
  - 单元测试里模拟某种信号路径

---

## 4. 信号处理函数怎么写才“生产可用”

信号处理函数（handler）限制非常多：
- 大多数库函数都不是 async-signal-safe
- 不能在 handler 里做复杂逻辑（malloc、printf、大量锁等都可能出事）

### 4.1 推荐模式：只设置一个原子标志
```c
#include <signal.h>
#include <stdatomic.h>

static volatile sig_atomic_t g_stop = 0;

static void on_term(int sig) {
    (void)sig;
    g_stop = 1;
}

int main() {
    struct sigaction sa = {0};
    sa.sa_handler = on_term;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART; // 可选：让被信号打断的某些系统调用自动重启
    sigaction(SIGTERM, &sa, NULL);

    while (!g_stop) {
        // 主循环做正常工作
    }

    // 收尾清理：关闭 fd、flush、join 线程...
    return 0;
}
```

> 这也是服务端程序优雅退出的标准姿势：SIGTERM 到来 → 置标志 → 主循环退出 → 收尾。

---

## 5. 典型示例：父进程优雅终止子进程

### 5.1 子进程：处理 SIGTERM，优雅退出
```c
static volatile sig_atomic_t stop = 0;
static void on_term(int sig) { (void)sig; stop = 1; }

int child_main() {
    struct sigaction sa = {0};
    sa.sa_handler = on_term;
    sigaction(SIGTERM, &sa, NULL);

    while (!stop) {
        // do work...
    }
    // cleanup...
    return 0;
}
```

### 5.2 父进程：先 SIGTERM，超时后 SIGKILL
思路：
1) `kill(child, SIGTERM)` 请求退出
2) `waitpid(child, ..., WNOHANG)` 轮询一段时间
3) 还不退出 → `kill(child, SIGKILL)` 强制终止

---

## 6. 常见坑总结（非常实用）

1. **`kill` 不是“杀死”，是“发信号”**：`SIGSTOP`、`SIGUSR1` 都可以发。
2. **`kill(pid, 0)`** 是探测利器：
   - 成功：存在且有权限
   - `EPERM`：存在但没权限
   - `ESRCH`：不存在
3. 对于一条管道/一个 job，应该用 **进程组**（`kill(-pgid, sig)`）
4. handler 里不要 printf/malloc/加锁做大活；尽量置标志。
5. 多线程程序里，`raise()` 触发的信号未必由“当前线程”处理；要么用 `pthread_kill` 指定线程，要么让 handler 逻辑保持最小。

---

## 7. 你可以继续问的两个“进阶点”（推荐）

1) `sigaction` 的 `SA_RESTART` 到底会重启哪些系统调用？哪些不会？
2) 进程组/会话/控制终端与 Ctrl+C / Ctrl+Z 的完整链路：shell 是如何用 `setpgid` + `tcsetpgrp` + `waitpid(WUNTRACED)` 管作业的？

