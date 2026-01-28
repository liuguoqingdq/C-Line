# 三、权限与用户管理：详细讲解

## 1. Linux 权限模型（rwx、所有者 / 用户组 / 其他）

### 1.1 基本概念：谁对文件有权限？

Linux 中，每个文件 / 目录都绑定三类主体：

1. **所有者（user，简写 u）**：通常是创建文件的用户。
2. **所属用户组（group，简写 g）**：文件归属的组。
3. **其他人（others，简写 o）**：系统里除前两者以外的所有用户。

对应地，每一类主体对该文件都有一组 **rwx 权限位**：

- `r`（read）：读权限
- `w`（write）：写权限
- `x`（execute）：执行权限

三类主体各自一组三位，一共 9 位，再加上文件类型位，总共 10 个字符。

### 1.2 `ls -l` 中的权限字段如何看？

示例：

```bash
ls -l test.sh
-rwxr-x--- 1 alice dev  1234 Jan  1 10:00 test.sh
```

字段拆解：

- 第 1 位：`-` 表示普通文件；
  - `d` 表示目录（directory）；
  - `l` 表示符号链接（link）；
  - 还有其他：`c` 字符设备、`b` 块设备等。

- 后 9 位分成三组：
  - `rwx`：**所有者**的权限（user）
  - `r-x`：**用户组**的权限（group）
  - `---`：**其他人**的权限（others）

对应关系：

- 所有者（alice）：`rwx` → 可读、可写、可执行
- 组成员（dev 这个组的其他用户）：`r-x` → 可读、可执行，不可写
- 其他所有用户：`---` → 完全无权访问

### 1.3 文件 vs 目录的权限含义（非常重要）

对**普通文件**：
- `r`：允许读取内容；
- `w`：允许修改内容、截断、删除（前提是目录权限允许）；
- `x`：允许作为“可执行文件”运行（脚本 / 二进制）。

对**目录**：
- `r`：可以列出目录内容（例如 `ls`）；
- `w`：可以在目录中创建、删除、重命名文件 / 子目录；
- `x`：可以“进入目录”（`cd`）以及访问其中条目的 inode（没有 `x` 即使知道文件名也无法访问）。

典型组合例子：

- 目录 `r-x`：可以进入并列出文件，但不能新建 / 删除；
- 目录 `--x`：可以进入目录并访问那些你知道名字的文件，但不能列出目录内容（"偷偷知道名字就能进"）；
- 目录 `r--`：几乎没用，因为不能 `cd` 进去（缺 `x`）。

### 1.4 数字权限表示法：rwx → 7 6 5...

每一位权限可以用数字 4 / 2 / 1 表示：

- `r` = 4
- `w` = 2
- `x` = 1

一组 `rwx` 的总值 = 可用权限之和：

- `rwx` = 4+2+1 = 7
- `rw-` = 4+2+0 = 6
- `r-x` = 4+0+1 = 5
- `r--` = 4+0+0 = 4
- `-wx` = 0+2+1 = 3
- `--x` = 0+0+1 = 1
- `---` = 0

于是 9 位权限可以用一个三位八进制数表示：

- `rwxr-x---` = 所有者 7、组 5、其他 0 → `750`
- `rw-r--r--` = `644`
- `rwxr-xr-x` = `755`

我们在后面使用 `chmod 755 file` 等数字形式就是这个含义。

---

## 2. `chmod` / `chown` / `chgrp` 的使用

### 2.1 `chmod`：修改权限

`chmod` 支持两种写法：数字式 和 符号式。

#### 2.1.1 数字式（最常用）

```bash
chmod 644 file.txt    # 所有者 rw-，组 r--，其他 r--
chmod 755 script.sh   # 所有者 rwx，组 r-x，其他 r-x
chmod 700 secret.txt  # 只有自己可以访问
```

典型场景：

- Web 目录文件：`644`（所有者可写，其他只读）；
- 可执行脚本：`755`（大家可以执行，但只有作者能改）；
- SSH 私钥：`600` 或 `400`。

#### 2.1.2 符号式（更直观，可局部修改）

语法：

```bash
chmod [ugoa][+-=][rwx] file
```

- 主体：
  - `u`：所有者
  - `g`：组
  - `o`：其他人
  - `a`：所有（u+g+o）
- 操作符：
  - `+`：增加权限
  - `-`：移除权限
  - `=`：直接设置为指定权限

示例：

```bash
chmod u+x script.sh     # 给所有者加执行权限
chmod go-rw file.txt    # 移除组和其他人的读写权限
chmod a+r file.txt      # 所有人增加读权限
chmod u=rw,go= file.txt # 所有者 rw，组和其他无权限
```

