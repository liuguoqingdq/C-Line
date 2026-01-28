# 第6章 高级进程管理（6.1–6.7）详细讲解

这一章的关键词是：**“谁运行、运行多久、在哪个 CPU 上跑、遇到竞争怎么让、以及系统如何限制进程消耗资源”**。

从系统编程角度，建议你把它拆成两条主线：
1) **调度相关**：策略（policy）+ 优先级（priority）+ 让出 CPU（yield）+ CPU 亲和性（affinity）+ 实时（RT）
2) **资源治理**：rlimit / ulimit（以及更现代的 cgroups）

---

## 6.1 进程调度（Scheduling）

### 6.1.1 你必须先建立的调度模型
Linux 的调度要解决两个问题：
- **选择谁运行**：从 runnable（可运行）队列里挑一个任务
- **什么时候切换**：时间片用完、任务阻塞、被抢占、更高优先级到来等

在现代 Linux（多核）里，每个 CPU 都有自己的运行队列（runqueue），核心思路是：
- 尽量让任务在**本地 CPU** 上运行（减少迁移成本、cache 热度更好）
- 必要时做负载均衡（load balancing）

### 6.1.2 调度类（sched class）与调度策略（policy）
从“用户态能选择什么”的角度，你常接触到的策略是：
- **普通进程**：`SCHED_OTHER`（也叫 CFS 体系）
- **批处理**：`SCHED_BATCH`（更少抢占/更偏吞吐）
- **极低优先**：`SCHED_IDLE`
- **实时**：`SCHED_FIFO`、`SCHED_RR`
- **截止期**：`SCHED_DEADLINE`（更高级，常见于特定 RT 场景）

系统编程常用 API：
```c
#include <sched.h>
int sched_getscheduler(pid_t pid);
int sched_setscheduler(pid_t pid, int policy, const struct sched_param *param);
int sched_setparam(pid_t pid, const struct sched_param *param);
int sched_getparam(pid_t pid, struct sched_param *param);
int sched_get_priority_max(int policy);
int sched_get_priority_min(int policy);
```

> 记住：**policy 决定“规则”，priority/nice 决定“在规则内的位置”**。

### 6.1.3 普通程序如何“观察”调度状态
- `ps -o pid,cls,pri,ni,stat,psr,comm -p <pid>`：看调度类、优先级、nice、跑在哪个 CPU
- `top/htop`：观察 NI/PR
- `chrt -p <pid>`：查看/设置实时策略

---

## 6.2 完全公平调度器（CFS, Completely Fair Scheduler）

### 6.2.1 CFS 想“公平”什么？
CFS 服务的对象主要是 **普通进程**（`SCHED_OTHER` 等）。它的公平是：
- **按权重分配 CPU 时间**：权重由 `nice` 间接决定
- 目标：在足够长时间窗口内，每个任务获得的 CPU 时间 ≈ 它应得的份额

### 6.2.2 核心概念：`vruntime`（虚拟运行时间）
CFS 不直接用“真实运行时间”排序，而是用 `vruntime`：
- 任务跑得越多 → `vruntime` 增长越多
- **权重越高（nice 越小）** → 同样跑一段真实时间，`vruntime` 增长得更慢

因此：
- CFS 总是挑 **vruntime 最小** 的任务运行（“亏欠最多的人先跑”）

### 6.2.3 数据结构：红黑树（RB-tree）
可运行任务以 `vruntime` 为 key 放入红黑树：
- 取最小 vruntime：树的最左节点（O(logN) 插入/删除）
- 这比传统 O(N) 遍历更适合大量任务

### 6.2.4 时间片不是固定的：由目标延迟与最小粒度控制
传统调度器常用固定时间片；CFS 更像：
- 设一个“目标调度延迟”（target latency）：希望在这个时间里让所有 runnable 任务至少跑一次
- 但同时又受“最小粒度”（min granularity）约束，避免任务太多导致每个任务时间片小到不可用

### 6.2.5 nice 的意义
`nice` 影响权重，从而影响“你占 CPU 的份额”。
- nice 值范围通常是 **-20（更优先）到 +19（更不优先）**
- 普通用户通常只能把 nice 调大（变“更友好”），降低 nice（更抢资源）需要权限

常见 API：
```c
#include <sys/time.h>
#include <sys/resource.h>
int getpriority(int which, id_t who);
int setpriority(int which, id_t who, int prio);
```

> 注意：`getpriority` 返回值范围与 errno 的交互容易踩坑（-1 可能是合法值，需先清 errno 再调用）。

