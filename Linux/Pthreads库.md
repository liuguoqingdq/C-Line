# pthread 线程 API（超详细版）：create / self / equal / exit / cancel / join / detach

> 适用场景：Linux 系统编程（C / pthread）。内容以“你写代码时真正会踩的坑”为主：**参数怎么填、返回值怎么判错、语义边界在哪里、怎样写才安全**。
>
> 你提到的 **start_thread**：在 pthread 对外 API 里没有 `start_thread()` 这个函数，它通常是教材/笔记对 **线程入口函数指针** 的叫法，也就是 `pthread_create()` 的 `start_routine`。
>
> 你写的 `pthread_detch`：应为 **`pthread_detach`**。

---

## 0. pthread 一般判错规则（必须先讲清）

### 0.1 返回值约定
- **pthread 大多数函数：成功返回 0；失败返回“错误码本身”**（例如 `EINVAL` / `ESRCH` / `EDEADLK` 等）。
- **通常不会像系统调用那样返回 -1 并设置 errno**（有少数例外/扩展，但学习阶段按这个规律就不会错）。

典型写法：
```c
int rc = pthread_mutex_lock(&m);
if (rc != 0) {
    fprintf(stderr, "pthread_mutex_lock: %s\n", strerror(rc));
}
```

### 0.2 线程生命周期的两个关键状态：joinable vs detached
- **joinable（可 join）**：线程退出后资源不会自动回收，必须 `pthread_join()` 才释放。
- **detached（分离态）**：线程退出后资源自动回收，**不能 join**。

> 你后面学 `join` / `detach` 的所有坑，几乎都来自对这点不牢。

---

## 1) pthread_create —— 创建线程

### 1.1 原型
```c
#include <pthread.h>

int pthread_create(pthread_t *thread,
                   const pthread_attr_t *attr,
                   void *(*start_routine)(void *),
                   void *arg);
```

### 1.2 作用（功能）
创建一个新线程，新线程从 `start_routine(arg)` 开始执行。

### 1.3 参数逐个解释

#### (1) `pthread_t *thread`
- **输出参数**。
- 成功时被写入一个“线程句柄”（`pthread_t`）。
- 注意：`pthread_t` 是抽象类型（可能是整数、指针、结构体），不要假设能直接 `%ld` 打印。

#### (2) `const pthread_attr_t *attr`
- 线程属性；为 `NULL` 表示默认属性。
- 常见可配置项：
  - 分离态（joinable / detached）
  - 栈大小（stack size）
  - 调度策略/优先级（很多情况需要权限，且不一定生效）

#### (3) `void *(*start_routine)(void *)`
- 线程入口函数指针（你说的 start_thread）。
- 线程创建成功后，这个函数在新线程上下文里被调用。

#### (4) `void *arg`
- 传给入口函数的参数。
- 常见用法：传结构体指针，把多个参数打包。

### 1.4 返回值
- 成功：0
- 失败：错误码（典型）
  - `EAGAIN`：资源不足（线程数/内存/系统限制）
  - `EINVAL`：属性不合法（比如栈大小不合规）
  - `EPERM`：属性要求的权限不够（例如设置实时调度）

### 1.5 最常见坑（非常重要）

#### 坑 A：把“栈上局部变量地址”传给线程
```c
int x = 123;
pthread_create(&t, NULL, worker, &x); // x 可能很快出作用域
```
如果主线程很快返回/覆盖栈，子线程读到的地址就悬空。

**安全写法**：
- 用堆内存（malloc）
- 或保证主线程生命周期覆盖子线程使用时间

#### 坑 B：忘记 join/detach → 资源泄漏
- joinable 线程退出后，如果你不 join，它的线程控制块等资源会一直留着（类似“线程版僵尸”）。

---

## 2) start_routine（线程入口函数）—— 线程从哪里开始执行

### 2.1 典型签名
```c
void *worker(void *arg);
```

### 2.2 参数与返回
- 参数 `arg`：来自 `pthread_create(..., arg)`
- 返回值：线程结束时的“返回指针”，供 `pthread_join()` 拿到

### 2.3 线程结束的三种常见方式
1) 入口函数 `return ptr;`
2) 显式调用 `pthread_exit(ptr);`
3) 被取消（`pthread_cancel`），且在允许取消 + 取消点处生效

