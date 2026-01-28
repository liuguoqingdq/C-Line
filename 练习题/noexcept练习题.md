# 一、基础理解题（1–6）

### 1️⃣（判断题）

下面说法是否正确？为什么？

> 给函数加上 `noexcept`，只是给编译器一个优化提示，即使抛异常也能被正常捕获。

**答案：错。**  
**解析：**  
`noexcept` 是**强承诺**：函数声明为 `noexcept(true)` 后，若异常**逃逸出函数体**（没在内部 catch），标准要求调用 `std::terminate()`，**不会**像普通函数那样继续向上传播并被外层 catch。它不是“优化提示”，而是语义契约。

---

### 2️⃣（选择题）

以下哪种写法**等价于未写 `noexcept`**？

A. `void f() noexcept(false);`  
B. `void f() noexcept(true);`  
C. `void f() noexcept;`  
D. `void f() noexcept(noexcept(true));`

**答案：A。**  
**解析：**

- `noexcept(false)` 明确表示“可能抛”，等价于不写。
    
- `noexcept` / `noexcept(true)` 承诺不抛。
    
- `noexcept(noexcept(true))`：内层 `noexcept(true)` 是编译期常量 `true`，外层变成 `noexcept(true)`，所以不是等价于没写。
---

### 3️⃣（简答题）

为什么说 **`noexcept` 是“强承诺”而不是“建议”**？  
请描述违反 `noexcept` 承诺时发生的完整过程。
**答案要点：**

- `noexcept` 是函数异常说明：承诺“不抛”。
    
- 若异常从 `noexcept` 函数逃逸：运行时发现违反承诺 → 调 `std::terminate()` → 程序终止。  
    **解析：**  
    这条规则的目的之一是允许编译器/STL做更激进的优化与分支选择（例如容器重分配时选择 move）。
---

### 4️⃣（判断题）

`bool b = noexcept(throw 1);`

这段代码是否合法？`b` 的值是多少？**为什么？**
**答案：合法；值为 `false`。**  
**解析：**  
`noexcept(expr)` 不会真的执行 `expr`，而是编译期判断“这个表达式是否可能抛异常”。  
`throw 1` 显然会抛，因此 `noexcept(throw 1)` 为 `false`。  
（注意：`noexcept(expr)` 运算符本身不抛异常。）

---

### 5️⃣（简答题）

下面两个东西的本质区别是什么？

`void f() noexcept; 
`bool b = noexcept(f());`

**答案：**

- `void f() noexcept;`：**异常说明**，改变函数契约与运行时行为（违约 terminate）。
    
- `noexcept(f())`：**运算符**，编译期产生一个 `bool` 常量，表示“表达式是否可能抛”，本身不抛。  
    **解析：**  
    一个是“承诺”，一个是“询问”。

---

### 6️⃣（推理题）

下面代码中，`foo` 是否是 `noexcept`？说明理由。

`void bar(); 
`void foo() noexcept(noexcept(bar())) {     bar(); }`

**答案：不一定；取决于 `bar()` 是否声明为 `noexcept`。**  
**解析：**

- 若 `bar()` 是 `noexcept`，则 `noexcept(bar()) == true`，`foo()` 变成 `noexcept(true)`。
    
- 若 `bar()` 可能抛（默认），`noexcept(bar()) == false`，`foo()` 等价于 `noexcept(false)`（相当于没写）。
---

# 二、`noexcept` 与对象语义（7–11）

### 7️⃣（判断题）

以下代码中，`A` 的移动构造函数是否一定会被 `std::vector<A>` 在扩容时使用？

`struct A {     A(A&&) {} };`
**答案：不一定。**  
**解析：**  
很多实现会在重分配时做类似判断：

- 如果 `T` **nothrow move constructible**（通常由 `noexcept` 推导），就移动；
    
- 否则可能退回拷贝（更强异常安全）。  
    这里 `A(A&&)` **没写 noexcept** → 通常不被认为是 nothrow move → 容器可能选择拷贝（若可拷贝），或在某些条件下仍移动但异常安全策略更复杂（以实现为准）。从你给出的内容角度：关键点就是 **是否 `noexcept` 影响分支选择**。
---

### 8️⃣（简答题）

为什么 **`std::vector` 在扩容时更偏向使用 `noexcept` 的移动构造**，而不是“只要有移动构造就用”？
**答案要点：异常安全。**  
**解析：**  
扩容要把旧缓冲区元素搬到新缓冲区：