---

## 6.3 让出处理器（Yield）

### 6.3.1 “让出”是什么意思
让出 CPU 是一种 **自愿放弃当前可运行资格的行为**，常用于：
- 你在忙等，但希望别把 CPU 占满
- 你是低优先级线程，愿意让其他线程先跑

Linux 常用接口：
```c
#include <sched.h>
int sched_yield(void);
```

### 6.3.2 `sched_yield()` 的真实效果（要讲清楚）
`sched_yield()` 并不是“睡一会儿”，而是：
- **把当前任务放回其调度队列尾部（或相应位置）**，触发一次重新挑选
- 若系统里没有更合适的可运行任务，你可能**马上又被选中**（因此它不保证你会“让出很久”）

### 6.3.3 什么时候不该用 yield
- 用 yield 解决同步问题通常是“坏味道”：正确做法是 mutex/condvar/futex/epoll 等阻塞式等待
- yield 可能导致：
  - 无谓的上下文切换
  - CPU 利用率异常
  - 在实时调度下产生意外的调度行为

### 6.3.4 更推荐的替代
- 等待事件：`pthread_cond_wait` / `futex` / `epoll_wait` / `select/poll`
- 需要“轻量退让”时：短自旋 + `nanosleep` 或 “自旋次数超阈值后阻塞”

---

## 6.4 进程优先级（Priority）

Linux 里“优先级”有两套体系，必须分清：

### 6.4.1 普通优先级：nice（影响 CFS 权重）
- 适用于 `SCHED_OTHER/BATCH/IDLE`
- nice 不是“硬优先级”，而是“份额权重”

工具：
- `nice -n 10 cmd`：以更低优先级启动
- `renice +5 -p <pid>`：调整正在运行的进程 nice

### 6.4.2 实时优先级：RT priority（1–99）
- 适用于 `SCHED_FIFO`、`SCHED_RR`
- 这是“硬优先级”：高 RT priority 进程会抢占低的

API：
```c
#include <sched.h>
struct sched_param sp = {.sched_priority = 50};
sched_setscheduler(pid, SCHED_FIFO, &sp);
```

> 现实警告：RT 优先级用不好会让系统“卡死感”明显（低优先级任务得不到运行机会）。

### 6.4.3 `SCHED_RR` vs `SCHED_FIFO`
- `SCHED_FIFO`：同优先级下，谁先运行谁一直跑，除非阻塞/主动让出/被更高优先级抢占
- `SCHED_RR`：同优先级下按时间片轮转，更“公平”一些

---

## 6.5 处理器亲和力（CPU Affinity）

### 6.5.1 亲和力是什么
CPU 亲和力就是：**限制/指定进程（或线程）可以在哪些 CPU 核上运行**。

为什么要做亲和力？
- cache 热度：固定在某核上，缓存命中更好
- 减少迁移：线程迁移会带来 cache/TLB 失效
- NUMA：把线程绑到靠近内存的节点上能显著影响延迟
- 隔离：把 noisy 任务绑到某些核，把关键任务留在其它核

### 6.5.2 系统调用接口
```c
#define _GNU_SOURCE
#include <sched.h>

int sched_setaffinity(pid_t pid, size_t cpusetsize, const cpu_set_t *mask);
int sched_getaffinity(pid_t pid, size_t cpusetsize, cpu_set_t *mask);
```

常见宏：
```c
CPU_ZERO(&set);
CPU_SET(3, &set);
CPU_ISSET(3, &set);
```

命令行工具：
- `taskset -p <mask> <pid>`
- `taskset -c 0,2,4 cmd`

### 6.5.3 线程亲和性（pthread）
很多场景你是线程级控制：
- `pthread_setaffinity_np()`（非标准但常见）

### 6.5.4 工程注意点
- 绑核并不总是更快：绑得不合理会导致某核过载、负载均衡失效
- 服务器程序常用策略：
  - I/O 线程与 worker 线程分离
  - 关键线程绑“干净核”，把噪声任务放到其它核

---

## 6.6 实时系统（Real-time）

### 6.6.1 实时的真正含义
“实时”不是“更快”，而是：
- **可预测**：在限定时间内完成（deadline）
- 关注的是延迟上界（worst-case latency），而不是平均性能

Linux 提供“软实时”能力：
- 普通内核也能做一定实时，但不保证极端条件下的严格上界
- PREEMPT_RT 补丁/实时内核可进一步改善（更强实时性）