> 等价理解：`return X;` ≈ `pthread_exit(X);`

## 1) 那我想返回 int / struct / double 怎么办？

核心思路就两种：

### 方案 A：把结果放到“输出参数”里（最常用、最安全）

主线程准备一块内存（或结构体），把指针传给线程，线程写进去。
```C
typedef struct { int code; double x; } Result;

void *worker(void *arg) {
    Result *out = arg;
    out->code = 123;
    out->x = 3.14;
    return NULL; // 返回值不用也行
}

int main() {
    pthread_t t;
    Result r;
    pthread_create(&t, NULL, worker, &r);
    pthread_join(t, NULL);
    // r 里就是结果
}
```

## 2) 我能不能直接把整数“塞进 void* 返回”？

在 64 位系统上很多人会写这种技巧：

```C
return (void*)(intptr_t)some_int;
```
`intptr_t` 是一个**整数类型**，它的设计目的只有一个：

> **能“装得下一个指针”的整数类型**  
> 也就是说，你可以把 `void*` 指针值安全地转换成 `intptr_t` 再转换回来（不丢位）。

主线程：

```C
int x = (int)(intptr_t)retval;
```

**能用，但不推荐新手在作业/生产里用**，原因：

- 依赖 `intptr_t` 的转换语义
    
- 只适合“小整数”，不适合复杂返回值
    
- 可读性差
    

如果你一定要用，正确姿势是包含头文件：

`#include <stdint.h>`

---

## 3) pthread_self —— 获取当前线程句柄

### 3.1 原型
```c
pthread_t pthread_self(void);
```

### 3.2 作用
返回当前调用线程的 `pthread_t`。

### 3.3 返回值
- 返回 `pthread_t`（不会失败）。

### 3.4 易混点：pthread_t vs Linux TID
- `pthread_t` 是 **pthread 层**的“句柄”。
- Linux 内核的线程 ID（TID）常用 `gettid()`（glibc 上是 `syscall(SYS_gettid)`）。
- 很多日志/调试工具显示的是 TID，而不是 `pthread_t`。

---

## 4) pthread_equal —— 比较两个线程句柄

### 4.1 原型
```c
int pthread_equal(pthread_t t1, pthread_t t2);
```

### 4.2 作用
判断 `t1` 和 `t2` 是否代表同一个线程。

### 4.3 参数
- `t1`, `t2`：两个 `pthread_t`。

### 4.4 返回值
- 非 0：相等
- 0：不相等

### 4.5 为什么不能直接 `t1 == t2`？
因为 `pthread_t` 是抽象类型，规范只保证 `pthread_equal` 正确。

---

## 5) pthread_exit —— 结束当前线程

### 5.1 原型
```c
void pthread_exit(void *retval);
```

### 5.2 作用
终止**当前线程**，并设置一个返回值给 `pthread_join`。

### 5.3 参数
- `retval`：线程的“退出值”。这个 `retval` 的意义只有一个：**把“线程结束时要交给别人（通常是 join 者）的结果/状态”带出去**。

因为线程结束后，它的栈会消失、寄存器上下文也没了，所以 pthread 需要一个**统一的、可被回收方拿到的“退出值通道”**。这就是 `retval`。

## 1) `retval` 会被谁拿到？怎么拿？

只要线程是 **joinable** 的，另一个线程调用：

`void *ret; pthread_join(tid, &ret);`

这里 `ret` 就会得到线程的 `retval`（或者入口函数 `return` 的值）。

> 等价关系：  
> 在线程入口函数里 `return X;` ≈ `pthread_exit(X);`

---

## 2) 这个返回值具体能用来干嘛？

常见用法有 4 类：

### A. 返回“成功/失败状态码”

比如用 `malloc` 分配一个 int 返回（主线程负责 free）：

```C
void *worker(void *arg) {
    int *rc = malloc(sizeof(int));
    *rc = 0; // 成功
    pthread_exit(rc);
}
```
主线程：

```C
void *p;
pthread_join(t, &p);
int code = p ? *(int*)p : -1;
free(p);

```

### B. 返回“计算结果指针”

线程计算出一个数组/结构体，放堆上，返回指针：

`Result *r = malloc(sizeof(Result)); r->sum = ... return r;`

### C. 返回“被取消”的状态（特殊值）

如果线程是被 `pthread_cancel` 取消并且生效，`pthread_join` 拿到的返回值通常是：

