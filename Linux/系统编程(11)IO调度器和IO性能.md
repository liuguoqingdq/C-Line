# Linux IO 调度器与 IO 性能：从磁盘寻址到调优实战

这份讲解按“**磁盘为什么慢 → 内核怎么排队 → 怎么选调度器 → 怎么把读写做快**”的顺序来。
##### 在现代系统中，磁盘和系统其他组件的性能差距很大，而且还在增大。磁盘性能最糟糕的部分在于把读写头（即磁头）从磁盘的一个位置移动到另一个位置，该操作称为“查找定位（seek）”。在实际应用中，很多操作是以处理器周期（大概是1/3纳秒）来衡量，而单次磁盘查找定位平均需要8毫秒以上——这个值虽然不大，但是它却是CPU周期的2500万倍。 

 #### 由于磁盘驱动和系统其他组件在性能上的巨大差异，如果每次有I/O请求时，都按序把这些I/O请求发送给磁盘，就显得过于“残忍”，效率也会非常低下。因此，现代操作系统内核实现了I/O调度器（I/O Scheduler），通过操纵I/O请求的服务顺序以及服务时间点，最大程度减少磁盘寻址次数和移动距离。I/O调度器尽力将硬盘访问的性能损失控制在最小。
---

## 1. 磁盘寻址与性能瓶颈（先理解硬件）

### 1.1 HDD（机械硬盘）为什么怕随机 I/O
HDD 访问一个块，典型要付出三类成本：
- **寻道时间**：磁头移动到目标磁道（ms 级）
- **旋转延迟**：等盘片转到目标扇区（ms 级）
- **传输时间**：真正把数据读出来（相对更小）

结论：
- **顺序读写**（连续 LBA）可以把寻道/旋转摊薄 → 吞吐高
- **随机读写**（跳跃 LBA）会让寻道/旋转主导 → IOPS 很低

### 1.2 SSD / NVMe（固态）“没有寻道”，但仍有性能结构
SSD 不需要机械寻道，所以随机 IOPS 远高于 HDD，但它仍受：
- **并行度**：通道/Die/队列深度（QD）决定能否跑满
- **写放大与 GC**：小写、随机写、频繁 fsync 会触发内部搬运
- **对齐与块大小**：4K 对齐/写入大小会影响放大

NVMe 的特点：
- 多队列（多提交队列/完成队列）+ 高 QD → 更适合并发 I/O

### 1.3 “页缓存”和“磁盘缓存”不是一回事
- **page cache**：Linux 内核在 RAM 里缓存文件数据页（4K）
- **CPU cache**：L1/L2/L3 缓存的是“内存访问”

很多“磁盘很慢”的感觉，其实是：
- 你在做 **cache miss**（页缓存未命中）→ 真的触发磁盘 I/O

---

## 2. Linux I/O 路径（理解调度器在什么位置）

简化路径（文件 I/O）：

1) 应用 `read/write`
2) VFS + 文件系统
3) **page cache**（可能直接命中返回）
4) 未命中 → 生成块 I/O 请求（bio/request）
5) **blk-mq 多队列层**
6) **I/O 调度器（scheduler）**（可选/可配置）
7) 设备驱动 → 磁盘/NVMe

关键点：
- 你如果一直命中 page cache，调度器几乎“出不了场”。
- 调度器主要影响 **真实落到块设备的请求如何排队与合并**。

---

## 3. I/O 调度器的功能（它到底在做什么）

调度器不是“加速魔法”，它做的是**在吞吐、公平、延迟之间做权衡**。

### 3.1 合并（merge）：把相邻请求拼成大请求
- 例如：两个连续的 4K 读 → 合成一个 8K 读
- 好处：减少请求数量、降低寻道/命令开销

### 3.2 排序（sort）：把请求按 LBA 排队
- 对 HDD 很重要：尽量按磁盘顺序读写，减少寻道
- 对 SSD/NVMe 意义小一些，但依然可能减少设备端开销

### 3.3 防止饥饿（starvation）与延迟控制
- 纯追求吞吐可能让某些请求一直排不到
- “deadline”类算法会给请求一个最后期限，避免读/写饿死

### 3.4 公平性（fairness）
- 多进程竞争磁盘时，保证“谁都能获得一定份额”
- 桌面交互场景经常更关心 **低延迟 + 公平**

---

## 4. 常见 Linux I/O 调度器（现代 blk-mq 视角）

> 你的机器可用哪些调度器，取决于内核版本/编译选项/设备类型。

现代常见（blk-mq）：
- **none**：不做软件层调度（把请求更直接交给设备/驱动）
- **mq-deadline**：deadline 思想的多队列版本，偏“稳健/通用”
- **kyber**：更强调延迟/吞吐的折中控制（通过令牌等机制）
- **bfq**：强调“交互性与公平”（桌面/多媒体体验常更好）

老式（单队列时代）你可能会见到：noop、deadline、cfq（较老）。

---

## 5. 如何查看 / 选择 / 配置 I/O 调度器

### 5.1 查看当前设备调度器
```bash
cat /sys/block/sda/queue/scheduler
cat /sys/block/nvme0n1/queue/scheduler
```
输出类似：
```
[mq-deadline] none kyber bfq
```
方括号表示当前正在用的。

### 5.2 临时切换（重启后失效）
```bash
echo mq-deadline | sudo tee /sys/block/sda/queue/scheduler
echo none        | sudo tee /sys/block/nvme0n1/queue/scheduler
```

