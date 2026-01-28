# Makefile 详细教学资料

> 适合：已经会用 gcc/g++ 编译 C/C++ 程序，想系统掌握 Makefile 的同学。

---

## 一、为什么需要 Makefile？

### 1.1 手动编译的问题

假设你有一个小项目：

```text
project/
├── main.c
├── foo.c
└── foo.h
```

手工编译：

```bash
gcc -c main.c -o main.o
gcc -c foo.c  -o foo.o
gcc main.o foo.o -o app
```

如果你改了 `foo.c`，理论上只需要重新编译 `foo.c`，但很多人会直接：

```bash
gcc -c main.c -o main.o
…
```

- 浪费时间
- 容易忘记编译哪些文件，产生奇怪错误

### 1.2 Makefile 的目的

`make` + `Makefile` 做三件事：

1. 根据**依赖关系**决定哪些文件需要重新编译；
2. 按正确顺序自动执行编译 / 链接命令；
3. 提供统一入口：`make` / `make clean` / `make test` 等。

核心思想：

> 把“如何从一堆源文件得到目标文件”的规则写在 Makefile 里，交给 `make` 自动判断和执行。

---

## 二、Makefile 的基本语法模型

### 2.1 规则的一般形式

```make
目标: 依赖1 依赖2 ...
	命令1
	命令2
```

注意：

- `\t` **必须是 TAB，而不是空格**。
- 一个目标可以有多个依赖、多个命令。

示例：

```make
app: main.o foo.o
	gcc main.o foo.o -o app

main.o: main.c foo.h
	gcc -c main.c -o main.o

foo.o: foo.c foo.h
	gcc -c foo.c -o foo.o
```
如果是用Header文件夹和Source文件夹：
```lua
project/
├── main.c
├── Source/foo.c
└── Header/foo.h
```
对于g++编译命令来说：
```shell
cd project

# 1. 直接一步编译链接成可执行文件 app
g++ -IHeader main.c Source/foo.c -o app
```
或者分部编译：
```shell
# 1. 编译生成 .o 文件
g++ -IHeader -c main.c -o main.o
g++ -IHeader -c Source/foo.c -o Source/foo.o

# 2. 链接生成最终程序
g++ main.o Source/foo.o -o app
```

### 2.2 `make` 的默认目标

- 没有指定目标时：
  - `make` 会构建 Makefile 中 **第一个目标**，通常我们写成 `all`。

```make
all: app

app: main.o foo.o
	gcc main.o foo.o -o app
```

---

## 三、make 的工作原理（依赖 + 时间戳）

1. 选择目标：用户执行 `make app`，或默认目标。
2. 找到该目标的依赖。
3. 对每个依赖递归判断：
   - 依赖是否存在？不存在就先构建依赖。
   - 依赖的“修改时间”是否比目标新？如果是，则重新构建目标。

结论：

> Make 的核心是“**文件时间戳** + **依赖图**”。

这也是为什么：改动了 `foo.c`，只会触发 `foo.o` 以及依赖它的目标重新编译。

---

## 四、变量（Variables）

### 4.1 基本用法

```make
CC   = gcc
CFLAGS = -g -Wall

app: main.o foo.o
	$(CC) $(CFLAGS) main.o foo.o -o app
```

使用时写成 `$(变量名)`。
`-g` = **生成调试信息（debug info）**

- 编译出来的可执行文件里，会多带一份：
    
    - 源代码行号
        
    - 变量名
        
    - 函数名、文件名等调试用信息
        
- 这样你就可以用 **gdb**、lldb 之类的调试器：
    
    - 下断点：`break main` / `break foo.c:42`
        
    - 单步执行：`next / step`
        
    - 看变量：`print`
## `-Wall` 是干嘛的？

`-Wall` = **打开一大批常见的编译警告（Warnings All 的意思）**

注意：**不是“所有所有警告”**，只是“GCC 认为最重要的一组”。

开启后能帮你发现很多坑，比如：

- 使用了未初始化的变量
    
- 函数声明和定义不匹配
    
- 隐式转换可能丢失精度
    
- `if (a = b)` 这种疑似写错的代码
    
- 没有使用的变量、没有使用的参数等等

### 4.2 几种赋值形式

常见的有四种：

```make
A = xxx      # 递归展开（lazy）
B := xxx     # 简单展开（立即求值）
C ?= xxx     # 若未定义则赋值
D += xxx     # 在原基础上追加
```

#### 4.2.1 递归赋值 `=`

```make
X = $(Y)
Y = 123

foo:
	@echo $(X)
```

输出：`123`（在使用时才展开）。

#### 4.2.2 简单赋值 `:=`

```make
X := $(Y)
Y := 123
```

- `X` 在定义时就展开，当时 `Y` 是空，所以 `X` 为空。
- 适合希望“**固定一次**”的值，例如编译器版本、系统命令结果等。