### 6.6.2 实时调度策略
- `SCHED_FIFO` / `SCHED_RR`：经典 RT
- `SCHED_DEADLINE`：按运行时间/周期/截止期建模（更先进）

### 6.6.3 实时程序的常见工程措施
1) **优先级设计**：
- 关键路径任务高优先级
- 低优先级任务必须能让路

2) **避免缺页/换页抖动**：
- 用 `mlockall(MCL_CURRENT | MCL_FUTURE)` 锁内存（防止页被换出）

3) **减少不可控阻塞**：
- 避免在 RT 线程里做 IO、malloc、锁粒度过大
- 采用无锁结构或短临界区

4) **防止饿死（starvation）**：
- RT 线程可能让普通线程长期得不到 CPU
- 需要谨慎设置 RT priority、并考虑 watchdog/超时

### 6.6.4 常用工具
- `chrt`：查看/设置实时策略与优先级
- `cyclictest`（在 RT 评估里常见）：测调度延迟

---

## 6.7 资源限制（Resource Limits, rlimit / ulimit）

### 6.7.1 rlimit 是什么
rlimit 是内核提供的一种“进程资源上限”机制：
- 限制进程可消耗的某类资源（CPU 时间、文件描述符数、最大文件大小、地址空间等）
- 通常有两档：
  - **soft limit（软限制）**：可触发信号或返回错误
  - **hard limit（硬限制）**：软限制的上限，普通进程不能随意提高

API：
```c
#include <sys/resource.h>
int getrlimit(int resource, struct rlimit *rlim);
int setrlimit(int resource, const struct rlimit *rlim);
```

### 6.7.2 常见 resource 项（高频）
- `RLIMIT_NOFILE`：最多打开文件描述符数（网络服务器特别重要）
- `RLIMIT_NPROC`：用户可创建进程数（防 fork bomb）
- `RLIMIT_CORE`：core dump 最大大小（调试崩溃）
- `RLIMIT_CPU`：CPU 时间上限（超出通常 SIGXCPU）
- `RLIMIT_AS`：虚拟内存地址空间上限
- `RLIMIT_STACK`：栈大小
- `RLIMIT_FSIZE`：单个文件最大大小
- `RLIMIT_MEMLOCK`：可锁定内存上限（实时程序 mlockall 相关）
- `RLIMIT_RTPRIO` / `RLIMIT_RTTIME`：实时相关限制（不同系统支持不同）

### 6.7.3 `ulimit` 与系统配置
- shell 的 `ulimit -n` 影响当前 shell 及其子进程
- 生产上还常需要改：
  - `/etc/security/limits.conf`（pam_limits）
  - systemd service 的 `LimitNOFILE=` 等

### 6.7.4 资源限制在工程中的典型用法
1) **防止资源失控**
- 限制最大打开 fd、防止泄漏拖垮系统
- 限制最大进程数、防止 fork bomb

2) **控制 core dump**
- 线上默认可关（RLIMIT_CORE=0）
- 需要排查崩溃再打开并配合 core_pattern

3) **给实时/高性能程序配置上限**
- 提高 `RLIMIT_NOFILE`，否则 accept/epoll 可能很快撞上上限
- 若要 `mlockall`，需要足够 `RLIMIT_MEMLOCK`

### 6.7.5 补充：cgroups（现代资源治理）
rlimit 是“进程级/用户级”的传统机制。
在容器/服务治理中，更多用 **cgroups** 控 CPU/内存/IO（更强大、可层级管理）。
但学习 rlimit 仍非常重要：它是系统编程里最常见的“自我保护阀门”。

---

# 本章串联：从“写服务程序”的视角看 6.1–6.7

假设你写一个网络服务：
1) `RLIMIT_NOFILE`：先把 fd 上限设置够（或在 systemd 配好）
2) `SIGCHLD + waitpid(WNOHANG)`：避免 worker 退出变僵尸
3) 线程模型：关键线程可考虑 affinity（减少迁移）
4) 普通任务：默认 CFS；对后台维护任务可 `nice(+10)`
5) 只有在严格时延需求下才考虑 RT（并配合 mlockall、避免阻塞）

---

# 快速自测（建议你做完再看答案）

1) `nice` 和实时 priority 有什么本质区别？
2) 为什么 `sched_yield()` 不能替代互斥锁/条件变量？
3) 为什么很多守护进程要提高 `RLIMIT_NOFILE`？
4) affinity 一定能提高性能吗？哪些情况下会变差？