`PTHREAD_CANCELED`

所以你可以用它判断线程是不是被 cancel 结束的。

### D. 返回“共享对象的指针”（但要保证生命周期）

你也可以返回指向全局对象/主线程提供的内存的指针，但必须保证：

- 被指向的对象在线程结束后仍然有效
    
- 并且并发访问已正确同步
    

---

## 3) 重要限制：`retval` **不能**指向线程栈上的局部变量

这是最常见坑：

```C
void *worker(void *arg) {
    int x = 123;
    return &x;  // ❌ 错：线程结束后栈销毁，指针悬空
}
```
正确做法：要么用堆 `malloc`，要么由调用者提供输出缓冲区。

---

## 4) 如果我不需要返回值怎么办？

- 可以 `pthread_exit(NULL);`
    
- 或者入口函数 `return NULL;`
    
- join 时也可以传 `NULL`，表示不关心返回值：
    

`pthread_join(t, NULL);`

---

## 5) `retval` 本质上是什么？

可以把它理解为“**线程的退出码/返回值**”，但因为 pthread 统一用 `void*`，所以它更像“一个指针通道”：

- 想返回复杂结果 → 返回指针（堆分配/共享对象）
    
- 想返回整数 → 用输出参数更好；或用 `intptr_t` 做小整数编码（不推荐当主方案）
### 5.4 返回值
无（不会返回）。

### 5.5 与进程退出的区别（考试常问）
- **线程入口函数 return / pthread_exit**：只结束当前线程。
- **调用 `exit()` 或 main 返回**：结束整个进程，所有线程都终止。

因此在主线程里：
- `return 0;` → 整个进程结束
- `pthread_exit(NULL);` → 主线程结束，但进程只要还有其他线程，就不会退出

---

## 6) pthread_cancel —— 请求取消一个线程

### 6.1 原型
```c
int pthread_cancel(pthread_t thread);
```

### 6.2 作用（核心语义）
向目标线程发送“取消请求”。

> **注意：cancel 不是 kill。它只是发请求。目标线程是否退出、何时退出，取决于它的取消状态和取消类型。**

### 6.3 参数
- `thread`：目标线程句柄。

### 6.4 返回值
- 成功：0（表示“请求已成功发出”）
- 失败：错误码
  - `ESRCH`：线程不存在

### 6.5 取消生效机制：state + type + cancellation point
取消真正生效必须满足：
1) 目标线程 `cancel state = ENABLE`
2) 取消类型：
   - `DEFERRED`：到达取消点才退出（默认）
   - `ASYNCHRONOUS`：几乎随时可被取消（非常危险）
3) 在 `DEFERRED` 下，线程必须执行到一个取消点（或主动 `pthread_testcancel()`）

### 6.6 取消点（cancellation point）到底是什么？
在 DEFERRED 模式下，线程只有在执行到某些“可能阻塞/库函数指定的点”时才检查是否有 pending cancel。
典型取消点（常见）：
- `pthread_cond_wait / pthread_cond_timedwait`
- 很多阻塞 I/O：`read`/`write`/`accept`/`recv`/`send`（依实现与调用模式）
- `sleep`/`nanosleep`

> 这也是为什么你 cancel 一个忙循环线程，它可能“永远不退出”：因为它没到取消点。

---

## 7) pthread_setcancelstate —— 开关：允许/禁止响应取消

### 7.1 原型
```c
int pthread_setcancelstate(int state, int *oldstate);
```

### 7.2 作用
设置当前线程的取消“开关”。

### 7.3 参数
- `state`：
  - `PTHREAD_CANCEL_ENABLE`：允许取消
  - `PTHREAD_CANCEL_DISABLE`：禁止取消（cancel 请求会挂起 pending）
- `oldstate`：输出旧状态，可为 NULL

### 7.4 返回值
- 成功：0
- 失败：`EINVAL`（参数非法）

### 7.5 最典型用法：保护临界区，防止“被取消时锁没释放”
```c
int old;
pthread_setcancelstate(PTHREAD_CANCEL_DISABLE, &old);

pthread_mutex_lock(&m);
// 修改共享数据
pthread_mutex_unlock(&m);

pthread_setcancelstate(old, NULL);
```

