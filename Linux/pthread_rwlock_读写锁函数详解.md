# pthread 读写锁（pthread_rwlock_*）函数详解

读写锁（Read-Write Lock, `pthread_rwlock_t`）用于保护共享资源：
- **读锁（shared lock）**：允许多个线程同时“只读”进入临界区。
- **写锁（exclusive lock）**：写者独占进入；写锁持有期间，其他读/写者都不能进入。

典型场景：**读多写少**。如果读写比例接近、临界区很短，读写锁未必比互斥锁快（实现更复杂、开销可能更大）。

> 下面所有函数：成功通常返回 `0`；失败返回 **错误码**（不是 `-1`），例如 `EINVAL`、`EBUSY`、`EDEADLK` 等。

---

## 1) 初始化与销毁

### 1.1 `int pthread_rwlock_init(pthread_rwlock_t *rw, const pthread_rwlockattr_t *a);`

**作用**：初始化一个读写锁对象，使其可用。

**参数**
- `rw`：输出/就地初始化的读写锁对象地址。
- `a`：读写锁属性对象指针。
  - 传 `NULL` 表示使用默认属性（通常是 `PTHREAD_PROCESS_PRIVATE`，仅线程间共享）。

**返回值**
- `0`：成功。
- 非 `0`：错误码，常见：
  - `EINVAL`：参数无效（例如 `rw` 指针不合法，或属性内容不合法）。
  - `ENOMEM`：资源不足（实现可能需要分配内部资源）。

**使用要点**
- **静态初始化**：很多实现还支持 `pthread_rwlock_t rw = PTHREAD_RWLOCK_INITIALIZER;`（无需显式 `init`），但若需要非默认属性，仍要 `init`。
- 初始化后才能 `rdlock/wrlock`。

---

### 1.2 `int pthread_rwlock_destroy(pthread_rwlock_t *rw);`

**作用**：销毁读写锁，释放其内部资源。

**参数**
- `rw`：要销毁的读写锁对象地址。

**返回值**
- `0`：成功。
- 非 `0`：错误码，常见：
  - `EBUSY`：锁仍被某些线程持有（仍处于读锁或写锁状态），或仍在被使用。
  - `EINVAL`：锁对象无效（未初始化/已销毁/损坏）。

**使用要点**
- 只能在确保**没有任何线程持锁**、也不会再使用该锁时调用。
- 和 `pthread_mutex_destroy` 类似，`destroy` 不是“强制解锁”。

---

## 2) 加锁（阻塞版）

### 2.1 `int pthread_rwlock_rdlock(pthread_rwlock_t *rw);`

**作用**：申请读锁。
- 若当前**没有写锁持有者**，则可能立即获得读锁；
- 若有写锁持有者，或实现策略阻止读者插队（避免写者饥饿），调用线程会**阻塞等待**。

**参数**
- `rw`：读写锁对象地址。

**返回值**
- `0`：成功获得读锁。
- 非 `0`：错误码，常见：
  - `EINVAL`：锁无效。
  - `EDEADLK`：检测到潜在死锁（例如同一线程在某些实现中错误地重入写锁/读锁组合）。

**注意**
- POSIX 对“公平性/优先级（读优先还是写优先）”通常**不强制**，不同实现可能不同。

---

### 2.2 `int pthread_rwlock_wrlock(pthread_rwlock_t *rw);`

**作用**：申请写锁（独占锁）。
- 只有当**没有任何读者**、也**没有其他写者**持锁时，才能获得。
- 否则调用线程会**阻塞等待**直到可用。

**参数**
- `rw`：读写锁对象地址。

**返回值**
- `0`：成功获得写锁。
- 非 `0`：错误码，常见：
  - `EINVAL`：锁无效。
  - `EDEADLK`：死锁风险（例如同一线程已持有读锁又请求写锁升级，许多实现会认为可能死锁）。

**重要概念：升级/降级**
- **升级（读→写）**：POSIX 并不保证安全可用，很多实现会导致死锁或直接返回 `EDEADLK`。
  - 正确做法通常是：先 `unlock` 读锁，再尝试 `wrlock`（但这不是原子升级，期间可能被其他线程插入）。
- **降级（写→读）**：也没有统一原子 API；常见策略是持写锁时先再加读锁（是否允许取决于实现），再释放写锁。更可移植的方法是：释放写锁后再加读锁（同样非原子）。

---

## 3) 加锁（非阻塞 try 版）