#### 2.1.3 递归修改目录

```bash
chmod -R 755 /var/www/html
```

`-R` 表示递归，将权限应用到目录及其所有子目录和文件。

> 注意：递归改权限风险很大，尤其是从 `/` 之类的路径开始时，务必三思再回车。

### 2.2 `chown`：修改所有者

语法：

```bash
chown [新所有者][:新组] 文件
```

例子：

```bash
chown alice file.txt         # 改所有者为 alice，组不变
chown alice:dev file.txt     # 所有者 alice，所属组 dev
chown :dev file.txt          # 只改组为 dev

chown -R www-data:www /var/www  # 递归修改 Web 目录所属用户和组
```

> 普通用户通常只能把自己拥有的文件“让”给 root 或由 root 来修改。一般情况下，`chown` 需要 root 权限（或 sudo）。

### 2.3 `chgrp`：单独修改组

如果只想改组，也可以用 `chgrp`：

```bash
chgrp dev file.txt
chgrp -R dev /project
```

等价于 `chown :dev file.txt`。

### 2.4 `umask`：默认权限的屏蔽掩码

当你创建一个文件时，系统不会默认给它 `777` 权限，而是根据 `umask` 进行“减法”。

- 一般文件的“最大默认”权限：`666`（rw-rw-rw-），因为对普通文件默认不给执行位；
- 目录的“最大默认”权限：`777`（rwxrwxrwx）。

`umask` 是一个掩码，表示需要从最大权限中移除的位：

- 比如 `umask 022`：
  - 文件默认：`666 - 022 = 644`（rw-r--r--）；
  - 目录默认：`777 - 022 = 755`（rwxr-xr-x）。
*`做非运算`*

查看当前 umask：

```bash
umask
```

设置当前 shell 的 umask：

```bash
umask 027
```

全局或用户级的 umask 通常在 `/etc/profile`、`~/.bashrc` 等配置文件中设置。

---

## 3. 用户与用户组管理（useradd、userdel、groupadd 等）

### 3.1 用户 / 组 的基本概念

Linux 是多用户系统：

- 每个用户有一个唯一的 UID（用户 ID）；
- 每个组有一个唯一的 GID（组 ID）；
- 用户可以属于一个“主组”和多个“附加组”；
- 文件的权限是基于 **UID / GID** 来判断的。

查看当前用户和所属组：

```bash
whoami      # 当前用户名
id          # UID、GID、附加组信息
id alice    # 查看 alice 的 UID/GID 以及其所在组
```

核心账号文件：

- `/etc/passwd`：记录用户名、UID、GID、home 目录、登录 shell 等；
- `/etc/shadow`：存放密码哈希（只有 root 能读）；
- `/etc/group`：记录用户组信息。

### 3.2 创建 / 删除用户

**注意**：下列命令通常需要 root 或 sudo 权限。

#### 3.2.1 `useradd` 创建用户

比较底层、偏原始，很多发行版提供 `adduser` 这个更友好的封装。

示例：

```bash
# 简单创建用户（可能不会自动创建 home）
useradd alice

# 推荐方式：指定创建 home 目录、默认 shell
useradd -m -s /bin/bash alice

# 指定主组和附加组
useradd -m -s /bin/bash -g dev -G sudo,docker alice
```

常见选项：
- `-m`：自动创建 home 目录（例如 `/home/alice`）；
- `-s SHELL`：登录 shell，比如 `/bin/bash`；
- `-g`：主组；
- `-G`：附加组列表，以逗号分隔。

创建完用户后设置密码：

```bash
passwd alice
```

#### 3.2.2 `adduser`（更友好的脚本）

在 Debian/Ubuntu 系中通常推荐用：

```bash
adduser alice
```

它是 `useradd` 的一层封装，会引导你：
- 输入密码；
- 自动创建家目录；
- 填写一些备注信息等。

#### 3.2.3 删除用户：`userdel`

```bash
userdel alice          # 删除账号，保留 home 目录
userdel -r alice       # 删除账号并连带 home 目录
```

`-r` 会清除用户的 home 和邮件队列，操作前一定确认清楚。

### 3.3 创建 / 管理用户组

#### 3.3.1 创建组：`groupadd`

```bash
groupadd dev
```

这会在 `/etc/group` 中增加一条 dev 组记录。

#### 3.3.2 把用户加入组：`usermod`

```bash
# 把 alice 加入 dev 组（附加组）
usermod -aG dev alice
```