解释：如果线程在持锁期间被取消而直接退出，锁永远不释放，其他线程可能永久阻塞。
所以要么临界区 disable cancel，要么使用 cleanup handler。

---

## 8) pthread_setcanceltype —— 响应时机：延迟/异步

### 8.1 原型
```c
int pthread_setcanceltype(int type, int *oldtype);
```

### 8.2 作用
设置当前线程的取消响应方式。

### 8.3 参数
- `type`：
  - `PTHREAD_CANCEL_DEFERRED`：延迟取消（默认）——到取消点才退出
  - `PTHREAD_CANCEL_ASYNCHRONOUS`：异步取消——可能在任何指令边界附近退出
- `oldtype`：输出旧 type，可为 NULL

### 8.4 返回值
- 成功：0
- 失败：`EINVAL`

### 8.5 为什么 ASYNCHRONOUS 非常危险？
因为线程可能在以下时刻被“掐死”：
- 持有 mutex
- 正在修改共享结构体（写了一半）
- 正在 malloc/free 内部
- 正在写文件/写 socket 的中途逻辑

这会造成：死锁、内存破坏、数据结构不一致、资源泄漏。

工程建议：**默认坚持 DEFERRED**，必要时在循环里主动插入 `pthread_testcancel()`。

---

## 9) pthread_join —— 等待线程结束并回收资源

### 9.1 原型
```c
int pthread_join(pthread_t thread, void **retval);
```

### 9.2 作用
阻塞等待目标线程退出，并回收其资源（仅对 joinable 线程）。

### 9.3 参数
- `thread`：目标线程
- `retval`：输出线程退出值（可 NULL）
  - 若线程被取消：`*retval` 通常等于 `PTHREAD_CANCELED`

### 9.4 返回值
- 成功：0
- 失败：错误码（常见）
  - `ESRCH`：线程不存在
  - `EINVAL`：线程不可 join（已 detach 或不是 joinable）
  - `EDEADLK`：会导致死锁
    - 例如：线程 join 自己
    - 或者 A join B，同时 B join A

### 9.5 join 的关键规则
- 一个线程只能 join 一次。
- join 会阻塞调用者，因此常见模式：
  - 主线程 join worker
  - 或者专门一个“回收线程” join 一批 joinable 线程

---

## 10) pthread_detach —— 分离线程（自动回收）

### 10.1 原型
```c
int pthread_detach(pthread_t thread);
```

### 10.2 作用
把线程设置为 detached：线程退出时系统自动回收其资源。

### 10.3 参数
- `thread`：目标线程

### 10.4 返回值
- 成功：0
- 失败：错误码（常见 `ESRCH`、`EINVAL`）

### 10.5 detach 的关键语义
- **detach 后不能再 join**（否则 `EINVAL`）。
- detach 适合“fire-and-forget”：你不需要线程返回值，也不需要等待它结束。

### 10.6 detach 的两种设置方式
- 创建时就 detached：通过 `pthread_attr_setdetachstate`（最干净）
- 创建后 detach：`pthread_detach(t)`（需要确保线程句柄有效且只 detach 一次）

---

## 11) 取消（cancel）写对的两种标准套路（强烈建议掌握）

### 11.1 套路 A：关键临界区禁用取消
适用于：临界区很短，锁简单。

步骤：
1) `pthread_setcancelstate(DISABLE, &old)`
2) 加锁 → 修改共享数据 → 解锁
3) `pthread_setcancelstate(old, NULL)`

优点：简单
缺点：如果临界区里做了可能长时间阻塞的操作，会让取消变得不可控

### 11.2 套路 B：cleanup handler（更通用）
思想：无论线程如何退出（正常 return / pthread_exit / cancel），都保证释放资源。

常见用法：
- 在持锁后立刻 push 一个 cleanup 来 unlock
- 线程在取消点退出时，会自动执行 cleanup

> 这部分如果你需要，我可以给你完整可编译 demo（带 cond_wait 作为取消点）。

---

## 12) 把这些 API 串起来的一套“最小正确模型”（你写项目就按这个走）

1) 主线程 `pthread_create` worker（默认 joinable）
2) worker 在循环里工作：
   - 如果可能阻塞，用 cond_wait / epoll_wait 等（会形成取消点）
   - 或者显式 `pthread_testcancel()` 让取消可控生效