### 5.3 永久配置（通过内核启动参数或 udev 规则）
- 内核参数示例（不同发行版写法略有差异）：
  - `elevator=mq-deadline`（对默认/某些设备生效）
- 更通用做法：写 udev 规则对特定设备设置（生产环境更可控）。

> 实务建议：先用“临时切换 + fio 压测”确认收益，再做永久化。

---

## 6. “改进读请求”的典型手段（比换调度器更常见）

### 6.1 顺序读取：靠大块 + 预读（readahead）
- 让请求变大、变连续，比调度器更有效。

**应用层**：
- 用更大的 `read()` buffer（例如 256KB、1MB）
- 合并小文件（打包 packfile/asset bundle）

**内核层**：
- `posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL)`：告诉内核你是顺序读
- `readahead(fd, off, len)`：更“直接”的 Linux 预读接口（只是把页读进 page cache）
- `madvise(mapped, len, MADV_SEQUENTIAL / MADV_WILLNEED)`：对 mmap 映射区提示

### 6.2 随机读取：减少无用预读 + 做热点缓存
- `posix_fadvise(..., POSIX_FADV_RANDOM)` 或 `madvise(..., MADV_RANDOM)`
- 业务层做索引/热点缓存（LRU、块缓存）
- 尽量把随机读聚合为“较大的块读”（例如按 64KB/256KB 读块）

### 6.3 page cache 与 O_DIRECT（直接 I/O）的取舍
- **走 page cache**：适合“重复读、热点多、顺序扫”等
- **O_DIRECT**：绕过页缓存，减少双缓存/内存压力，适合数据库类自管缓存
  - 代价：对齐限制更严格、编程复杂、对小 I/O 不友好

---

## 7. 选择调度器的经验规则（按设备/场景）

### 7.1 HDD（机械盘）
- 通常：**mq-deadline**（稳健，减少读延迟，防止饥饿）
- 桌面交互：可试 **bfq**（交互体验常更好）

### 7.2 SATA SSD
- 通常：**mq-deadline** 或 **none**
- 如果你观察到写抖动/延迟尖刺，mq-deadline 往往更“稳”。

### 7.3 NVMe SSD
- 常见：**none**（让 NVMe 控制器自己做队列调度，减少软件层干预）
- 如果你的负载对尾延迟敏感，可试 **kyber** 或 **mq-deadline**，以“延迟更稳”为目标。

### 7.4 虚拟机 / 云盘
- 先搞清楚底层是什么（网络块设备、共享存储）。
- 有些环境里调度器影响很小，瓶颈在网络/后端。

---

## 8. 关键配置项（常用调优旋钮）

> 先测再改，避免“调参玄学”。

### 8.1 readahead 大小
```bash
cat /sys/block/sda/queue/read_ahead_kb
sudo blockdev --setra 4096 /dev/sda   # 4096 sectors（注意单位）
```
- 顺序扫：可适当增大
- 随机读：过大反而污染缓存

### 8.2 队列深度 / 请求数
```bash
cat /sys/block/sda/queue/nr_requests
cat /sys/block/nvme0n1/queue/nr_requests
```
- 并发 I/O 高的场景，合理的队列深度很重要

### 8.3 脏页回写（影响写入抖动/延迟）
```bash
sysctl vm.dirty_ratio
sysctl vm.dirty_background_ratio
sysctl vm.dirty_expire_centisecs
sysctl vm.dirty_writeback_centisecs
```
- 写入突刺、fsync 频繁、后台回写策略会显著影响尾延迟。

### 8.4 文件系统挂载选项（按场景）
- `noatime`：减少读引起的元数据写
- 日志参数（ext4/xfs）会影响写延迟与一致性
- `discard`/`fstrim`：SSD 回收（通常用定期 `fstrim` 更合适）

---

## 9. 优化 I/O 性能的系统化方法（建议你照这个流程做）

### 9.1 先识别你的负载类型
- 顺序读/顺序写/随机读/随机写？
- I/O 大小（4K 还是 1MB）？
- 同步写（fsync）频率？
- 并发度（线程数/QD）？

### 9.2 用工具量化
- `iostat -x 1`：util、await、svctm、队列长度等
- `fio`：构造顺序/随机、不同块大小/QD 的对比压测
- `pidstat -d 1`：哪个进程在打磁盘
- 深入：`blktrace/bpftrace/perf`（看排队、合并、延迟分布）

### 9.3 再做改动（从收益最大、风险最小开始）
1) 应用层：合并小 I/O、批量、异步、减少 fsync
2) 预读策略：SEQUENTIAL/RANDOM、readahead/madvise
3) 调度器：none / mq-deadline / kyber / bfq（用 fio 证实）
4) 系统参数：dirty_*、队列深度、文件系统选项

---

## 10. 给“游戏场景初始化/资源加载”的落地建议（你之前问过）

- **最大收益通常来自：资源打包 + 顺序大块读 + 后台预取**
- 如果是 packfile 顺序读：
  - 大 buffer read（≥256KB/1MB）
  - `posix_fadvise(SEQUENTIAL)` + 合理 readahead
  - 预取下一段：可用 `readahead` 或后台线程提前触发读取
- 如果是大量小文件：
  - 优先做 bundle/pack
  - 否则会被元数据查找 + 小 I/O 放大拖死，调度器救不了

---

如果你愿意，把你的目标环境告诉我（HDD/SATA SSD/NVMe、文件系统 ext4/xfs、负载是顺序扫还是随机点查、平均 I/O 大小、是否频繁 fsync），我可以给你一套“**调度器 + readahead + dirty_* + fio 验证脚本**”的具体参数组合。

