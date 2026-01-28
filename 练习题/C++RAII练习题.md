# 一、概念与理解题（1–6）

### 1. RAII 一句话解释（概念压缩）

请用**不超过 20 个字**解释什么是 RAII，要求包含“构造 / 析构 / 资源”三个关键词。

- **答：资源在构造时获取，并在析构时自动释放。**

---

### 2. 判断题（为什么）

下面说法对还是错？**说明原因**。

> “RAII 只是为了少写几行代码，和程序正确性关系不大。”

- **结论：** 错误。
    
- **原因：** RAII 的核心价值是**异常安全**。在发生 `throw` 或提前 `return` 时，手动释放资源的代码会被跳过导致泄漏，而 RAII 依靠**栈解旋（Stack Unwinding）**确保析构函数一定执行，这是程序正确性的基石。
---

### 3. 成对操作识别

下列哪些操作**适合用 RAII 管理**？（可多选）

- A. `new / delete`
    
- B. `malloc / free`
    
- C. `printf`
    
- D. `fopen / fclose`
    
- E. `lock / unlock`
    
- F. `if / else`
    
**ABDE**
解析：这些操作都有明确的“申请”与“释放”边界。`printf` 是单次调用，`if/else` 是逻辑分支，不需要生命周期管理。

---

### 4. 确定性析构理解

为什么 **C++** 能很好地支持 RAII，而 **GC 语言（如 Java）** 做不到同样的效果？

（提示：析构时机）
- **核心原因：** C++ 的本地对象在超出作用域时会触发**立即且确定**的析构调用。
    
- **对比：** Java 的垃圾回收（GC）是不确定的，资源释放（如 `finalize`）可能在很久之后才发生，甚至不发生。因此 Java 需要手动写 `try-with-resources`，而 C++ 靠对象生命周期即可。
---

### 5. 异常与 RAII

下面两段代码，哪一段是 **异常安全的**？为什么？

```C++
void f() {
    lock();
    do_something();
    unlock();
}
```

```C++
void f() {
    LockGuard lk;
    do_something();
}
```
- **结论：** **第二段代码**是异常安全的。
    
- **原因：** 如果 `do_something()` 抛出异常，第一段代码的 `unlock()` 永远不会执行。而第二段代码中，`lk` 是局部对象，异常发生时系统会清理栈内存，自动调用其析构函数执行 `unlock`。

---

### 6. RAII 的边界

RAII 能否解决**所有资源管理问题**？  
请至少举出 **1 个 RAII 不适合直接解决的场景**。

- **场景：** **循环引用**（例如两个 `shared_ptr` 互相指向）或**异步资源所有权模糊**。
    
- 当资源的生命周期不再由固定的作用域（Scope）决定，或者需要跨网络、跨进程协同释放时，简单的 RAII 往往不够，需要配合引用计数或手动干预。
---

# 二、代码理解与推理题（7–12）

### 7. 析构顺序推理

写出下面程序的输出顺序：

```C++
struct A {
    A() { std::cout << "A"; }
    ~A() { std::cout << "~A"; }
};

struct B {
    B() { std::cout << "B"; }
    ~B() { std::cout << "~B"; }
};

void f() {
    A a;
    B b;
}
```
- **输出：** `AB~B~A`
    
- **原理：** 局部变量存储在栈上，**构造顺序自上而下，析构顺序自下而上**（后进先出）。

---

### 8. RAII 与 return

下面函数中，资源是否一定会被释放？为什么？

```C++
void f() {
    File f("data.txt", "r");
    if (error) return;
}
```
- **结论：** **一定会被释放**。
    
- **原因：** 只要 `f` 是局部对象，在 `return` 发生时，它所在的栈帧被销毁前会先调用其析构函数。
---

### 9. RAII 与 throw

下面代码是否安全？析构函数会不会执行？
```C++
void f() {
    File f("data.txt", "r");
    throw std::runtime_error("err");
}
```
- **结论：** **安全，析构函数会执行**。
    
- **原因：** 这就是所谓的“栈解旋”：异常抛出后，程序会逐层退出函数，并销毁该路径上所有已构造完毕的局部对象。

---

### 10. 禁止拷贝的原因

