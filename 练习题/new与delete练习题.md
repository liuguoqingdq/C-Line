# 一、概念与理解题（1–10）

### 1️⃣ `new Foo()` 到底做了哪两件事？

请写出等价的“拆解版伪代码”。
```C++
void *mem=::operator new(sizeof(T));
T* p=new(mem) T;
```
**标准答案**
```C++
void* mem = ::operator new(sizeof(Foo));
Foo* p = static_cast<Foo*>(mem);
try {
    new (p) Foo();      // 构造
} catch (...) {
    ::operator delete(mem); // 构造失败要释放内存
    throw;
}
```

---

### 2️⃣ `operator new` 和 `new` 的区别是什么？

为什么说 `operator new` 是函数，而 `new` 是语言特性？

**答案：**

- `new`：**语言关键字/表达式**，语法层面同时负责“分配+构造”
    
- `operator new`：**可调用的函数**，只负责“分配原始内存”，可重载

---

### 3️⃣ 为什么 `malloc(sizeof(Foo))` 得到的内存“不能当 Foo 用”？

请从 **对象生命周期** 的角度解释。

**答：**`malloc` 只提供一块“原始字节内存”，**没有开始 Foo 对象的生命周期**。  
对 `p` 做 `p->member` 或调用成员函数前，必须用 placement new 构造对象，否则行为未定义（尤其对有构造/虚表/不平凡类型）。

---

### 4️⃣ 以下代码中，哪些语句是 **未定义行为（UB）**？说明原因。

```C++
Foo* p = (Foo*)malloc(sizeof(Foo));
p->do_something();
free(p);
```
**答案：**

- `p->do_something()`：**UB**（未构造对象就使用）
    
- `free(p)`：如果对象从未构造，释放裸内存是 OK；但这段整体仍错，因为用了未构造对象
- **正确写法：**
```C++
Foo* mem = (Foo*)malloc(sizeof(Foo));
Foo* p=new(mem) Foo(...);
p->do_something();
p->~Foo();
free(p);
```

---

### 5️⃣ 为什么 `delete` 一定不能用于 `malloc` 得到的内存？

请明确指出 `delete` 内部隐含做了哪些事。
**答案：**  
`delete p` 概念上做：

1. `p->~T()` 调析构
    
2. `operator delete(p)` 释放（与 new 对应）
    

而 `malloc` 得到的内存必须 `free` 释放。**释放机制不匹配 → UB**。

---

### 6️⃣ `new int`、`new int()`、`new int(5)` 的区别是什么？

分别说明初始化规则。
**答案：**

- `new int;`：默认初始化（未初始化，值不确定）
    
- `new int();`：值初始化（变 0）
    
- `new int(5);`：直接初始化（值为 5）
---

### 7️⃣ `delete` 和 `delete[]` 的本质区别是什么？

为什么不能混用？
**答案：**

- `delete`：销毁单个对象，调用 1 次析构
    
- `delete[]`：销毁数组对象，调用 N 次析构（并且需要运行时知道 N，通常由编译器在分配时做“cookie”记录）  
    混用会导致析构次数错误/读取错误 cookie → UB。
---

### 8️⃣ `new(std::nothrow)` 的使用场景是什么？

为什么现代 C++ 很少推荐它？
**答案：**

- 场景：不能抛异常的代码路径（极少数底层库、异常禁用、实时系统）
    
- 不推荐：现代 C++ 倾向异常处理；nothrow 容易漏判空导致崩溃；且 RAII/智能指针更安全。
---

### 9️⃣ placement new **“构造了对象”还是“分配了内存”？**

为什么它常用于 allocator / 内存池？
**答案：**

- placement new **只构造对象，不分配内存**。
    
- 常用于 allocator/内存池：先批量拿一块内存，再在指定位置原地构造对象。
---

### 🔟 下面代码中，`Foo` 的析构函数会被调用几次？

```C++
Foo* p = new Foo[3];
delete[] p;
```
**答案：**  
`~Foo()` 调用 **3 次**，然后释放整块内存 1 次。

---

# 二、判断与推理题（11–20）

### 1️⃣1️⃣ 判断对错，并说明理由

> `free()` 会调用对象的析构函数。

**答案：错。** `free` 只释放内存，不调用析构。

---

### 1️⃣2️⃣ 判断对错

> `delete nullptr;` 是未定义行为。

