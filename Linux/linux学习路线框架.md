# Linux 学习路线总框架（纲要版）

> 只列出模块与知识点，不展开细节，便于后续逐条深入。

---

## 一、基础入门阶段[[linux基础入门阶段]]
1. Linux 基本概念与发行版选择
2. 图形界面 vs 命令行
3. 终端与 Shell 基础（bash 为主）
4. 基本命令使用习惯与帮助系统（man / --help）

---

## 二、命令行与文件系统[[linux命令行与文件系统]]
1. 路径与目录结构（/、/home、/etc、/var、/usr 等）
2. 常用文件与目录操作命令（ls、cd、cp、mv、rm、mkdir、find 等）
3. 文件查看与搜索（cat、less、head、tail、grep、wc 等）
4. 硬链接、软链接与 inode

---

## 三、权限与用户管理[[linux权限与用户管理详细讲解]]
1. Linux 权限模型（rwx、所有者/用户组/其他）
2. chmod / chown / chgrp 的使用
3. 用户与用户组管理（useradd、userdel、groupadd 等）
4. sudo 与最小权限原则

---

## 四、文本编辑器与开发环境
1. vi/vim 基本使用
2. nano 或其他编辑器（可选）
3. 常见 shell 环境配置文件（.bashrc / .profile 等）
4. 常见开发工具链（gcc、g++、make、gdb、cmake 等）

---

## 五、进程与作业控制
1. 进程概念与 PID
2. 前台/后台作业，作业控制（&、jobs、fg、bg）
3. ps、top/htop、kill 等命令
4. 系统启动与 systemd（service、systemctl 基本概念）

---

## 六、文件系统与存储管理
1. 磁盘与分区基础概念
2. 挂载与卸载（mount、umount、/etc/fstab）
3. df、du 查看磁盘与目录占用
4. 日志文件与日志目录（/var/log）

---

## 七、软件包管理与系统更新
1. 不同发行版的包管理工具：
   - Debian/Ubuntu 系：apt/apt-get
   - CentOS/RHEL 系：yum/dnf
2. 软件安装、升级、卸载
3. 软件源配置与镜像源

---

## 八、Shell 脚本基础
1. Shell 脚本基本语法与执行方式
2. 变量、环境变量与配置文件
3. 条件判断与循环（if、for、while）
4. 管道与重定向（|、>、>>、2> 等）
5. 简单自动化脚本与定时任务（cron）

---

## 九、网络基础与远程管理
1. IP、端口、协议（TCP/UDP）基础
2. 网络配置查看与基础工具（ifconfig/ip、ping、traceroute、netstat/ss 等）
3. ssh 远程登录与密钥登录
4. scp/rsync 进行文件传输

---

## 十、常见服务与应用部署
1. Web 服务（如 nginx/apache 的基础安装与配置）
2. 数据库服务（MySQL/PostgreSQL 基本使用）
3. 简单应用部署流程示例（如部署一个简单的后端服务）
4. systemd 管理自定义服务

---

## 十一、系统监控与性能排查
1. 资源监控（CPU、内存、磁盘、网络）
2. top/htop、free、vmstat、iostat、sar 等常用工具
3. 常见性能问题排查思路

---

## 十二、安全与备份基础
1. 防火墙基础（ufw / firewalld 简单使用）
2. SSH 安全配置（端口、禁用密码登录等）
3. 基本备份策略与备份工具

---

## 十三、进阶方向（按兴趣选）
1. Linux 内核与操作系统原理
   - 进程调度、内存管理、文件系统实现等
2. 系统编程
   - 系统调用、进程间通信、Socket 编程
3. 运维与自动化 / DevOps
   - Ansible、Docker、Kubernetes、CI/CD 等
4. 性能调优与故障排查
   - perf、eBPF、各种 tracing 工具
5. 安全方向
   - 权限提升、日志审计、安全加固、Linux 下攻防实战

---

> 后续可以按模块顺序，一节一节展开，比如：先详细讲“命令行与文件系统”，再讲“权限与用户管理”，循序渐进。

