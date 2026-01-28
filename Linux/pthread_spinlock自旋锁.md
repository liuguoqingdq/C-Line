# pthread 自旋锁（pthread_spin_*）函数详解

自旋锁（Spin Lock, `pthread_spinlock_t`）是一种**忙等**锁：
- **拿不到锁就循环自旋**（不断检查锁是否可用），而不是像 mutex 那样把线程挂起睡眠。
- 适合**临界区极短**、竞争不大、CPU 核数充足的场景；否则会白白烧 CPU。

> 这些函数成功一般返回 `0`；失败返回**错误码**（error number），而不是 `-1`。([pubs.opengroup.org](https://pubs.opengroup.org/onlinepubs/009695399/functions/pthread_spin_destroy.html?utm_source=chatgpt.com))

---

## 1) 初始化与销毁

### 1.1 `int pthread_spin_init(pthread_spinlock_t *s, int pshared);`

**作用**：初始化一个自旋锁对象，使其可用。

**参数**
- `s`：要初始化的自旋锁对象地址。
- `pshared`：是否允许**进程间共享**，取值通常是：
  - `PTHREAD_PROCESS_PRIVATE`：仅同一进程内线程共享。
  - `PTHREAD_PROCESS_SHARED`：允许不同进程共享（前提是锁对象放在共享内存中）。

**非常重要：实现差异**
- POSIX/部分系统支持 `PTHREAD_PROCESS_SHARED`（不支持则可能返回 `ENOTSUP`）。([man.openbsd.org](https://man.openbsd.org/pthread_spin_init.3?utm_source=chatgpt.com))
- 但在 Linux/glibc 上，手册明确提示：把自旋锁跨进程共享可能是**未定义行为**（并非可靠特性）。([man7.org](https://man7.org/linux/man-pages/man3/pthread_spin_init.3.html?utm_source=chatgpt.com))

**返回值（常见错误码）**
- `0`：成功。
- `EAGAIN`：系统资源不足，无法再初始化新的自旋锁。([pubs.opengroup.org](https://pubs.opengroup.org/onlinepubs/009695399/functions/pthread_spin_destroy.html?utm_source=chatgpt.com))
- `ENOMEM`：内存不足。([pubs.opengroup.org](https://pubs.opengroup.org/onlinepubs/009695399/functions/pthread_spin_destroy.html?utm_source=chatgpt.com))
- `ENOTSUP`：`pshared` 指定的共享属性不被实现支持。([man.openbsd.org](https://man.openbsd.org/pthread_spin_init.3?utm_source=chatgpt.com))
- `EINVAL`：参数无效（比如 `s` 不合法，或 `pshared` 值不在允许范围）。([docs.oracle.com](https://docs.oracle.com/cd/E53394_01/html/E54803/ggecq.html?utm_source=chatgpt.com))

**使用要点**
- 对同一个 `s` 重复 `init`、或未 `init` 就使用，结果可能未定义。([sourceware.org](https://sourceware.org/pthreads-win32/manual/pthread_spin_init.html?utm_source=chatgpt.com))

---

### 1.2 `int pthread_spin_destroy(pthread_spinlock_t *s);`

**作用**：销毁自旋锁，释放其内部资源。

**参数**
- `s`：要销毁的自旋锁对象地址。

**返回值（常见错误码）**
- `0`：成功。
- `EBUSY`：锁仍在使用中（例如仍被某线程持有，或另一个线程正在 `pthread_spin_lock()` 上使用它）。([pubs.opengroup.org](https://pubs.opengroup.org/onlinepubs/009695399/functions/pthread_spin_destroy.html?utm_source=chatgpt.com))
- `EINVAL`：锁对象无效/未初始化。([pubs.opengroup.org](https://pubs.opengroup.org/onlinepubs/009695399/functions/pthread_spin_destroy.html?utm_source=chatgpt.com))

**使用要点**
- `destroy` **不是强制解锁**；必须确保无人持锁、无人再用。([pubs.opengroup.org](https://pubs.opengroup.org/onlinepubs/009695399/functions/pthread_spin_destroy.html?utm_source=chatgpt.com))

---

## 2) 加锁（忙等）

### 2.1 `int pthread_spin_lock(pthread_spinlock_t *s);`

**作用**：获取自旋锁。
- 如果锁空闲 → 立即获得。
- 如果锁被占用 → 调用线程会在用户态/内核态实现所定义的方式中**持续自旋等待**，直到获得锁。([pubs.opengroup.org](https://pubs.opengroup.org/onlinepubs/009695299/functions/pthread_spin_lock.html?utm_source=chatgpt.com))

**参数**
- `s`：自旋锁对象地址。

**返回值（常见错误码）**
- `0`：成功获得锁。
- `EINVAL`：`s` 无效或未初始化。([man.openbsd.org](https://man.openbsd.org/pthread_spin_lock.3?utm_source=chatgpt.com))
- `EDEADLK`：检测到死锁（最常见情况：**同一线程重复加同一把自旋锁**，自旋锁不支持递归）。([man.openbsd.org](https://man.openbsd.org/pthread_spin_lock.3?utm_source=chatgpt.com))

**使用要点（很关键）**
- **临界区必须非常短**：锁内不要 `sleep`、不要做 IO、不要等待外部事件，否则会让别的线程一直烧 CPU。
- 在单核或高竞争下，自旋锁可能比 mutex 更慢。

---

### 2.2 `int pthread_spin_trylock(pthread_spinlock_t *s);`

**作用**：尝试获取自旋锁，但**不阻塞**。

**返回值（常见错误码）**
- `0`：成功获得锁。
- `EBUSY`：锁当前被别的线程持有（最常见）。([hpc-tutorials.llnl.gov](https://hpc-tutorials.llnl.gov/posix/man/pthread_spin_lock.txt?utm_source=chatgpt.com))
- `EINVAL`：锁无效或未初始化。([man.openbsd.org](https://man.openbsd.org/pthread_spin_lock.3?utm_source=chatgpt.com))
- `EDEADLK`：调用线程已经持有该锁（实现可能检测并返回）。([man.openbsd.org](https://man.openbsd.org/pthread_spin_lock.3?utm_source=chatgpt.com))

**典型用途**
- “尽力而为”路径：拿不到就走替代策略（比如稍后重试、改用队列、合并写入等），避免长时间自旋。

---

## 3) 解锁

### `int pthread_spin_unlock(pthread_spinlock_t *s);`

**作用**：释放当前线程持有的自旋锁。
- 若有其他线程正在自旋等待，其中某个线程会获得锁（一般不保证公平）。([pubs.opengroup.org](https://pubs.opengroup.org/onlinepubs/007904875/functions/pthread_spin_unlock.html?utm_source=chatgpt.com))

**返回值（常见错误码）**
- `0`：成功。
- `EINVAL`：参数无效/锁对象不合法。([pubs.opengroup.org](https://pubs.opengroup.org/onlinepubs/007904875/functions/pthread_spin_unlock.html?utm_source=chatgpt.com))
- `EPERM`：调用线程并未持有该锁。([pubs.opengroup.org](https://pubs.opengroup.org/onlinepubs/007904875/functions/pthread_spin_unlock.html?utm_source=chatgpt.com))

**注意：不要依赖“解锁错误检查”**
- 有的实现（例如 Linux 手册页措辞）把“非持有者解锁”描述为**未定义行为**，也就是你不能指望一定返回 `EPERM`。最稳妥做法是：从逻辑上保证只由持锁线程解锁。([man7.org](https://man7.org/linux/man-pages/man3/pthread_spin_lock.3.html?utm_source=chatgpt.com))

---

## 4) 最小示例：自旋锁保护极短临界区

```c
#include <pthread.h>
#include <stdio.h>

static pthread_spinlock_t g_spin;
static long g_counter = 0;

void* worker(void* arg) {
    (void)arg;
    for (int i = 0; i < 1000000; ++i) {
        pthread_spin_lock(&g_spin);
        // 临界区要尽量短：只做共享变量的快速更新
        ++g_counter;
        pthread_spin_unlock(&g_spin);
    }
    return NULL;
}

int main() {
    // 通常使用 PTHREAD_PROCESS_PRIVATE
    if (pthread_spin_init(&g_spin, PTHREAD_PROCESS_PRIVATE) != 0) {
        perror("pthread_spin_init");
        return 1;
    }

    pthread_t t1, t2;
    pthread_create(&t1, NULL, worker, NULL);
    pthread_create(&t2, NULL, worker, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("counter=%ld\n", g_counter);

    pthread_spin_destroy(&g_spin);
    return 0;
}
```

---

## 5) 面试/笔试高频对比点（Spin vs Mutex）

1. **阻塞方式**：Spin 忙等；Mutex 一般会睡眠/唤醒（内核参与）→ 临界区长时 mutex 更合适。
2. **适用场景**：Spin 适合“非常短 + 高频”；Mutex 适合“一般临界区”。
3. **可组合性**：条件变量（`pthread_cond_*`）要求配合 mutex；自旋锁不适合做条件等待。
4. **公平性/饥饿**：自旋锁通常不保证公平，高竞争下可能导致某些线程长期抢不到。

如果你愿意，我可以给你一张“选锁决策表”（mutex / rwlock / spinlock / semaphore）以及常见反例（什么时候用自旋锁必翻车）。

