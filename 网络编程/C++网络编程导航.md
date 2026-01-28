## 模块 0：前置基础（C++ + Linux 环境）

**目标：**能在 Linux 下用 g++ 写、编、调 C++ 程序。

- C++ 语言
    
    - 指针/引用、结构体、class/构造析构
        
    - RAII + 智能指针（`std::unique_ptr` / `std::shared_ptr`）
        
    - STL：`std::string / vector / map`，范围 for、`auto`、`nullptr`
        
- Linux 基础
    
    - 目录/权限/进程基本命令
        
    - g++：`g++ -g -Wall -std=c++17 main.cpp -o app`
        
    - Makefile 基本使用（你已经在学）
        

> 这一块主要靠你现有 C++ 经验和 Linux 习惯，不对应 PDF 某一页。

---

## 模块 1：网络 & TCP/IP 基础

**目标：**把“协议/分层/报文/握手/挥手”这些概念吃透，为后面的 API 打地基。

**对应 PDF：**第 1 部分“网络基础”：协议概念、分层模型、以太网帧、IP 报文、TCP/UDP 报文、三次握手、四次挥手、TCP 状态迁移、SYN 攻击等。

2.1计算机网络

- 协议 & 分层
    
    - 什么是“协议”“报文格式”“分层模型”（应用/传输/网络/链路）
        
- 报文格式
    
    - 以太网帧、IP 首部字段（TTL、源/目的 IP）
        
    - TCP 段、UDP 报文：端口号、序列号、确认号、窗口
        
- TCP 行为
    
    - 三次握手、四次挥手
        
    - 状态迁移图：`SYN_SENT / ESTABLISHED / FIN_WAIT_1 / TIME_WAIT ...`
        
    - 为什么有 TIME_WAIT、SYN 攻击的原理
        

**建议练习：**  
自己画一张 _从浏览器访问服务器_ 的时序图，把“ARP → 三次握手 → 收发数据 → 四次挥手”过程标清楚。

---

## 模块 2：网络编程基础概念 & 字节序[[IP + 端口、字节序与地址转换]]

**目标：**理解“IP + 端口 = 唯一进程”的抽象，搞清楚字节序和地址转换函数。

**对应 PDF：**“二、网络编程”里的基础与字节序部分，`htonl/htons/ntohl/ntohs` 函数。

2.1计算机网络

- IP + Port 表示唯一进程
    
- 主机字节序 vs 网络字节序：小端 vs 大端
    
- 四个转换函数：
    
    - `htonl/htons`：host→network
        
    - `ntohl/ntohs`：network→host
        
- 练习：写个小程序打印一个 `0x12345678` 在本机内存和网络字节序下的二进制表示。
    

---

## 模块 3：地址结构 & 地址转换[[地址结构与地址转换]]

**目标：**搞懂各种 `sockaddr*` 结构体的意义，熟练用 `inet_pton/inet_ntop` 等。

**对应 PDF：**“常用结构体”部分，`sockaddr/sockaddr_in`、`in_addr`、`inet_pton/inet_ntop/inet_aton/inet_ntoa`。

2.1计算机网络【找it资源dbbp.net】【微 信6487…

- 通用地址结构 `sockaddr`
    
- IPv4：`sockaddr_in`、`in_addr` 字段及含义
    
- 常用地址转换函数：
    
    - `inet_pton / inet_ntop`（推荐）
        
    - `inet_aton / inet_ntoa / inet_addr`（知道即可）
        
- 本机地址查询：`getsockname` / `getpeername`（了解）
    

**建议练习：**  
写一个 “字符串 IP + 端口 → sockaddr_in → 再打印回字符串”的小工具，确认你真的搞懂了。

---

## 模块 4：基础 Socket API & 阻塞式 TCP/UDP[[基础socket与阻塞式tcp_udp]]

**目标：**能写出最基本的 **阻塞式 TCP/UDP 客户端 + 服务器**。

**对应 PDF：**第 4 节“网络编程相关函数”及“服务器端源码/客户端源码”：`socket/bind/listen/accept/connect/close/setsockopt`、简单 echo server/client 示例。

2.1计算机网络

- 创建 & 关闭
    
    - `socket()`、`close()`、`shutdown()`
        
- 服务端流程（TCP）：
    
    - `socket → setsockopt(SO_REUSEADDR/SO_REUSEPORT) → bind → listen → accept → read/write`
        
- 客户端流程（TCP）：
    
    - `socket → connect → read/write`
        
- UDP 基本流程（了解）：`sendto/recvfrom`
    
- `INADDR_ANY`、端口复用（解决 TIME_WAIT 占用端口）
    

**建议练习：**

1. 用 **C 风格 API 写 echo server/client**（可以先照 PDF 代码敲一遍）。
    
2. 再用 C++ 把 socket 封成一个 RAII 类（构造里 `socket`，析构里 `close`）。
    

---

## 模块 5：可靠 I/O 与粘包问题

**目标：**理解 `read` 返回值语义、粘包/拆包问题，能封装 `readn/writen`。