**答案：错。** `delete nullptr;` 是安全的，无效果。

---

### 1️⃣3️⃣ 判断对错

> placement new 构造的对象，可以直接用 `delete` 释放。

**答案：通常不行（错）。**  
是否能 `delete` 取决于**内存来源**。

- 若内存来自 `operator new`，且对象确实是“new 表达式产生”的那种语义，才可能匹配
    
- 若内存来自 `malloc`/栈/静态缓冲区 → `delete` 必 UB  
    通用正确做法：**手动析构 + 按来源释放**。
---

### 1️⃣4️⃣ 下面代码是否正确？为什么？

```C++
Foo* p = (Foo*)malloc(sizeof(Foo));
new(p) Foo();
delete p;
```
**答案：不正确（UB）。**  
原因：内存来自 malloc，却用 delete 释放。应 `p->~Foo(); free(p);`

---

### 1️⃣5️⃣ 下面代码中是否存在内存泄漏？

```C++
void f() {
    Foo* p = new Foo;
    if (error()) return;
    delete p;
}
```
**答案：有泄漏。** 若 `error()` 为真，直接 return，`p` 没 delete。  
修复：RAII（`std::unique_ptr<Foo> p = std::make_unique<Foo>();`）或 try/finally 风格

---

### 1️⃣6️⃣ `std::unique_ptr<T>` 内部是如何“替代 delete”的？

它和 RAII 的关系是什么？
`unique_ptr` 在析构函数中调用其 deleter（默认 deleter 是 `delete`）。对象离开作用域就析构 `unique_ptr` → 自动释放资源，这就是 RAII。

---

### 1️⃣7️⃣ 为什么 `std::make_unique<T>()` 比 `unique_ptr<T>(new T)` 更安全？
**答案：**

- 避免重复写 `new`，更简洁
    
- 强异常安全：构造过程中异常不会泄漏
    
- 避免“临时裸指针”窗口期（尤其在复杂表达式中）
---

### 1️⃣8️⃣ `shared_ptr` 为什么必须搭配 `weak_ptr` 才能避免循环引用？
**答案：**  
`shared_ptr` 用引用计数，循环引用时计数无法归零，导致泄漏。`weak_ptr` 不增加计数，可打破环。

---

### 1️⃣9️⃣ 判断：

> `operator delete` 一定会调用析构函数。


**答案：错。**  
析构是 `delete` 表达式做的；`operator delete` 仅负责释放内存（像 free 一样的角色）。

---

### 2️⃣0️⃣ 为什么 STL 容器内部几乎不用 `malloc/free`，而是 allocator？

**答案：**

- 分配策略可定制（内存池、对齐、统计）
    
- 分离“分配原始内存”和“构造对象”（`allocate` + placement new）
    
- 支持异常安全、性能优化、类型特性（trivial/非 trivial）

---

# 三、编程练习（21–30）【重点：内存操纵】

> ✅ 下面 **10 题都是编程题**  
> 建议你 **至少写 5 道以上**，收获会非常明显

---

## 🧪 基础内存管理（21–24）

### 2️⃣1️⃣（编程）实现一个 `RawBuffer`

要求：

- 构造：`malloc(n)`
    
- 析构：`free`
    
- 禁止拷贝
    
- 支持移动
    
- 提供 `void* data()` 和 `size()` 接口
    
```C++
#include <cstdlib>
#include <stdexcept>
#include <utility>
#include <cstddef>

class RawBuffer {
public:
    RawBuffer() = default;

    explicit RawBuffer(size_t n) : size_(n) {
        if (n == 0) return;
        data_ = std::malloc(n);
        if (!data_) throw std::bad_alloc();
    }

    ~RawBuffer() {
        std::free(data_);
    }

    RawBuffer(const RawBuffer&) = delete;
    RawBuffer& operator=(const RawBuffer&) = delete;

    RawBuffer(RawBuffer&& other) noexcept
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }

    RawBuffer& operator=(RawBuffer&& other) noexcept {
        if (this != &other) {
            std::free(data_);
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }

    void* data() noexcept { return data_; }
    const void* data() const noexcept { return data_; }//供const对象调用
    size_t size() const noexcept { return size_; }

private:
    void*  data_ = nullptr;
    size_t size_ = 0;
};

```

---

### 2️⃣2️⃣（编程）`MallocObject<T>`（placement new 封装）

实现一个模板类：

`template<typename T> class MallocObject;`