- 若移动构造可能抛，搬迁一半抛了会导致“新旧两边状态半死不活”，实现强异常安全更难。
    
- 若移动构造 `noexcept`，则搬迁不会中途失败，容器能更轻松保证强异常安全，因此优先走 move 路径。
---

### 9️⃣（判断题）

`struct A {     ~A() noexcept(false) {         throw 1;     } };`

这个类型在栈展开期间被销毁是否安全？为什么？
**答案：不安全（非常危险）。**  
**解析：**

- 析构函数在栈展开（异常传播）期间被调用时，如果又抛异常，会触发 `std::terminate()`（双异常）。
    
- 即使显式写 `noexcept(false)`，在栈展开场景依然致命。  
    工程规则：**析构不要抛**。
- C++11开始所有析构函数默认noexcept(true)
---

### 🔟（推理题）

下面哪一个类型更“适合”放进 `std::vector`，并频繁发生扩容？请说明理由。
```C++
struct A { A(A&&) noexcept; };
struct B { B(B&&); };
```
**答案：A 更适合。**  
**解析：**  
A 的移动构造不抛，使 `std::vector<A>` 扩容更可能走 move 分支，搬迁更快、更简单。  
B 的移动构造可能抛，容器为异常安全可能退回拷贝或走更保守路径，通常更慢/更复杂。

---

### 1️⃣1️⃣（简答题）

为什么说 **“给移动构造加 `noexcept` 是性能承诺，而不是语法装饰”**？
**答案要点：STL 分支选择 + 优化空间。**  
**解析：**

- STL（如 vector）会根据 `is_nothrow_move_constructible` 选择 move/copy。
    
- 编译器也能省去异常处理元数据、做更大胆内联与路径简化。  
    因此 `noexcept` 直接影响性能与实现策略。
---

# 三、条件 `noexcept` 与模板（12–15）

### 1️⃣2️⃣（判断题）

```C++
template <typename T>
void f(T&&) noexcept(noexcept(T(std::declval<T>())));
```

这个 `f` 的 `noexcept` 是否依赖于 `T`？  
它是在**运行期**还是**编译期**决定的？
**答案：依赖 `T`；在编译期决定。**  
**解析：**  
`noexcept(...)` 运算符是编译期常量表达式，模板实例化时根据 `T` 的构造是否 `noexcept` 决定 `f` 的异常说明。

---

### 1️⃣3️⃣（简答题）

解释下面代码中 **内外两层 `noexcept` 的作用**：

`void f() noexcept(noexcept(g()));`

**答案：**

- 内层 `noexcept(g())`：编译期询问“调用 `g()` 是否可能抛”，得到 `true/false`。
    
- 外层 `noexcept(<bool>)`：把这个布尔值作为 `f` 的异常说明条件：
    
    - 若内层为 true → `f` 是 `noexcept(true)`
        
    - 否则 `f` 是 `noexcept(false)`  
        **核心：**“`f` 是否不抛”由“`g` 是否不抛”决定。

---

### 1️⃣4️⃣（推理题）

```C++
template <typename T>
void push(T&&) noexcept(std::is_nothrow_move_constructible_v<T>);
```

如果 `T = std::string`，`push` 是否是 `noexcept`？为什么不能直接写成 `noexcept(true)`？
**答案：**

- 是否 `noexcept` 取决于 `std::string` 的 move 构造是否被实现声明为 `noexcept`（通常是，但你这里的练习按原则回答：**依赖实现/类型特性**）。
    
- 不能写死 `noexcept(true)`，因为对某些 `T`（移动可能抛）写死会变成“错误承诺”，一旦异常逃逸就 terminate。  
    **解析：**  
    泛型代码不能假设所有 `T` 都不抛，条件 `noexcept` 才是正确工程做法。
---

### 1️⃣5️⃣（设计题）

你在设计一个泛型容器接口，什么时候应该：

- 使用 `noexcept(true)`
    
- 使用 **条件 `noexcept`**
    
- 完全不写 `noexcept`
    

请各举一个典型场景。
**参考答案：**

- `noexcept(true)`：你能严格保证不抛，且违约就是 bug。例：交换两个指针、移动一个 trivially movable 的资源句柄、`size()`、简单 getter。
    
- 条件 `noexcept`：模板/泛型封装，是否不抛取决于 `T` 或某个操作。例：完美转发构造包装、容器 `push_back`/`emplace` 内部依赖 `T` 构造。
    