3) 主线程需要结束 worker：
   - `pthread_cancel(worker)`（发请求）
   - `pthread_join(worker, &ret)`（等待回收）
4) 如果是 fire-and-forget：创建后 `pthread_detach` 或创建时直接 detached

---

## 13) 你背诵/写题可直接套用的“回答模板”

- `pthread_create`：创建线程（线程句柄输出、属性、入口函数、参数）；返回 0/错误码
- `start_routine`：线程入口（void* 参数、void* 返回）；return ≈ pthread_exit
- `pthread_self`：取当前线程句柄
- `pthread_equal`：比较两个 pthread_t
- `pthread_exit`：结束当前线程并传出返回值
- `pthread_cancel`：发取消请求；是否退出取决于 cancel state/type + 取消点
- `pthread_setcancelstate`：允许/禁止取消（保护临界区）
- `pthread_setcanceltype`：延迟/异步（异步危险，默认延迟）
- `pthread_join`：等待线程退出并回收资源，获得返回值（被 cancel 时为 PTHREAD_CANCELED）
- `pthread_detach`：设置分离态，退出自动回收，不能 join

-----
## 1) 互斥锁 `pthread_mutex_t`（最核心）

### 1.1 解决什么问题

当多个线程会同时读写同一份共享数据（链表、队列、计数器、全局结构体）时，需要保证**同一时刻只有一个线程进入临界区**，避免竞态。

### 1.2 基本 API（你必须会）

```C
#include <pthread.h> 
int pthread_mutex_init(pthread_mutex_t *m, const pthread_mutexattr_t *attr); 
int pthread_mutex_destroy(pthread_mutex_t *m);  
int pthread_mutex_lock(pthread_mutex_t *m);      // 阻塞直到拿到锁 
int pthread_mutex_trylock(pthread_mutex_t *m);   // 不阻塞，拿不到通常返回 EBUSY 
int pthread_mutex_unlock(pthread_mutex_t *m);
```

**返回值规则**：成功返回 0，失败返回错误码本身（不是 -1）。

### 1.3 最关键语义（很多人没讲清的点）

- **所有权（ownership）**：mutex 一般要求“谁 lock 谁 unlock”。别的线程 unlock 是未定义/错误（具体看实现和 mutex 类型）。
    
- **互斥 + 内存可见性**：`lock`/`unlock` 不只是“排队”，它还建立同步关系：
    
    - 在 A 线程 `unlock` 前对共享数据的写入，B 线程在 `lock` 成功后能看到（这就是你写多线程时“改了数据别人看得见”的基础）。
        
- **临界区要短**：锁内不要做 I/O、sleep、长计算，否则会把并发彻底“串行化”。
    

### 1.4 初始化两种方式

**静态初始化**（最省事）：

```C
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
```

**动态初始化**（需要属性时用）：

```C
pthread_mutex_t m; 
pthread_mutex_init(&m, NULL);
```

---

## 2) mutex 属性（pthread_mutexattr）——考试/工程常考

### 2.1 锁类型：NORMAL / ERRORCHECK / RECURSIVE

```C
pthread_mutexattr_t a;
pthread_mutexattr_init(&a);
pthread_mutexattr_settype(&a, PTHREAD_MUTEX_ERRORCHECK); // 或 RECURSIVE/NORMAL
pthread_mutex_init(&m, &a);
pthread_mutexattr_destroy(&a);
```

- `PTHREAD_MUTEX_NORMAL`：最快，但误用时可能死锁/未定义（比如同线程重复 lock）
    
- `PTHREAD_MUTEX_ERRORCHECK`：更“安全”，误用会返回错误码（便于调试）
    
- `PTHREAD_MUTEX_RECURSIVE`：同一线程可重复加锁（计数型），对应次数 unlock 才真正释放  
    适合某些递归/回调场景，但**滥用会掩盖设计问题**。
    

### 2.2 进程共享（pshared）

如果你把 mutex 放在共享内存里（shm/mmap），可让不同进程同步：

```C
pthread_mutexattr_setpshared(&a, PTHREAD_PROCESS_SHARED);
```
这个 **mutex 是否允许跨进程共享使用**（process-shared），而不是“读共享/写独占”。

普通线程内同步用默认 `PTHREAD_PROCESS_PRIVATE` 就行。

### 2.3 优先级继承/天花板（实时系统会用）