#### 4.2.3 条件赋值 `?=`

```make
CC ?= gcc
```

- 如果环境变量或命令行里已经定义了 `CC`，则保持原值；
- 否则设为 `gcc`。

#### 4.2.4 追加 `+=`

```make
CFLAGS = -Wall
CFLAGS += -g
```

---

## 五、自动变量（Automatic Variables）

在配方（命令）中常用的自动变量：

- `$@`：当前规则的目标名
- `$<`：第一个依赖文件
- `$^`：所有依赖文件（去重）
- `$?`：比目标新的依赖

示例：

```make
app: main.o foo.o
	$(CC) $(CFLAGS) $^ -o $@

main.o: main.c foo.h
	$(CC) $(CFLAGS) -c $< -o $@

foo.o: foo.c foo.h
	$(CC) $(CFLAGS) -c $< -o $@
```

- `app` 目标：`$^` 展开为 `main.o foo.o`，`$@` 展开为 `app`；
- `main.o` 规则：`$<` 是 `main.c`，`$@` 是 `main.o`。

---

## 六、模式规则（Pattern Rules）与通配

### 6.1 模式规则

当有很多 `*.c -> *.o` 的规则时，可以写成一条：

```make
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```

- `%` 匹配任意前缀；
- 如果需要从子目录编译，可以写：

```make
obj/%.o: src/%.c
	$(CC) $(CFLAGS) -c $< -o $@
```

### 6.2 利用变量管理对象文件

```make
SRCS = main.c foo.c bar.c
OBJS = $(SRCS:.c=.o)

app: $(OBJS)
	$(CC) $(CFLAGS) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```

- `$(SRCS:.c=.o)` 是“模式替换”：把所有 `.c` 替换成 `.o`。

---

## 七、伪目标（.PHONY）

有些“目标”并不是文件，例如 `clean`：

```make
clean:
	rm -f *.o app
```

如果目录下刚好有一个名为 `clean` 的文件，执行 `make clean` 会出现奇怪行为。为避免冲突，要声明伪目标：

```make
.PHONY: all clean

all: app

clean:
	rm -f *.o app
```

- `.PHONY` 告诉 `make`：这些目标不对应真实文件，每次都要执行。

---

## 八、多文件项目的典型 Makefile

假设项目结构：

```text
project/
├── main.c
├── foo.c
├── foo.h
├── bar.c
└── bar.h
```

一个常见的 Makefile 模板：

```make
CC      = gcc
CFLAGS  = -Wall -g

SRCS    = main.c foo.c bar.c
OBJS    = $(SRCS:.c=.o)
TARGET  = app

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)
```

使用：

```bash
make       # 构建 app
make clean # 清理
```

---

## 九、Make 的内置规则与覆盖

- `make` 自带许多隐式规则，例如：

```text
%.o: %.c
	$(CC) -c $(CPPFLAGS) $(CFLAGS) $< -o $@
```

- 如果你没有写自己的 `.o` 规则，`make` 会尝试使用内置规则；
- 通常建议**显式写出自己的规则**，便于控制 CFLAGS 等参数。

可以通过：

```bash
make -p
```

查看所有内置规则（输出很长，一般只用调试时看）。

---

## 十、函数：wildcard / patsubst / addprefix / shell

Makefile 里有很多类似函数的语法：`$(func args...)`。

### 10.1 `$(wildcard pattern)`

列出匹配的文件：

```make
SRCS = $(wildcard src/*.c)
OBJS = $(patsubst src/%.c, build/%.o, $(SRCS))
```

### 10.2 `$(patsubst pattern,replacement,text)`

模式替换：

```make
OBJS = $(patsubst %.c, %.o, $(SRCS))
```

### 10.3 `$(addprefix prefix,names...)`

加前缀：

```make
SRC_NAMES = main foo bar
SRCS = $(addprefix src/, $(addsuffix .c, $(SRC_NAMES)))
```

### 10.4 `$(shell command)`

执行 shell 命令，捕获输出：

```make
GIT_HASH := $(shell git rev-parse --short HEAD)

version.h:
	@echo "#define GIT_HASH \"$(GIT_HASH)\"" > $@
```

注意：

- `$(shell ...)` 会启动子进程，频繁使用会影响性能；
- 适合做版本号、时间戳等不常变的内容。

---

## 十一、条件判断（Conditional）

Makefile 支持简单的条件编译：

```make
DEBUG ?= 0

ifeq ($(DEBUG), 1)
  CFLAGS += -g -O0
else
  CFLAGS += -O2
endif
```

- `ifeq (a, b)` / `ifneq (a, b)` / `ifdef VAR` / `ifndef VAR` 等。

可配合命令行参数：

```bash
make DEBUG=1
```

---

## 十二、多目录工程的组织方式