- 不写：函数确实可能抛，或你无法保证不抛。例：分配内存、解析输入、可能 throw 的业务逻辑。
---

# 四、编程题（16–20）⭐重点

---

## 1️⃣6️⃣【编程题】

下面代码会发生什么？**能否编译？运行结果是什么？**

```C++
#include <iostream>

void f() noexcept {
    throw 42;
}

int main() {
    try {
        f();
    } catch (...) {
        std::cout << "caught\n";
    }
}

```

👉 要求：

- 写出 **是否会进入 `catch`**
    
- 写出 **是否调用 `std::terminate`**
    
- 说明原因
    
**结论：**

- **不会进入 `catch`**
    
- 会调用 **`std::terminate()`**（程序直接终止）  
    **解析：**  
    异常从 `noexcept` 函数逃逸 → 违反承诺 → terminate，异常不会再传播到外层 try/catch。
---

## 1️⃣7️⃣【编程题】

补全下面代码，使 `static_assert` 成立：

```C++
#include <type_traits>

struct A {
    A(A&&) noexcept;
};

struct B {
    B(B&&);
};

static_assert(std::is_nothrow_move_constructible_v<A>);
static_assert(!std::is_nothrow_move_constructible_v<B>);
```

👉 要求：

- 不允许改 `static_assert`
    
- 只能修改 `A / B`
    
```C++
#include <type_traits>

struct A {
    A() = default;
    A(A&&) noexcept = default;
};

struct B {
    B() = default;
    B(B&&) = default; // 不写 noexcept => 默认可能抛
};

static_assert(std::is_nothrow_move_constructible_v<A>);
static_assert(!std::is_nothrow_move_constructible_v<B>);
```
---

## 1️⃣8️⃣【编程题】

实现一个**完美转发的包装函数**，要求：

- 如果被包装的构造操作不抛异常 → 包装函数也是 `noexcept`
    
- 否则 → 包装函数可能抛
    

```C++
template <typename T, typename... Args>
T make(Args&&... args) /* 在这里补 noexcept */ {
    return T(std::forward<Args>(args)...);
}
```
**答案**
```C++
#include <utility>

template <typename T, typename... Args>
T make(Args&&... args) noexcept(noexcept(T(std::forward<Args>(args)...))) {
    return T(std::forward<Args>(args)...);
}

```
---

## 1️⃣9️⃣【编程题】

下面代码中，`Vec<T>::push_back` 的 `noexcept` 是否正确？  
如果不完全正确，请修改。

```C++
template <typename T>
struct Vec {
    void push_back(T&& t) noexcept {
        T x(std::move(t));
    }
};
```

👉 要求说明 **错误点 + 修改理由**
**答案：不正确（通常是错的）。**  
**原因：**  
`T x(std::move(t));` 这一步依赖 `T` 的移动构造是否抛异常。你写死 `noexcept` 等价于 `noexcept(true)`，对“移动可能抛”的 `T` 会造成**错误承诺**，一旦抛出并逃逸就 terminate。

**修改：**
```C++
#include <type_traits>
#include <utility>

template <typename T>
struct Vec {
    void push_back(T&& t) noexcept(std::is_nothrow_move_constructible_v<T>) {
        T x(std::move(t));
    }
};
```
---

## 2️⃣0️⃣【编程题 · 综合】

实现一个类型 `SafeMove<T>`，要求：

- 只有当 `T` 的移动构造是 `noexcept` 时，`SafeMove<T>` 的移动构造才是 `noexcept`
    
- 否则回退为“可能抛”
    

```C++
template <typename T>
struct SafeMove {
    T value;

    // 在这里实现
};
```

```C++
#include <type_traits>
#include <utility>

template <typename T>
struct SafeMove {
    T value;

    SafeMove() = default;
    SafeMove(const SafeMove&) = default;

    SafeMove(SafeMove&& other) noexcept(std::is_nothrow_move_constructible_v<T>)
        : value(std::move(other.value)) {}

    SafeMove& operator=(SafeMove&& other) noexcept(std::is_nothrow_move_assignable_v<T>) {
        value = std::move(other.value);
        return *this;
    }
};
```

---

## 🎯 建议你怎么刷这些题

你可以按 **三轮** 来：

1. **第一轮**：只写“结论”（对 / 错 / 是否 noexcept）
    
2. **第二轮**：强迫自己写“为什么”
    
3. **第三轮（编程题）**：
    
    - 自己先写
        
    - 再用 `static_assert` / `type_traits` 验证