**对应 PDF：**“read 返回值说明”与 `readn/writen` 实现。

2.1计算机网络【找it资源dbbp.net】【微 信6487…

- `read` 可能的返回值：
    
    - > 0：实际读到的字节数
        
    - ==0：对端关闭（EOF）
        
    - <0：错误，区分 `EINTR` / `EAGAIN` / 其他
        
- `readn/writen` 封装思想：循环读/写直到读完指定长度或出错
    
- TCP 粘包/拆包：为什么一次 `read` 不一定得到一条“消息”
    

**建议练习：**  
在 echo server 上加一层“**长度 + 数据**”的协议，用 `readn/writen` 来保证一次收发一条完整消息。

---

## 模块 6：I/O 多路复用（select / poll / epoll）[[C++网络编程select详解]]

**目标：**理解多路复用的思想，能写出基于 select/poll/epoll 的多客户端服务器。

**对应 PDF：**“三、IO多路复用”章节，包括 select 概念与接口、select 版本服务器 & 客户端、poll 接口与代码、epoll 接口、LT/ET 图解和 epoll 服务器代码。

2.1计算机网络【找it资源dbbp.net】【微 信6487…

1. **I/O 模型 & 多路复用思想**
    
    - 为什么“每连接一个进程/线程”不适合大规模并发
        
    - 内核帮你监视多个 fd，应用只处理“被就绪的那个”
        
2. **select**
    
    - `fd_set` 与 `FD_SET/FD_CLR/FD_ISSET/FD_ZERO`
        
    - `select(nfds, &readset, ...)` 的语义与缺点：1024 上限、集合重置、线性扫描
        
3. **poll**
    
    - `struct pollfd { fd, events, revents }`
        
    - 优点：打破 1024 限制，监听集与返回集分离
        
    - 缺点：仍需遍历整个数组
        
4. **epoll**
    
    - `epoll_create / epoll_ctl / epoll_wait`
        
    - 就绪队列 + O(1) 复杂度
        
    - LT/ET 模式区别：ET 要配合非阻塞 + 循环读到 `EAGAIN`
        
    - 对照 PDF 的 epoll 服务端代码，自己在本地跑一遍
        

**建议练习：**  
按顺序实现三个版本的“多客户端 echo 聊天室”：

- select 版 → poll 版 → epoll(LT) 版，比较性能和代码复杂度。
    

---

## 模块 7：C++ 并发模型与高并发服务器架构

**目标：**从 “会写 epoll 服务端” 进化为 “能设计一个靠谱的高并发网络服务”。

（这一块 PDF 只是打基础，你需要在它之上做 C++ 设计延伸）

- 并发模型
    
    - 单 reactor + 多线程（主线程 epoll，worker 线程处理业务）
        
    - 连接对象/会话对象（Session）的抽象
        
- C++ 线程 & 线程池
    
    - `std::thread / std::mutex / std::condition_variable`
        
    - 任务队列 + 多工作线程
        
- 典型架构：
    
    - 主线程 `epoll_wait` → 将“可读事件”封一个任务扔进线程池 → worker 解析协议 + 业务处理 → 写回 socket
        

**建议练习：**  
给你的 epoll 服务器加一个简单线程池，把业务处理移到后台线程中。

---

## 模块 8：应用层协议 & 简易 HTTP 服务器

**目标：**在已有 socket & epoll 能力上，实现一个简易 HTTP 服务器。

- HTTP 基础
    
    - 请求行：`GET /index.html HTTP/1.1`
        
    - 请求头/响应头、状态码（200/404/500）
        
    - 短连接 vs keep-alive
        
- 实现要点
    
    - 行缓冲读取 + 解析请求行/头
        
    - 根据 URL 映射文件，使用 `sendfile` 或 `read + write` 发送
        
    - 简单日志输出（IP、URL、响应码、耗时）
        

**建议练习：**  
实现：

- 支持静态文件访问
    
- 支持并发连接（基于 epoll）
    
- 通过浏览器访问验证
    

---

## 模块 9：调试 / 性能 / 安全

**目标：**不仅“能跑”，而且“跑得稳、看得清、扛得住”。

- 调试 & 观测
    
    - `strace` 看系统调用
        
    - `tcpdump` / Wireshark 抓包，看三次握手/HTTP 报文
        
    - 日志系统：按连接 / 请求打印关键信息
        
- 性能
    
    - `ulimit -n` 与 fd 上限、机器 file-max
        
    - 零拷贝：`sendfile`（可应用在文件服务器里）
        
- 安全
    
    - 防止简单 DoS：连接数限制、超时关闭、输入长度限制
        
    - 简单认证/鉴权机制（了解）
        

---

## 模块 10：综合项目路线（按难度递进）

1. **多客户端聊天室（epoll + 线程池）**
    
2. **简易 HTTP 服务器（静态资源 + 日志）**
    
3. **端口转发/反向代理（前端接入 + 后端转发）**
    
4. （进阶）在上述项目中进一步加入：配置文件解析、优雅退出、统计接口等。