要求：

- 内部用 `malloc`
    
- 构造时用 placement new 构造 `T`
    
- 析构时：`~T()` + `free`
    
- 禁止拷贝，支持移动
    
```C++
template<typename T>
class MallocObject{
public:
	MallocObject()=delete;
	template<typename... Args>
	explicit MallocObject(Args&&... args): mem(std::malloc(sizeof(T))){
		if (!mem) throw std::bad_alloc();
		try{
		t=new(mem) T(std::forward<Args>(args)...);
		}catch(...){
			std::free(mem);
			mem=nullptr;
			t=nullptr;
		}
	}
	~MallocObject(){
		if(t){
		t->~T();
		t=nullptr;				
		}
		free(mem);
		mem=nullptr;
	}
	MallocObject(const MallocObject&)=delete;
	MallocObject& operator=(const MallocObject&)=delete;
	MallocObject(MallocObject&& other):t(other.t),mem(other.mem){
		other.t=nullptr;
		other.mem=nullptr;
	}
	MallocObject& operator=(MallocObject&& other){
		if(this!=&other){
			this->~MallocObject();
			t=other.t;
			mem=other.mem;
			other.t=nullptr;
			other.mem=nullptr;
		}
		return *this;
	}
private:
	T* t=nullptr;
	void * mem=nullptr;
};
```
---

### 2️⃣3️⃣（编程）手写 `make_unique`（简化版）

实现：

```C++
template<typename T, typename... Args>
std::unique_ptr<T> my_make_unique(Args&&... args);
```

要求：

- 使用 `new`
    
- 完美转发构造参数
    
- 返回 `unique_ptr<T>`
    
```C++
template<typename T, typename... Args>
std::unique_ptr<T> my_make_unique(Args&&... args){
		return sdt::unique_ptr<T>(new T(std::forward<Args>(args)...));
		//T* t=new T(std::forward<Args>(args)...);
		//return std::unique_ptr<T>(t);
		//t=nullptr;

		//如果内存是用malloc申请的
		//return std::unique_ptr<T, decltype(&std::free)>(t,&free);
		//t=nullptr;
}

```

---

### 2️⃣4️⃣（编程）`new[] / delete[]` 错误示例修复

下面代码有 UB，请改正：

```C++
Foo* p = new Foo[5];
delete p;
```

```C++
Foo* p = new Foo[5];
delete[] p;
```
---

## 🧠 对象生命周期操纵（25–27）

### 2️⃣5️⃣（编程）在栈上用 placement new 构造对象

```C++
alignas(Foo) char buf[sizeof(Foo)];
```

要求：

- 在 `buf` 上构造 `Foo`
    
- 手动析构
    
- 不能 `delete`
    
```C++
#include <new>
#include <utility>
#include <cstddef>

struct Foo {
    Foo(int, int) {}
    ~Foo() {}
};

int main() {
    alignas(Foo) unsigned char buf[sizeof(Foo)];

    Foo* p = new (buf) Foo(1, 2);  // 构造（不分配）
    // 使用 p...

    p->~Foo();                     // 手动析构
    // 注意：不能 delete p，也不能 free(buf)
}
```
---

### 2️⃣6️⃣（编程）实现一个 `ObjectPool<Foo>`（简化）

要求：

- 内部维护一块连续内存
    
- 用 placement new 构造对象
    
- 提供 `allocate()` / `deallocate()`
    
```C++
#include <new>
#include <cstddef>
#include <vector>
#include <stdexcept>

template<typename T>
class ObjectPool {
public:
    explicit ObjectPool(size_t cap) : cap_(cap) {
        storage_.resize(cap_);
        free_.reserve(cap_);
        for (size_t i = 0; i < cap_; ++i) free_.push_back(i);
    }

    template<typename... Args>
    T* allocate(Args&&... args) {
        if (free_.empty()) throw std::bad_alloc();
        size_t idx = free_.back();
        free_.pop_back();
        void* place = &storage_[idx];
        return new (place) T(std::forward<Args>(args)...);
    }

    void deallocate(T* p) noexcept {
        if (!p) return;
        // 计算索引
        auto base = reinterpret_cast<unsigned char*>(storage_.data());
        auto cur  = reinterpret_cast<unsigned char*>(p);
        size_t idx = (cur - base) / sizeof(Slot);

        p->~T();
        free_.push_back(idx);
    }

    ~ObjectPool() {
        // 简化：不追踪哪些已分配，真实实现需追踪并析构
    }

private:
    union Slot {
        alignas(T) unsigned char bytes[sizeof(T)];
    };

    size_t cap_;
    std::vector<Slot> storage_;
    std::vector<size_t> free_;
};
```
---

