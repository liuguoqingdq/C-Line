# pthread 条件变量（pthread_cond_*）函数详解

条件变量（Condition Variable, `pthread_cond_t`）用于在多线程中实现“**等待某个条件成立**”。

**核心思想**：
- 条件变量本身**不保存条件是否成立的状态**，它只是一个“等待/通知”机制；
- 真正的“条件”必须由你自己维护（通常是一个共享变量/队列是否为空/计数器等），并且必须由**同一把互斥锁 `pthread_mutex_t`**保护。

> 这些函数成功一般返回 `0`；失败返回**错误码**（error number），不是 `-1`。

---

## 1) 等待（Wait）[[pthread_cond_t]]

### 1.1 `int pthread_cond_wait(pthread_cond_t *c, pthread_mutex_t *m);`

**作用**：
让当前线程在条件变量 `c` 上等待，直到被 `signal/broadcast` 唤醒。

**最关键语义（一定要背下来）**：
- 调用该函数时，线程必须**已经持有**互斥锁 `m`（即已 `pthread_mutex_lock(m)`）。
- `pthread_cond_wait` 会做一个**原子操作**：
  1) **释放 `m`**；
  2) **把线程挂到 `c` 的等待队列并睡眠**；
  3) 被唤醒后，先**重新加锁 `m`**；
  4) 然后才返回给你。

因此：`pthread_cond_wait` 返回时，调用线程**一定已经重新持有 `m`**。

**参数**
- `c`：条件变量指针（等待在哪个“条件队列”上）。
- `m`：互斥锁指针（用来保护“条件”的共享状态）。

**返回值（常见）**
- `0`：成功（注意：成功只表示“被唤醒并重新拿到了锁”，不等价于“条件一定成立”。）
- `EINVAL`：`c` 或 `m` 无效（未初始化、已销毁、损坏等）。
- `EPERM`：调用线程在调用时**没有持有**互斥锁 `m`（实现可能返回该错误）。

**必须使用 while 的原因：虚假唤醒 & 竞争**
- 条件变量允许**虚假唤醒（spurious wakeup）**：即线程可能在没有收到 signal/broadcast 的情况下返回。
- 即使确实被唤醒，返回时也不保证条件仍成立：因为多个线程竞争，可能别的线程先抢到锁并改变了状态。

所以标准写法永远是：

```c
pthread_mutex_lock(&m);
while (!condition) {
    pthread_cond_wait(&c, &m);
}
// condition 成立，安全访问共享资源
pthread_mutex_unlock(&m);
```

---

### 1.2 `int pthread_cond_timedwait(pthread_cond_t *c, pthread_mutex_t *m, const struct timespec *abstime);`

**作用**：
和 `pthread_cond_wait` 一样，但带**超时**：在指定的绝对时间 `abstime` 到达之前如果还没被唤醒，就返回超时。

**参数**
- `c`：条件变量指针。
- `m`：互斥锁指针（调用时必须已持有）。
- `abstime`：**绝对时间**（不是“等多久”）。类型为 :
```C
struct timespec { 
	time_t tv_sec; 
	long tv_nsec; //`tv_nsec` 的含义就是 “纳秒（nanoseconds）部分
	}
```
  - `tv_nsec` 范围应为 `[0, 999999999]`。
  - 绝对时间通常基于 `CLOCK_REALTIME`（具体可由条件变量属性决定）。

**返回值（常见）**
- `0`：被唤醒并重新拿到锁（不保证条件成立，仍需 while 检查）。
- `ETIMEDOUT`：到达 `abstime` 仍未等到通知。
- `EINVAL`：参数无效（如 `abstime->tv_nsec` 越界，或 `c/m` 无效）。
- `EPERM`：调用时没有持有互斥锁 `m`。

**同样需要 while**
```c
pthread_mutex_lock(&m);
while (!condition) {
    int rc = pthread_cond_timedwait(&c, &m, &abstime);
    if (rc == ETIMEDOUT) {
        // 超时退出或走降级逻辑
        break;
    }
}
pthread_mutex_unlock(&m);
```

**如何构造 abstime（示例）**：

```c
#include <time.h>

static struct timespec make_abstime_ms(long ms) {
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts); // 常见做法

    ts.tv_sec  += ms / 1000;
    ts.tv_nsec += (ms % 1000) * 1000000L;
    if (ts.tv_nsec >= 1000000000L) {
        ts.tv_sec += 1;
        ts.tv_nsec -= 1000000000L;
    }
    return ts;
}
```

---

## 2) 通知（Signal/Broadcast）

### 2.1 `int pthread_cond_signal(pthread_cond_t *c);`

**作用**：
唤醒**至少一个**正在 `c` 上等待的线程（如果当前没有等待者，这次 signal 可能什么也不发生）。

**参数**
- `c`：条件变量指针。

**返回值（常见）**
- `0`：成功。
- `EINVAL`：`c` 无效。

**重要理解**
- “唤醒”并不等于“马上运行并进入临界区”。
- 被唤醒的线程会去竞争互斥锁 `m`；只有抢到锁后才能从 `wait` 返回。

**是否必须持锁再 signal？**
- 规范上不强制，但**强烈建议**在持有 `m` 的情况下：
  1) 修改条件（共享状态）
  2) 再 signal
  3) 再 unlock

这样可以避免“丢失唤醒/竞态窗口”。

---

### 2.2 `int pthread_cond_broadcast(pthread_cond_t *c);`

**作用**：
唤醒**所有**正在 `c` 上等待的线程。

**参数**
- `c`：条件变量指针。

**返回值（常见）**
- `0`：成功。
- `EINVAL`：`c` 无效。

**使用场景**
- 条件变化会让**多个等待者**都可能继续（例如：从“停止状态”切换为“运行状态”，所有工作线程都要醒来）。
- 或者你无法/不想判断应该唤醒几个线程。

**副作用：惊群效应（thundering herd）**
- 广播会把所有等待线程都唤醒，但它们仍要争同一把 mutex，很多线程会被唤醒后又马上睡回去，造成大量上下文切换与竞争。
- 能用 `signal` 唤醒一个就不要 `broadcast`。

---

## 3) 经典生产者-消费者模板（必会）

假设共享队列 `q`，条件是“队列非空”。

### 消费者：
```c
pthread_mutex_lock(&m);
while (q_empty()) {
    pthread_cond_wait(&c, &m);
}
item = q_pop();
pthread_mutex_unlock(&m);
```

### 生产者：
```c
pthread_mutex_lock(&m);
q_push(item);
// 条件从“空”变为“非空”，唤醒一个等待者即可
pthread_cond_signal(&c);
pthread_mutex_unlock(&m);
```

**要点总结**
1. 条件变量永远配合 mutex 使用。
2. `wait/timedwait` 调用时 mutex 必须已加锁。
3. `wait/timedwait` 返回时 mutex 仍处于加锁状态。
4. 永远用 `while` 检查条件，而不是 `if`。
5. `signal` 唤醒一个，`broadcast` 唤醒全部；broadcast 可能引起惊群。

如果你接下来还需要：`pthread_cond_init/destroy`、`pthread_condattr_setclock`（超时使用单调时钟避免系统时间跳变）以及和 `pthread_mutex` 的组合细节，我也可以继续整理。