注意：
- `-G`：设置附加组；
- `-a`（append）：追加到原有组列表，否则会替换原有附加组！

### 3.4 查看 / 修改信息

- 查看用户信息：`id alice`，`grep alice /etc/passwd`
- 修改登录 shell：

```bash
chsh alice             # 交互式修改
chsh -s /bin/zsh alice # 直接指定
```

在实践中，直接手改 `/etc/passwd` 等文件是有风险的，一般推荐用命令接口（`usermod` 等）。

---

## 4. sudo 与最小权限原则

### 4.1 root 用户与普通用户

- `root`：UID 为 0 的超级用户，对系统有完全控制权限；
- 普通用户：受权限系统限制，只能在自己的 home、自己拥有的文件等范围内操作。

长期以 root 身份日常操作，很容易因为一个小失误（例如 `rm -rf /`）造成灾难；因此现代 Linux 都推荐：

> **日常使用普通用户，只有需要管理操作时临时提升权限。**

### 4.2 `sudo` 的基本用法

`sudo` 允许普通用户以 **另一个用户（默认 root）** 的身份执行单条命令：

```bash
sudo command args...

# 例如：
sudo apt update
sudo systemctl restart nginx
sudo chown -R www-data:www /var/www
```

第一次使用 `sudo` 时，会要求输入自己的密码（而不是 root 的密码），验证通过后会在一段时间内不再提示。

查看自己是否有 sudo 权限：

```bash
sudo -l
```

### 4.3 sudoers 配置与 visudo

哪些用户可以用 `sudo`、可以执行哪些命令，是由 **sudoers 配置** 决定的：

- 主配置文件：`/etc/sudoers`
- 推荐做法：不要直接编辑 `/etc/sudoers`，而是：
  - 用 `visudo` 命令安全编辑（会做语法检查）；
  - 或在 `/etc/sudoers.d/` 下新建片段文件。

举个简单 sudoers 片段例子（仅理解）：

```text
# 允许 admin 组的成员执行任何命令（典型设置）
%admin ALL=(ALL) ALL

# 允许 deploy 用户免密码重启某个服务
deploy ALL=(root) NOPASSWD: /bin/systemctl restart myapp
```

解释：
- `%admin`：表示 admin 组；
- `ALL=(ALL) ALL`：在所有主机上，以任意身份执行任何命令；
- `NOPASSWD:`：表示执行该条目的命令时不再提示密码。

### 4.4 最小权限原则（Principle of Least Privilege）

安全和运维里的一个核心理念：

> **只授予完成工作所必需的最小权限。**

在 Linux 用户管理场景中体现为：

1. 日常开发、使用软件，用普通用户；
2. 只有在需要安装软件、改系统配置、管理服务时才用 `sudo`；
3. 即使使用 sudo，也尽量限制命令范围：
   - 比如只允许某个用户 `sudo systemctl restart myapp`，而不是任意命令；
4. 不随意给别人 root 密码，不让普通账号直接 `su` 到 root。

一些不太推荐的习惯：

- 开机后立刻 `sudo su -`，然后整天在 root 环境里干活；
- 给 sudo 配置 `NOPASSWD: ALL` 给一大堆用户；
- 大范围用 `chmod -R 777` 简单粗暴放开权限，带来安全隐患。

### 4.5 实战小建议

1. 新建用户后，把它加入一个“运维组”（比如 `sudo` 或 `wheel` / `admin`），通过组控制 sudo 权限；
2. 存放代码的目录尽量限定为某个组可写，其他只读，结合 `chown` + `chmod` 控制；
3. 每次在生产机器上敲 `sudo` 或执行破坏性命令前（rm/chown/chmod -R），确认路径无误；
4. 经常用 `id` 看自己当前身份和组，避免在错误账号下操作。

---

## 小结

这一节的核心要点：

1. **权限位模型**：三类主体（所有者 / 组 / 其他） × 三种权限（r/w/x），理解文件和目录语义的区别。
2. **权限修改工具**：掌握 `chmod`（数字和符号两种方式）、`chown`、`chgrp`、`umask` 的基本用法，能根据场景合理设置文件和目录权限。
3. **用户 / 组管理**：理解 UID/GID 的作用，能用 `useradd` / `adduser` / `userdel` / `groupadd` / `usermod` 等命令管理账号和用户组。
4. **sudo 与安全原则**：明白 root 的危险性，学会通过 sudo 临时提升权限，并遵守“最小权限”原则，避免随手开过大权限导致安全 / 运维事故。

理解并熟练掌握这些内容，你就已经具备了在多用户 Linux 环境中安全、规范地运维和开发的基础。