### 2️⃣7️⃣（编程）实现一个“延迟构造”的类

要求：

- 构造函数不构造 `Foo`
    
- 提供 `init()` 用 placement new 构造
    
- 析构时判断是否已构造再析构
    
```C++
#include <new>
#include <utility>

template<typename T>
class Lazy {
public:
    Lazy() = default;

    template<typename... Args>
    void init(Args&&... args) {
        if (inited_) return; // 或抛异常
        ptr_ = new (buf_) T(std::forward<Args>(args)...);
        inited_ = true;
    }

    bool inited() const { return inited_; }

    T* get() { return inited_ ? ptr_ : nullptr; }

    ~Lazy() {
        if (inited_) {
            ptr_->~T();
            inited_ = false;
            ptr_ = nullptr;
        }
    }

    Lazy(const Lazy&) = delete;
    Lazy& operator=(const Lazy&) = delete;

private:
    alignas(T) unsigned char buf_[sizeof(T)];
    T* ptr_ = nullptr;
    bool inited_ = false;
};
```
---

## 🧯 错误检测与 RAII（28–30）

### 2️⃣8️⃣（编程）修复 double delete

下面代码有严重错误，请重构为安全版本：

```C++
#include <utility>

class Good {
public:
    Good() : p(new int(1)) {}
    ~Good() { delete p; }

    Good(const Good&) = delete;
    Good& operator=(const Good&) = delete;

    Good(Good&& other) noexcept : p(other.p) {
        other.p = nullptr;
    }

    Good& operator=(Good&& other) noexcept {
        if (this != &other) {
            delete p;
            p = other.p;
            other.p = nullptr;
        }
        return *this;
    }

private:
    int* p = nullptr;
};
```

要求：

- 禁止拷贝
    
- 支持移动
    
- 不发生 double delete
    

---

### 2️⃣9️⃣（编程）实现一个带自定义 deleter 的 `unique_ptr`

要求：

- 管理 `malloc` 得到的内存
    
- 析构时自动 `free`
    
```C++
#include <memory>
#include <cstdlib>

struct FreeDeleter {
    void operator()(void* p) const noexcept {
        std::free(p);
    }
};

using MallocPtr = std::unique_ptr<void, FreeDeleter>;

MallocPtr make_malloc(size_t n) {
    void* p = std::malloc(n);
    if (!p) throw std::bad_alloc();
    return MallocPtr(p);
}
```
如果要“带类型”的版本（更常用）：
```C++
template<typename T>
struct FreeDeleterT {
    void operator()(T* p) const noexcept {
        std::free(p);
    }
};

template<typename T>
using MallocUPtr = std::unique_ptr<T, FreeDeleterT<T>>;
```
---

### 3️⃣0️⃣（编程）RAII 封装 `FILE*`

实现一个 `FileRAII`：

- 构造：`fopen`
    
- 析构：`fclose`
    
- 禁止拷贝
    
- 支持移动
    
```C++
class FILEopen{
#include <cstdio>
#include <stdexcept>
#include <utility>

class FileRAII {
public:
    FileRAII() = default;

    FileRAII(const char* path, const char* mode) {
        fp_ = std::fopen(path, mode);
        if (!fp_) throw std::runtime_error("fopen failed");
    }

    ~FileRAII() {
        if (fp_) std::fclose(fp_);
    }

    FileRAII(const FileRAII&) = delete;
    FileRAII& operator=(const FileRAII&) = delete;

    FileRAII(FileRAII&& other) noexcept : fp_(other.fp_) {
        other.fp_ = nullptr;
    }

    FileRAII& operator=(FileRAII&& other) noexcept {
        if (this != &other) {
            if (fp_) std::fclose(fp_);
            fp_ = other.fp_;
            other.fp_ = nullptr;
        }
        return *this;
    }

    std::FILE* get() const noexcept { return fp_; }

private:
    std::FILE* fp_ = nullptr;
};
```
---

## 🎯 建议练习顺序（强烈推荐）

`1 → 2 → 3 → 5 → 9 21 → 22 → 25 28 → 29 → 30`