为什么大多数 RAII 类都要：
```C++
Resource(const Resource&) = delete;
Resource& operator=(const Resource&) = delete;
```
如果不禁止，会出现什么具体问题？
- **问题：** 会导致**重复释放（Double Free）**。
    
- 如果允许拷贝，两个对象会指向同一个底层资源。当它们相继析构时，会对同一个指针调用两次 `delete` 或同一个文件句柄调用两次 `fclose`，导致崩溃或未定义行为。

---

### 11. unique_ptr 是 RAII 吗？

请说明 `std::unique_ptr` 是如何体现 RAII 思想的，并指出：

- 构造时做了什么？
    
- 析构时做了什么？
    
- **是**。它是 RAII 的教科书级实现。
    
- **构造时：** 接收并保存原始指针的所有权。
    
- **析构时：** 自动调用 `delete` 销毁指针所指对象。
---

### 12. RAII vs finally

对比：

- C++ 的 RAII
    
- Java 的 `try { } finally { }`
    

说出 **至少两个本质差异**。
- **自动化程度：** RAII 是隐式的（靠编译器），`finally` 是显式的（靠程序员手写）。
    
- **可组合性：** RAII 随对象走，如果你有 3 个资源，RAII 只需要声明 3 个对象；而 `finally` 可能需要嵌套 3 层 `try` 块，导致代码极度臃肿（“金字塔”代码）。

---

# 三、编程练习题（13–20）✅（重点）

> ⚠️ 以下 **8 题全部需要写代码**

---

## 13. 编程题：最简单的 RAII 类

实现一个 `IntGuard` 类：

- 构造时 `new int`
    
- 析构时 `delete`
    
- 禁止拷贝
    

**要求：**
```C++
IntGuard g(10);
std::cout << *g.get();
```

```C++
class IntGuard{
public:
	IntGuard(){}
	IntGuard(int a): val(new int(a)){}
	~IntGuard(){
		delete val;
	}
	IntGuard(const IntGuard& other)=delete;
	IntGuard& operator=(const IntGuard& other)=delete;
	IntGuard(IntGuard&& other) noexcept 
		: val(other.val){
			other.val=nullptr;
		};
	IntGuard& operator=(IntGuard&& other) noexcept {
		if(this!=&other){
			delete val;
			val=other.val;
			other.val=nullptr;
		}
		return *this;
	};
	int* get() const {return val;}
private:
	int* val = nullptr;
};
```

---

## 14. 编程题：文件 RAII（基础）

封装一个 `FileRAII` 类，管理 `FILE*`：

- 构造函数：`fopen`
    
- 析构函数：`fclose`
    
- 提供 `FILE* get()` 接口
    
- 禁止拷贝
    
```C++
class FileGuard{
public:
    FileGuard(const char* path, const char* mode)
        : fp_(std::fopen(path, mode)) {
        if (!fp_) {
            throw std::runtime_error("fopen failed");
        }
    }
	~FileGuard(){
	if(val){
			fclose(val);
	}
	}
	FileGuard(const FileGuard& other)=delete;
	FileGuard& operator=(const FileGuard& other)=delete;
	FileGuard(FileGuard&& other) noexcept 
		: val(other.val){
			other.val=nullptr;
		};
	FileGuard& operator=(FileGuard&& other) noexcept {
		if(this!=&other){
			if(val)fclose(val);
			val=other.val;
			other.val=nullptr;
		}
		return *this;
	};
	FILE* get() const {return val;}
private:
	FILE* val=nullptr;
};
```

---

## 15. 编程题：RAII + 异常安全验证

写一个函数：

```C++
void test() {
    FileRAII f("data.txt", "r");
    throw std::runtime_error("fail");
}
```

要求你**证明（用输出或注释）**：

- 析构函数一定会被调用
    
```C++
class FileRAII {
private:
    FILE* handle;
public:
    explicit FileRAII(const char* path, const char* mode) {
        handle = fopen(path, mode);
    }
    ~FileRAII() {
        if (handle) fclose(handle);
    }
    FILE* get() const { return handle; }
    
    // 禁止拷贝
    FileRAII(const FileRAII&) = delete;
    FileRAII& operator=(const FileRAII&) = delete;
};
```

---

## 16. 编程题：锁的 RAII（简化版）