### 3.1 `int pthread_rwlock_tryrdlock(pthread_rwlock_t *rw);`

**作用**：尝试获取读锁，**不阻塞**。
- 如果当前可立即获得读锁 → 成功。
- 如果不能立即获得（通常因为有写锁，或策略不允许） → 立即返回失败。

**参数**
- `rw`：读写锁对象地址。

**返回值**
- `0`：成功获得读锁。
- `EBUSY`：当前无法立即获得锁（最常见）。
- `EINVAL`：锁无效。

**典型用途**
- 做“尽力而为”的读访问：拿不到就走降级路径（比如返回缓存、稍后重试、或者做别的工作）。

---

### 3.2 `int pthread_rwlock_trywrlock(pthread_rwlock_t *rw);`

**作用**：尝试获取写锁，**不阻塞**。

**参数**
- `rw`：读写锁对象地址。

**返回值**
- `0`：成功获得写锁。
- `EBUSY`：当前有读者/写者持锁，无法立即获得。
- `EINVAL`：锁无效。

**典型用途**
- 避免写线程在热点锁上长时间阻塞：拿不到就先做别的、或者合并更新、或者采用队列化写入。

---

## 4) 解锁

### 4.1 `int pthread_rwlock_unlock(pthread_rwlock_t *rw);`

**作用**：释放当前线程持有的读锁或写锁。
- 如果释放的是写锁：可能唤醒等待的读者/写者（取决于实现策略）。
- 如果释放的是读锁：当最后一个读者释放后，可能唤醒等待写者。

**参数**
- `rw`：读写锁对象地址。

**返回值**
- `0`：成功。
- 非 `0`：错误码，常见：
  - `EPERM`：当前线程并未持有该锁，却调用了解锁（实现可能返回该错误）。
  - `EINVAL`：锁无效。

**注意**
- 解锁必须与加锁匹配：
  - 成功 `rdlock/tryrdlock` 后必须 `unlock` 一次；
  - 成功 `wrlock/trywrlock` 后也必须 `unlock` 一次。

---

## 5) 最小可运行示例（读多写少）

下面示例：多个读线程读取 `g_value`，写线程偶尔更新。

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

static pthread_rwlock_t g_rw = PTHREAD_RWLOCK_INITIALIZER;
static int g_value = 0;

void* reader(void* arg) {
    (void)arg;
    for (int i = 0; i < 5; i++) {
        pthread_rwlock_rdlock(&g_rw);
        int v = g_value;          // 读共享数据
        pthread_rwlock_unlock(&g_rw);

        printf("reader: %d\n", v);
        usleep(100 * 1000);
    }
    return NULL;
}

void* writer(void* arg) {
    (void)arg;
    for (int i = 0; i < 5; i++) {
        pthread_rwlock_wrlock(&g_rw);
        g_value++;                // 写共享数据
        pthread_rwlock_unlock(&g_rw);

        printf("writer: updated\n");
        usleep(250 * 1000);
    }
    return NULL;
}

int main() {
    pthread_t r1, r2, w;
    pthread_create(&r1, NULL, reader, NULL);
    pthread_create(&r2, NULL, reader, NULL);
    pthread_create(&w,  NULL, writer, NULL);

    pthread_join(r1, NULL);
    pthread_join(r2, NULL);
    pthread_join(w,  NULL);

    // 若不是 PTHREAD_RWLOCK_INITIALIZER 初始化，而是 init，则最后 destroy
    pthread_rwlock_destroy(&g_rw);
    return 0;
}
```

**读写锁使用规则总结**
1. 只读临界区：`rdlock` / `unlock`。
2. 修改临界区：`wrlock` / `unlock`。
3. `try*` 版本：拿不到返回 `EBUSY`，自己决定退路。
4. `destroy` 前必须保证无人持锁，否则 `EBUSY`。

---

## 6) 常见坑与面试/笔试点

- **错误码返回**：pthread 大多返回错误码，不一定设置 `errno`。
- **公平性**：不同实现可能读优先或写优先；写者可能饥饿或读者可能被阻塞。
- **升级死锁**：读锁升级写锁通常不保证安全；需要谨慎设计。
- **性能误区**：读写锁不一定比 mutex 快，尤其在竞争不大或临界区极短时。

如果你还需要 `pthread_rwlockattr_t`（比如 `pshared`、进程间共享）那一套属性函数的解释，我也可以把整套属性 API 一并整理。

