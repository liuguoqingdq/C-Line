- **Linux不是随意进行进程调度。相反，它给所有进程分配了一个“优先级”，影响它们的运行时间：回想一下，Linux通过进程“优先级（nice value）”来调整分配给该进程的处理器比例。先前，UNIX把这个优先级称为“nice values”，因为它背后的思想是要对其他进程“友好（nice）”，降低该进程的优先级，支持其他进程消耗更多的处理器时间。


 - **合法的nice value范围是(-20,19]，默认值为0。稍让人有些困惑的是：nice value越低，优先级越高，时间片越长；相反，nice value越高，优先级越低，时间片越短。因此，增加进程的nice value意味着该进程对系统更“友好（nice）”。数值上的反向对应很容易让人混淆。当一个进程有“优先级高”时，是指比起优先级低的进程，该进程运行时间更长，但是该进程的nice value值更低。

## 一、nice()系统调用
linux提供了获取和设置优先级的系统调用
**函数签名：
```C
#include <unistd.h>
int nice(int inc);
```

nice()调用成功时，会在现有的nice value上增加inc，并返回更新后的值。只有拥有CAP_SYS_NICE权限的进程（实际上即进程所有者为root）才可以使用负值inc，减少nice value，从而提升该进程的优先级。因此，非root用户的进程只能降低优先级（增加nice value）。 

 出错时，nice()返回-1。但是，由于nice()调用会返回更新后的值，-1也可能是成功时的返回值。因此，为了判断调用是否成功，在调用前应该先把errno值置为0，然后，在调用后再检查errno值。举个例子：
```C
int ret;
errno=0;
ret = nice(10);
if(ret == -1 && errno!=0){
	perror("nice()");
	....//错误处理
}
else{
	//成功
}
```
对于nice()，Linux只会返回一种错误码：EPERM，表示调用的进程试图提高其优先级（传递int值为负数），但没有CAP_SYS_NICE权限。当nice value超出指定范围时，其他系统还会返回 EINVAL，但Linux不会。相反，Linux会把非法值通过四舍五入方式获取对应值，或者当超出指定范围值时，就设置成下限值。 

 获得当前优先级的一种简单方式是给nice()函数传递参数0:
 ```C
 ret = nice(0);
 ```
 **当要设置绝对优先级而不是相对增量时：
 ```C
 int ret = nice(0);
 errno=0;
 int val = 10-ret;
 ret = nice(val);
 if(ret == -1 && errno!=0){
 perror("nice(val)");
 }
 ```

## 二、getpriority()与setpriority()

函数签名：
```C
#include <sys/time.h>
#include <sys/resource.h>

int getpriority(int which, id_t who);
int setpriority(int which, id_t who, int prio);
```
#### `which` 取值（你要操作“谁”的 nice）

- `PRIO_PROCESS`：按进程（PID）
    
- `PRIO_PGRP`：按进程组（PGID）
    
- `PRIO_USER`：按用户（UID）
    

### `who` 怎么填

- 对 `PRIO_PROCESS`：`who` 是 PID；`who==0` 表示当前进程
    
- 对 `PRIO_PGRP`：`who` 是 PGID；`who==0` 表示当前进程组
    
- 对 `PRIO_USER`：`who` 是 UID；`who==0` 表示当前用户
    

### `prio` 是什么

`setpriority()` 里的 `prio` 就是 **nice 值**，通常范围 **-20..19**（越小越“抢 CPU”，越大越“让 CPU”）。

### 3) 返回值与最容易踩的坑

### `setpriority()`

- 成功：返回 0
    
- 失败：返回 -1 并设置 `errno`（常见 `EPERM/ESRCH/EINVAL`）
    

### `getpriority()`（坑点在这里）

- 它返回的是 nice 值（或与 nice 对应的值），但**返回 -1 也可能是合法结果**（例如 nice 就是 -1）
    
- 所以不能只用 `== -1` 判断失败，正确写法是：

```C
int ret;
errno=0;
ret = setpriority(PRIO_PROCESS,0,10);
if(ret == -1 && errno!=0){
	perror("setpriority()");
}
else{...}
```
## 4) 权限规则（为什么会 EPERM）

- 普通用户一般**只能把 nice 调大**（更“友好”），不能随便调小（更“抢”）
    
- 想把 nice 调小（例如从 0 调到 -5）通常需要特权（root 或相应能力）

## 示例代码：seteuid + get/setpriority 提升 nice 优先级

```C
// elevate_nice_demo.c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <unistd.h>
#include <sys/time.h>
#include <sys/resource.h>

// 安全读取 getpriority：-1 既可能是合法值，也可能是错误
static int safe_getnice(pid_t pid) {
    errno = 0;
    int pr = getpriority(PRIO_PROCESS, pid);
    if (pr == -1 && errno != 0) {
        perror("getpriority");
        exit(1);
    }
    return pr;
}

int main(void) {
    uid_t ruid = getuid();   // 实际用户（通常是启动者）
    uid_t euid = geteuid();  // 有效用户（权限检查看它）

    printf("Start: ruid=%d euid=%d\n", (int)ruid, (int)euid);

    int old_nice = safe_getnice(0);
    printf("Old nice = %d\n", old_nice);

    // 1) 先降权到真实用户（如果当前是 root 或 setuid-root，这一步通常能成功）
    if (seteuid(ruid) < 0) {
        perror("seteuid(drop to ruid)");
        return 1;
    }
    printf("After drop: ruid=%d euid=%d\n", (int)getuid(), (int)geteuid());

    // 2) 需要“提升优先级”（nice 变小）时：临时切到 root
    if (seteuid(0) < 0) {
        perror("seteuid(regain root)");
        fprintf(stderr, "Hint: run as root OR use a setuid-root binary.\n");
        return 1;
    }
    printf("After regain root: ruid=%d euid=%d\n", (int)getuid(), (int)geteuid());

    // 3) 提升优先级：把 nice 调小（例如调到 -5）
    //    注意：nice 越小越“抢”CPU；普通用户一般无权调小
    int target_nice = -5;
    if (setpriority(PRIO_PROCESS, 0, target_nice) < 0) {
        perror("setpriority");
        return 1;
    }

    int new_nice = safe_getnice(0);
    printf("New nice = %d\n", new_nice);

    // 4) 用完特权立刻降权回真实用户
    if (seteuid(ruid) < 0) {
        perror("seteuid(drop again)");
        return 1;
    }
    printf("End: ruid=%d euid=%d\n", (int)getuid(), (int)geteuid());

    return 0;
}
```