实现一个 `MyLockGuard`，用于管理 `std::mutex`：

```C++
class MyLockGuard {
public:
    explicit MyLockGuard(std::mutex& m);
    ~MyLockGuard();
};
```

要求：

- 构造时 `lock`
    
- 析构时 `unlock`
    
- 禁止拷贝
    
```C++
class MyLockGuard {
public:
    explicit MyLockGuard(std::mutex& m):mx(m){
	    mx.lock();
    }
    ~MyLockGuard(){
	    mx.unlock();
    }
    MyLockGuard(const MyLockGuard& m) =delete;
	MyLockGuard& operator=(const MyLockGuard& m) =delete;
private:
std::mutex& mx;
};
```
---

## 17. 编程题：RAII 管理数组（malloc/free）

实现一个 `Buffer` 类：

- 构造：`malloc(n)`
    
- 析构：`free`
    
- 提供 `void* data()` 接口
    
```C++
class Buffer{
public:
	Buffer(){}
	explicit Buffer(ssize_t size): mem(std::malloc(size)){
		if(!mem){
			throw std::runtime_error("malloc failed");
		}
	}
	~Buffer(){
		std::free(mem);
	}
	void * data(){ return mem;}
	Buffer(const Buffer&) =delete;
	Buffer& operator=(const Buffer&)=delete;
private:
void * mem=nullptr;
};
```
---

## 18. 编程题：支持移动语义的 RAII 类（进阶）

在第 17 题基础上：

- 实现 **移动构造**
    
- 实现 **移动赋值**
    
- 确保不会 double free
    
```C++
class Buffer{
public:
	Buffer()=default;
	explicit Buffer(ssize_t size): mem(std::malloc(size)){
		if(!mem){
			throw std::runtime_error("malloc failed");
		}
	}
	~Buffer(){
		std::free(mem);
	}
	void * data() const { return mem;}
	Buffer(const Buffer&) =delete;
	Buffer& operator=(const Buffer&)=delete;
	
	Buffer(Buffer&& other) noexcept 
	: mem(other.mem)
	{
			other.mem=nullptr;
	}
	Buffer& operator=(Buffer&& other) noexcept {
		if(this!=&other){
			std::free(mem);
			mem=other.mem;
			other.mem=nullptr;
		}
		return *this;
	}
private:
void * mem=nullptr;
};
```
---

## 19. 编程题：RAII + early return

写一个函数，满足：

- 中间至少 2 个 `return`
    
- 使用 RAII 管理资源
    
- 不允许显式调用 `free / close / unlock`
    
```C++
void processFile() {
    FileRAII f("config.txt", "r");
    if (!f.get()) return; // 自动销毁（如果有的话）

    MyLockGuard lock(some_mutex);
    if (some_condition) return; // 锁会自动释放，文件会自动关闭

    // 业务逻辑...
}
```
---

## 20. 编程题：设计题（最接近真实工程）

设计一个 `SocketRAII` 类（伪代码即可）：

- 管理 `int fd`
    
- 构造时 `socket()`
    
- 析构时 `close(fd)`
    
- 禁止拷贝
    
- 支持移动
    

说明你的设计思路。
**fd=-1才是关闭**
```C++
class SocketRAII{
public:
	SocketRAII()=default;
	SocketRAII(int domain,int type,int protocol)
		: fd(socket(domain,type,protocol)){
		if(fd==-1){
		throw std::runtime_error("malloc failed");
		}}
	~SocketRAII(){
		if(fd==-1)close(fd);
	}
	SocketRAII(const SocketRAII&)=delete;
	SocketRAII& operator=(const SocketRAII&)=delete;
	SocketRAII(SocketRAII&& other): fd(other.fd){
		other.fd=-1;
	}
	SocketRAII& operator=(SocketRAII&& other){
		if(this!=&other){
			if(fd!=-1)close(fd);
			fd=other.fd;
			other.fd=-1;
		}
		return *this;
	}
private:
int fd =-1;
};
```

---

# 使用建议（非常重要）

- **第 1–12 题**：锻炼你是否真的“理解 RAII”
    
- **第 13–18 题**：决定你能不能“写对 RAII”
    
- **第 19–20 题**：决定你是不是“工程级 C++ 程序员”
    

---