在实时/优先级调度里可能遇到**优先级反转**，mutex 属性可以设置协议（不同系统支持度不同）：

- `PTHREAD_PRIO_INHERIT`：优先级继承
    
- `PTHREAD_PRIO_PROTECT`：优先级天花板
    

（一般业务开发用不到，但学调度时会提到。）

### 2.4 robust mutex（线程崩溃时避免永久死锁）

某线程持锁期间崩溃，其他线程 lock 会得到 `EOWNERDEAD`，你需要修复共享数据并调用：

`pthread_mutex_consistent(&m);`

这是更高级的可靠性特性。

---

## 3) 读写锁 `pthread_rwlock_t`（读多写少）

### 3.1 适用

- 多线程**大量读**共享数据，写很少
    
- 允许多个读者并发，但写者独占
    

### 3.2 API

```C
#include <pthread.h>

int pthread_rwlock_init(pthread_rwlock_t *rw, const pthread_rwlockattr_t *a);
int pthread_rwlock_destroy(pthread_rwlock_t *rw);

int pthread_rwlock_rdlock(pthread_rwlock_t *rw);
int pthread_rwlock_wrlock(pthread_rwlock_t *rw);
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rw);
int pthread_rwlock_trywrlock(pthread_rwlock_t *rw);
int pthread_rwlock_unlock(pthread_rwlock_t *rw);
```
[[pthread_rwlock_读写锁函数详解]]
### 3.3 常见坑

- **写者饥饿**（读者一直来导致写者一直等）是否发生取决于实现策略
    
- 写锁临界区一长，读性能就会突然塌
    

---

## 4) 自旋锁 `pthread_spinlock_t`（短临界区、不能睡）

### 4.1 适用

- 临界区极短（几十~几百条指令级别）
    
- 线程预计很快拿到锁
    
- 不希望睡眠/唤醒的系统调用开销
    

### 4.2 API

```C
#include <pthread.h>

int pthread_spin_init(pthread_spinlock_t *s, int pshared);
int pthread_spin_destroy(pthread_spinlock_t *s);

int pthread_spin_lock(pthread_spinlock_t *s);     // 忙等
int pthread_spin_trylock(pthread_spinlock_t *s);
int pthread_spin_unlock(pthread_spinlock_t *s);
```
[[pthread_spinlock自旋锁]]
### 4.3 大坑

- 自旋锁拿不到会**烧 CPU**，在单核或锁竞争大时会非常糟糕
    
- 锁内绝对不能做阻塞操作（I/O、sleep、cond_wait）
    

> 一般用户态程序更常用 mutex；spinlock 更多是内核态/极端优化场景。

---

## 5) 条件变量 `pthread_cond_t`（配合 mutex 等“条件成立”）
[[pthread cond 条件变量函数详解]]
它不是锁，但你问“线程锁”时通常也必须一起讲，因为典型同步模式是：**mutex + cond**。

### 5.1 API

```C
int pthread_cond_wait(pthread_cond_t *c, pthread_mutex_t *m);
int pthread_cond_timedwait(pthread_cond_t *c, pthread_mutex_t *m,
                           const struct timespec *abstime);

int pthread_cond_signal(pthread_cond_t *c);
int pthread_cond_broadcast(pthread_cond_t *c);
```

### 5.2 `cond_wait` 的核心语义（必须背下来）

`pthread_cond_wait(&cv, &m)` 会做一个原子动作：

1. **释放 m**
    
2. **睡眠等待**  
    被唤醒后：
    
3. **重新获得 m**
    
4. 返回
    

所以等待必须写成：

```C
pthread_mutex_lock(&m);
while (!ready) {
    pthread_cond_wait(&cv, &m);
}
... // ready 为真
pthread_mutex_unlock(&m);
```

用 `while` 是为了防**虚假唤醒**和“条件被别人抢先改变”。

---

## 6) 工程里最常见的锁使用错误（你写多线程一定会遇到）

1. **忘记解锁**（异常路径、return 路径）
    
2. **锁顺序不一致导致死锁**（A 先锁1再锁2，B 先锁2再锁1）
    
3. **锁内做 I/O / sleep / 长计算**（吞吐和尾延迟都崩）
    
4. **把指针/结构的生命周期搞错**（锁保护的数据已经 free）
    
5. **用 trylock 忙等**（变相自旋，CPU 100%）