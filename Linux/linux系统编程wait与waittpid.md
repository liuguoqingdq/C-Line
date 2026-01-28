## 1）它们到底用来干什么

父进程 `fork()` 出子进程后，子进程退出时内核会留下“退出信息”（退出码/被哪个信号杀死等）。父进程必须调用 `wait/waitpid` **取走这份信息**，这样内核才能**释放子进程相关资源**；否则子进程会一直处于 **zombie（僵尸）** 状态。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)

并且：如果子进程已经“状态变化”（如已退出），`wait/waitpid` 会立刻返回；否则会阻塞等待，或被信号打断（是否自动重启与 `sigaction` 的 `SA_RESTART` 有关）。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)

---

## 2）`wait()`：等“任意一个子进程退出”

`pid_t wait(int *wstatus);`

### 行为

- 阻塞直到**任意一个子进程终止**，然后返回该子进程的 PID。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
    
- `wait(&wstatus)` 等价于：`waitpid(-1, &wstatus, 0)`。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
    
- `wstatus` 可以传 `NULL`（你不关心退出原因/退出码时）。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
    
## `wstatus` 里装的是什么信息？

至少会包含这些之一（取决于你怎么等、子进程发生了什么）：

- **正常退出**：子进程调用 `exit(code)` / `_exit(code)` / `main` return  
    → `wstatus` 里会编码“正常退出”以及 **退出码 code（通常低 8 位）**
    
- **被信号杀死**：比如 `SIGKILL/SIGSEGV`  
    → `wstatus` 里会编码“被信号终止”以及 **信号编号**
    
- （可选）**被暂停 stop**：如 Ctrl+Z（`SIGTSTP`），需要 `WUNTRACED` 才会返回这种状态
    
- （可选）**继续运行 continued**：从 stop 被 `SIGCONT` 恢复，需要 `WCONTINUED`
### 返回值

- 成功：返回“发生状态变化的子进程 PID”。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
    
- 失败：返回 `-1`，常见 `errno`：
    
    - `ECHILD`：没有可等待的子进程/都被回收完了 [man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
        
    - `EINTR`：阻塞等待时被信号打断 [man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
        

---

## 3）`waitpid()`：更精确地等“哪个子进程 / 哪类状态变化”

```C
pid_t waitpid(pid_t pid, int *wstatus, int options);
```

### 3.1 `pid` 参数：你要等“谁”

`pid` 的取值语义很关键：[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)

- `pid > 0`：等待 **PID==pid** 的那个子进程
    
- `pid == -1`：等待 **任意子进程**（等价于 `wait()`）
    
- `pid == 0`：等待 **与调用者同一进程组** 的任意子进程
    
- `pid < -1`：等待 **进程组 ID == |pid|** 的任意子进程（常用于 job control：等待某个作业/管道组）
    

### 3.2 `options` 参数：你要等“什么状态变化”

默认只等“子进程终止”；`options` 可以扩展：[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)

- `WNOHANG`：**非阻塞**。如果指定范围内“有子进程但都还没变成可回收状态”，立刻返回 `0`（不是 -1）。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
    
- `WUNTRACED`：子进程被信号**暂停（stop）**也返回（例如 Ctrl+Z 导致停止）。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
    
- `WCONTINUED`：子进程从停止状态被 `SIGCONT` **恢复运行**也返回。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
    

### 返回值（比 wait 多一种情况）

- 成功：返回“状态变化的子进程 PID”。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
    
- `WNOHANG` 且没有符合条件的状态变化：返回 `0`。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
    
- 失败：返回 `-1`（如 `ECHILD`、`EINTR` 等）。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
    

---

## 4）`wstatus` 怎么读：一套宏（必背）

当 `wstatus != NULL` 时，`wait/waitpid` 会把状态写进 `*wstatus`，用这些宏解析（注意宏参数是 **wstatus 的值**，不是指针）：[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)

- 正常退出？
    
    - `WIFEXITED(wstatus)`：是否“正常退出”（`exit/_exit/return main`）[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
        
    - `WEXITSTATUS(wstatus)`：退出码低 8 位（只有在 `WIFEXITED` 为真时用）[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
        
- 被信号杀死？
    
    - `WIFSIGNALED(wstatus)`：是否被信号终止 [man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
        
    - `WTERMSIG(wstatus)`：终止它的信号号（只有在 `WIFSIGNALED` 为真时用）[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
        
    - `WCOREDUMP(wstatus)`：是否产生 core dump（非 POSIX，移植时要 `#ifdef`）[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
        
- 停止/继续（需要配合 `WUNTRACED/WCONTINUED`）
    
    - `WIFSTOPPED(wstatus)` / `WSTOPSIG(wstatus)` [man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
        
    - `WIFCONTINUED(wstatus)` [man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)
        

---

## 5）两段最常用的“标准模板”

### 模板 A：父进程阻塞等指定子进程结束

```C
int status;
pid_t r;
do {
    r = waitpid(child_pid, &status, 0);
} while (r == -1 && errno == EINTR);

if (r == -1) { perror("waitpid"); }
else if (WIFEXITED(status)) { printf("exit=%d\n", WEXITSTATUS(status)); }
else if (WIFSIGNALED(status)) { printf("killed by sig=%d\n", WTERMSIG(status)); }
```

`EINTR` 的处理很常见：阻塞等待可能被信号中断。[man7.org](https://man7.org/linux/man-pages/man2/wait.2.html)

### 模板 B：服务器/父进程不想阻塞，用 `WNOHANG` 扫僵尸

```C
while (1) {
    int status;
    pid_t r = waitpid(-1, &status, WNOHANG);
    if (r > 0) continue;   // 回收了一个，可能还有，继续扫
    if (r == 0) break;     // 还有子进程，但当前没人退出
    if (errno == ECHILD) break; // 没有子进程了
    if (errno == EINTR) continue;
    perror("waitpid");
    break;
}
```

`WNOHANG` 的“返回 0”语义是很多人第一次会踩的点。