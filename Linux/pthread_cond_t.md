
## 1）`pthread_cond_t` 是什么？

`pthread_cond_t` 是 **条件变量对象**（condition variable）。它本身不代表“条件是否成立”，它只提供两件事：

- **等待**：线程在这里睡眠，直到有人通知它
    
- **通知**：另一个线程发信号把等待者唤醒
    

> 条件变量不存状态；真正的条件是你自己维护的共享变量（比如队列是否为空、计数是否为 0、是否可以继续运行等）。

---

## 2）怎么“创建/初始化”一个条件变量？

两种方式：

### A. 静态初始化（最简单）

```C
pthread_cond_t  cond = PTHREAD_COND_INITIALIZER;
pthread_mutex_t mtx  = PTHREAD_MUTEX_INITIALIZER;
```

### B. 动态初始化（用 init / destroy）

```C
pthread_cond_t  cond;
pthread_mutex_t mtx;

pthread_cond_init(&cond, NULL);
pthread_mutex_init(&mtx, NULL);

// 用完销毁
pthread_cond_destroy(&cond);
pthread_mutex_destroy(&mtx);
```

---

## 3）条件变量怎么用？（必须配一把 mutex）

条件变量 **必须和互斥锁一起用**。原因是：你要“检查条件 + 进入等待”这一套操作是线程安全的。

最经典的例子：**生产者-消费者（队列非空条件）**

### 共享状态

```C
pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  cond = PTHREAD_COND_INITIALIZER;

int has_item = 0;   // 这就是“条件”，必须由 mtx 保护
int item = 0;
```

### 消费者：等到有数据再拿

```C
pthread_mutex_lock(&mtx);

// 必须用 while，不要用 if（虚假唤醒 + 多线程竞争）
while (!has_item) {
    pthread_cond_wait(&cond, &mtx);
}

// 能走到这里说明：在拿着锁的情况下，has_item 为真
printf("consume item=%d\n", item);
has_item = 0;

pthread_mutex_unlock(&mtx);
```

### 生产者：放入数据并通知

```C
pthread_mutex_lock(&mtx);

item = 42;
has_item = 1;

// 通知一个等待线程即可
pthread_cond_signal(&cond);

pthread_mutex_unlock(&mtx);
```

---

## 4）`pthread_cond_wait` 到底干了什么？（你必须理解的语义）

当线程调用：

```C
pthread_cond_wait(&cond, &mtx);
```

它会做一个**原子动作**：

1. **释放 mtx**
    
2. 把线程挂到 cond 的等待队列上并睡眠
    
3. 被唤醒后，先**重新加锁 mtx**
    
4. 然后返回
    

所以：**`pthread_cond_wait` 返回时，你仍然持有 mtx**。

---

## 5）为什么一定要 while？

因为两件事都可能发生：

- **虚假唤醒**：线程可能没被 signal 也返回（标准允许）
    
- **被唤醒不等于条件仍成立**：多个消费者都被唤醒后，只有一个能先拿到锁并消费，其他线程拿到锁时条件可能又变回假
    

所以必须：

```C
while (!condition) wait...
```

---

## 6）什么时候用 signal，什么时候用 broadcast？

- `pthread_cond_signal`：唤醒**一个**等待者（通常队列入一个元素就够了）
    
- `pthread_cond_broadcast`：唤醒**所有**等待者（比如“系统状态从暂停→运行”、或者线程池关闭，需要全部醒来退出）
    

---

## 那如果我有多个条件是不是可以创建多个pthread_cond_t 队列?

可以，而且这正是条件变量最常见的用法之一：**多个条件 → 你可以建多个 `pthread_cond_t`（多个等待队列）**。每个条件变量代表一类“等待事件/状态变化”，谁关心谁就等在哪个队列上。

不过要注意两条核心原则：

1. **条件变量不存条件**，条件还是你自己的共享状态变量
    
2. **每次 wait / signal 都要围绕同一把 mutex 来保护这些共享状态**（可以是一把 mutex 保护多个条件，也可以拆分）
    

---

## 典型例子：有界队列（同时有两个条件）

有界缓冲区最经典：

- 条件1：队列 **非空**（消费者才能取） → `cond_not_empty`
    
- 条件2：队列 **未满**（生产者才能放） → `cond_not_full`
    

```C
pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  not_empty = PTHREAD_COND_INITIALIZER;
pthread_cond_t  not_full  = PTHREAD_COND_INITIALIZER;

int count = 0;
const int CAP = 10;
```

生产者（等“未满”）：

```C
pthread_mutex_lock(&mtx);
while (count == CAP) {
    pthread_cond_wait(&not_full, &mtx);
}
// push
count++;
pthread_cond_signal(&not_empty);   // 现在“非空”成立，叫醒消费者
pthread_mutex_unlock(&mtx);
```

消费者（等“非空”）：

```C
pthread_mutex_lock(&mtx);
while (count == 0) {
    pthread_cond_wait(&not_empty, &mtx);
}
// pop
count--;
pthread_cond_signal(&not_full);    // 现在“未满”成立，叫醒生产者
pthread_mutex_unlock(&mtx);
```

这就是“多个条件变量队列”的标准模式。

---

## 一把 mutex + 多个 cond，还是多个 mutex？

两种都可以，但怎么选有规律：

### ✅ 常用：一把 mutex 保护同一份共享状态 + 多个 cond

适合：这些条件都依赖同一份状态（同一个队列/同一个对象的多个状态位）。  
优点：简单、不容易出错（避免跨锁的竞态）。

### ✅ 拆分：不同状态块用不同 mutex + 各自 cond

适合：共享状态能明确切分、并发要求很高。  
缺点：复杂、容易出现“先拿 A 锁看状态，再去等 B cond”这种错误，导致丢唤醒或死锁。

---

## 重要提醒：多个条件≠必须多个 cond（有时一个也够）

你也可以只用一个 `cond`，但会有两个问题：

- 你无法精确叫醒“需要的人”，容易叫醒不相关线程
    
- 可能要用 `broadcast`，更容易“惊群”
    

所以：**条件确实不同、等待者集合也不同**时，多个 `pthread_cond_t` 更合理。

---

## 实战建议（你可以直接记）

- “队列空/满”“任务到来/关闭”“资源可用/状态改变”这类**不同等待原因** → 分开用多个 cond
    
- 修改共享状态时：**先加锁 → 改状态 → signal/broadcast → 解锁**
    
- wait 永远写 `while`，别写 `if`