### 12.1 顶层驱动子目录

项目结构：

```text
project/
├── src/
│   ├── main.c
│   └── foo.c
├── include/
│   └── foo.h
└── Makefile
```

顶层 Makefile：

```make
.PHONY: all clean

all:
	$(MAKE) -C src

clean:
	$(MAKE) -C src clean
```

`src/Makefile`：

```make
CC      = gcc
CFLAGS  = -Wall -g -I../include

SRCS    = main.c foo.c
OBJS    = $(SRCS:.c=.o)
TARGET  = app

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)
```

- 顶层只负责调度子目录；
- 子目录各自管理自己的源码。

### 12.2 include 方式拆分 Makefile

也可以把公共配置单独放在 `config.mk` 中：

```make
# config.mk
CC     = gcc
CFLAGS = -Wall -g
```

在主 Makefile 中：

```make
include config.mk

app: main.o foo.o
	$(CC) $(CFLAGS) $^ -o $@
```

- `include` 类似 C 语言的 `#include`，在预处理阶段把文件内容展开进来。

---

## 十三、自动生成依赖（.d 文件）

手动写每个 `.o` 依赖哪些头文件很痛苦，可以用编译器自动生成：

```make
CC      = gcc
CFLAGS  = -Wall -g -MMD -MP

SRCS    = main.c foo.c bar.c
OBJS    = $(SRCS:.c=.o)
DEPS    = $(SRCS:.c=.d)

TARGET  = app

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) $^ -o $@

-include $(DEPS)

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(DEPS) $(TARGET)
```

说明：

- `-MMD -MP`：让 gcc 在编译时自动生成 `.d` 依赖文件；
- `-include`：即使依赖文件不存在也不报错（第一次编译时没有 `.d` 很正常）。

这样，当你修改头文件时，相关 `.o` 会自动重新编译。

---

## 十四、一个通用 C/C++ 工程 Makefile 模板

下面是一个略完整一些的模板，适合作为起点：

```make
# =============================
# 通用 C/C++ Makefile 模板
# =============================

# 编译器与标志
CC      ?= gcc
CXX     ?= g++

CFLAGS   = -Wall -Wextra -g
CXXFLAGS = $(CFLAGS) -std=c++17
LDFLAGS  =

# 目录与文件
SRC_DIR  = src
INC_DIR  = include
BUILD_DIR = build

SRCS     = $(wildcard $(SRC_DIR)/*.c) $(wildcard $(SRC_DIR)/*.cpp)
OBJS     = $(patsubst $(SRC_DIR)/%.c,$(BUILD_DIR)/%.o,$(SRCS:.cpp=.o))
DEPS     = $(OBJS:.o=.d)

TARGET   = app

# 启用自动依赖
CFLAGS   += -MMD -MP
CXXFLAGS += -MMD -MP

.PHONY: all clean run

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CXX) $(OBJS) $(LDFLAGS) -o $@

# C 源文件规则
$(BUILD_DIR)/%.o: $(SRC_DIR)/%.c | $(BUILD_DIR)
	$(CC) $(CFLAGS) -I$(INC_DIR) -c $< -o $@

# C++ 源文件规则
$(BUILD_DIR)/%.o: $(SRC_DIR)/%.cpp | $(BUILD_DIR)
	$(CXX) $(CXXFLAGS) -I$(INC_DIR) -c $< -o $@

# 创建 build 目录
$(BUILD_DIR):
	mkdir -p $(BUILD_DIR)

# 自动包含依赖文件
-include $(DEPS)

clean:
	rm -rf $(BUILD_DIR) $(TARGET)

run: all
	./$(TARGET)
```

使用说明：

1. 代码放在 `src/`，头文件放在 `include/`；
2. 对象文件和依赖文件放在 `build/`；
3. `make` 构建，`make run` 运行，`make clean` 清理。

---

## 十五、学习建议与进阶方向

1. **边学边写**
   - 看完一小节，就写一个小 Makefile 测试；
   - 故意写错路径 / 删除某些文件，观察 `make` 行为。

2. **学会调试 Makefile**
   - `make -n`：只打印命令，不执行；
   - `make VERBOSE=1`（配合你自己的变量）
   - 使用 `$(info ...)` 在 Makefile 中打印变量值：

```make
$(info SRCS = $(SRCS))
```

3. **多读开源项目的 Makefile / CMakeLists**
   - Linux 内核、Redis、nginx 等都有各自的构建系统；
   - 学习他们如何组织多个目录、不同平台选项。

4. **了解替代工具**
   - 大型工程常用 CMake、Meson、Bazel 等；
   - 但理解 Makefile 有助于看懂这些工具生成的底层规则。

---

如果你有具体的项目结构（比如你准备做一个小服务器或实验项目），可以把目录贴给我，我可以基于这份教程，帮你直接写一份适配你当前