先记住四个字母规则：

- `l` = list：参数用“可变参数列表”一个个写
    
- `v` = vector：参数用 `argv[]` 数组
    
- `p` = path：按 `PATH` 环境变量搜索可执行文件
    
- `e` = env：显式传入环境变量 `envp[]`
    

下面逐个解释。

---

## 1) `int execl(const char *path, const char *arg0, ..., (char *)NULL);`

**用法**：给定“可执行文件的完整路径/相对路径” `path`，用可变参数列出参数。

- `path`：要执行的程序路径（例如 `"/bin/ls"`）
    
- `arg0`：新程序看到的 `argv[0]`（通常写程序名，如 `"ls"`；也可以故意写别的）
    
- `...`：依次是 `argv[1], argv[2], ...`
    
- 结尾必须用 `(char*)NULL` 终止（告诉函数参数列表结束）
    

示例：

`execl("/bin/ls", "ls", "-l", "/tmp", (char*)NULL);`

特点：

- **不搜索 PATH**，必须给出可执行文件路径。
    
- 环境变量沿用当前进程的环境（等价于用 `environ`）。
    

---

## 2) `int execv(const char *path, char *const argv[]);`

**用法**：和 `execl` 一样不搜索 PATH，但参数用数组传。

- `path`：可执行文件路径
    
- `argv`：以 `NULL` 结尾的字符串数组（`argv[0]` 必须存在）
    

示例：

`char *argv[] = {"ls", "-l", "/tmp", NULL}; execv("/bin/ls", argv);`

适用场景：参数个数不固定、需要动态拼参数时，用 `v` 更舒服。

---

## 3) `int execle(const char *path, const char *arg0, ..., (char *)NULL, char *const envp[]);`

**用法**：在 `execl` 的基础上，**额外显式指定环境变量**。

参数有个“坑点”（顺序固定）：

- 可变参数列表最后仍然要 `(char*)NULL` 结束
    
- **紧接着**再传 `envp[]`
    

示例：

`char *envp[] = {"PATH=/usr/bin:/bin", "LANG=C", NULL}; execle("/usr/bin/env", "env", (char*)NULL, envp);`

特点：

- 不搜索 PATH（依然要求 `path`）
    
- 新程序的环境变量来自你传的 `envp`，而不是继承当前环境
    

---

## 4) `int execve(const char *path, char *const argv[], char *const envp[]);`（最核心）

这是语义“最原始/最完整”的形式：**路径 + argv + envp 都显式给**。  
很多系统里它对应内核提供的核心接口（其他 exec* 往往是 libc 包装成这个）。

- `path`：可执行文件路径（不搜索 PATH）
    
- `argv[]`：参数数组（NULL 结尾）
    
- `envp[]`：环境数组（NULL 结尾）
    

示例：

`char *argv[] = {"sh", "-c", "echo hello", NULL}; char *envp[] = {"PATH=/usr/bin:/bin", NULL}; execve("/bin/sh", argv, envp);`

---

## 5) `int execlp(const char *file, const char *arg0, ..., (char *)NULL);`

**用法**：等价于 `execl` + **按 PATH 搜索**。

- `file`：文件名或路径
    
    - 如果 `file` 里包含 `/`，通常就当作路径用，不走 PATH 搜索
        
    - 如果不包含 `/`，就在 `PATH` 各目录里找
        
- `arg0...`：同 `execl`
    

示例：

`execlp("ls", "ls", "-l", (char*)NULL);  // 搜索 PATH 找到 ls`

环境变量：继承当前环境。

---

## 6) `int execvp(const char *file, char *const argv[]);`

**用法**：等价于 `execv` + **按 PATH 搜索**。

- `file`：同 `execlp`
    
- `argv[]`：同 `execv`
    

示例（最常用）：

`char *argv[] = {"ls", "-l", NULL}; execvp("ls", argv);  // 搜索 PATH`

---

## 通用注意事项（这几种都一样）

- **成功不返回**；失败返回 `-1`，常见 `errno`：`ENOENT`（找不到）、`EACCES`（没权限）、`ENOEXEC`（不是可执行格式）、`ENOMEM` 等。
    
- `argv`/`envp` 都必须以 `NULL` 结尾。
    
- 打开文件描述符默认会继承到新程序，除非设置了 `FD_CLOEXEC`（close-on-exec）。
    
- 典型写法：`fork()` 后子进程 `exec...`；若 exec 失败要 `perror()` 然后 `_exit(127)`。