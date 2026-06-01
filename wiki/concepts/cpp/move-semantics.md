---
title: "移动语义与完美转发"
type: concept
created: 2026-05-28
updated: 2026-05-28
sources: ["raw/C++ cook book/语法基础.md"]
questions: [1]
tags: [c++11, move-semantics, templates, performance]
---

## 核心三件套

|                | 作用              | 本质                                         |
| -------------- | --------------- | ------------------------------------------ |
| `std::move`    | 把左值"标记"为右值，允许移动 | 无条件 cast 到 `T&&`                           |
| `std::forward` | 条件性地恢复原值类别      | 有条件 cast（lvalue → lvalue, rvalue → rvalue） |
| 万能引用 `T&&`     | 可绑定到左值或右值的模板参数  | 引用折叠规则推导                                   |

## 左值与右值：类型和值类别是两回事

这是最核心的入门坑：`T&&` 是**类型**，不是值类别。有名字的变量，无论类型是什么，作为表达式都是**左值**。

```cpp
std::string make_string() { return "hello"; }

std::string&& r = make_string();  // 类型是 std::string&&（右值引用）

std::string a = r;                // 拷贝！表达式 r 是左值
std::string b = std::move(r);    // 移动。std::move 把 r 转成右值表达式
```

### 为什么 C++ 要这样设计？

**避免隐式地反复移动同一个具名对象**。如果任何 `T&&` 变量出现在表达式中都自动当作右值，那么第一次使用后就悄无声息地被移动走了，后续使用就是 use-after-move。强制要求用 `std::move` 显式标记"我确认这个变量以后不会再用了"。

## std::move 只是强制转换

```cpp
// std::move 的标准实现（示意）
template <typename T>
constexpr std::remove_reference_t<T>&& move(T&& t) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(t);
}
```

`std::move` **不移动任何数据**。它只做一件事：把表达式的值类别转成右值。真正的资源转移发生在移动构造函数/移动赋值运算符里。

常见误解：
```cpp
std::string s1 = "hello";
std::string s2 = std::move(s1);  // s2 调用 move constructor 接管 s1 的 buffer
// 此时 s1 处于"有效但未指定"的状态（通常是空字符串，但不保证）
// s2 的内容是 "hello"
```

要理解的关键链：`std::move` 产生右值引用 → 重载决议选中移动构造函数 → 移动构造函数实际执行资源转移。

### 移动语义的优化本质

为什么移动构造比拷贝快？

```
拷贝构造：分配新 buffer → 复制 s1 的内容 → s2 持有新 buffer
                    ↑ 一次堆分配 + memcpy

移动构造：s2 直接偷走 s1 的指向 buffer 的指针 → 把 s1 的内部指针置空
                    ↑ 零分配，只改几个指针成员
```

```cpp
// 简化：std::string 的移动构造函数
string(string&& other) noexcept
    : data_(other.data_), size_(other.size_), capacity_(other.capacity_) {
    other.data_ = nullptr;   // 关键：把源对象置为可安全析构的空状态
    other.size_ = 0;
    other.capacity_ = 0;
}
```

指针类型（`unique_ptr`、`shared_ptr`）的移动也只是拷贝指针值然后置空，常数时间。

## 完美转发：模板里用 std::forward

问题：模板参数 `T&& x` 可以接受左值和右值，但 `x` 本身是具名变量（左值表达式），直接传给内层函数会丢掉右值信息。

```cpp
template <typename T>
void wrapper(T&& x) {
    target(x);  // 错误：x 是左值，始终调 target 的左值重载
}
```

解决：`std::forward<T>(x)` 保留原始的左右值信息。

```cpp
template <typename T>
void wrapper(T&& x) {
    target(std::forward<T>(x));  // 正确：左值传左值，右值传右值
}
```

### 引用折叠规则

`T&&` 的推导遵循引用折叠：

| 调用实参 | `T` 推导为 | `T&&` 折叠为 |
|---------|-----------|-------------|
| 左值 `int a` | `int&` | `int& &&` → `int&` |
| 右值 `5` | `int` | `int&&` |

为什么 `std::forward<T>` 能区分？

```cpp
// std::forward 的简化实现
template <typename T>
T&& forward(std::remove_reference_t<T>& t) noexcept {
    return static_cast<T&&>(t);
}
// 若 T = int：  T&& = int&&， 右值转发
// 若 T = int&： T&& = int& && = int&（引用折叠），左值转发
```

## 常见错误

**1. 对 const 对象 std::move 无效**
```cpp
const std::string s = "hello";
std::string t = std::move(s);  // 仍然是拷贝！const 对象不能移动（移动构造参数不是 const T&&）
```

**2. 移动后继续使用**
```cpp
std::string s = "hello";
std::string t = std::move(s);
std::cout << s.size();  // 合法但 s 处于未指定状态，通常是 0
std::cout << s;          // 合法但输出空字符串或垃圾信息
// s.clear()、s = "new" 都合法，能安全回用
// s[0] 危险——不保证有内容
```
规则：移动后对象处于"有效但未指定"状态。可以调用无前置条件的成员（赋值、clear、析构等），但不能假设其值。

**3. 误用 forward 做 move**
```cpp
// std::forward 用于转发，需要模板参数
// std::move 用于显式移动，不需要模板参数
std::forward<std::string>(x);  // 奇怪：forward 需要推导出的模板参数才有意义
std::move(x);                   // 简单直接
```

**4. 对 return 的局部变量写 std::move**
```cpp
// 错误——妨碍 NRVO
std::string make_string() {
    std::string s = "hello";
    return std::move(s);  // 禁止用 std::move 包装 return 的局部变量
    // 正确：return s;  // 编译器自动执行 NRVO 或隐式移动
}
```
C++ 标准规定 return 一个局部变量时，编译器首先尝试把它当成右值（隐式移动）。显式 `std::move` 反而可能抑制 NRVO（Named Return Value Optimization）。

**例外**：return 的不是局部变量而是成员变量/参数/全局变量时，需要 `std::move`，因为隐式移动规则只对局部变量和 `std::move` 的参数生效。

```cpp
std::string Widget::getName() const {
    return std::move(name_);  // 需要：成员变量不会被隐式移动
}
```

## 面试速记

- **`std::move` 做什么？** 无条件 cast 成右值引用。不移动数据。移动靠移动构造/赋值。
- **为什么具名 `T&&` 是左值？** 值类别 ≠ 类型。有名字 → 可以反复使用 → 左值。安全设计，防隐式重复移动。
- **`std::forward` 和 `std::move` 区别？** move 无条件转右值；forward 条件转发——左值留左值，右值转右值。
- **什么时候用哪个？** move 用于你"确定"要放弃变量的时候；forward 用于模板转发，不确定外面传的是左值还是右值。
- **移动后对象状态？** 有效但未指定。可安全赋值/析构，不应读值。
- **return 能加 std::move 吗？** 局部变量不加（妨 NRVO），成员变量要加。

## Open Questions

### Q1：为什么 C++ 没有"破坏性移动"？

Rust 的 move 是 destructive move——被移动的变量从作用域中消失，不可再使用，不需要析构。C++ 的 move 是 non-destructive——源对象仍然存活，析构函数仍然会执行。

后果：
- `unique_ptr` 移动后必须存 `nullptr`（否则析构函数会 double-free）
- `string` 移动后必须处于"可安全析构"的状态（空 buffer 或独立小 buffer）
- 移动构造函数必须额外写清理逻辑置空源对象

根本原因是 C++ 的 RAII + 自动析构模型：每个对象在作用域结束时无条件析构。移动只能"转移内容"，不能"阻止析构"。这是 C++ 和 Rust 在 ownership 模型上的根本分歧——C++ 选择向后兼容 C 的自动存储期，Rust 选择编译期所有权追踪。

### Q2：为什么 move constructor 需要 `noexcept`？不写会怎样？

直接影响 `std::vector` 的 reallocation 策略。

```cpp
std::vector<std::string> v;
v.reserve(10);
// 当 vector 扩容时，它需要把旧元素搬到新内存。
//
// 若 string 的移动构造函数是 noexcept：
//   → 直接移动所有元素，O(n)，失败率 0
//
// 若 string 的移动构造函数没有 noexcept：
//   → 拷贝所有元素，O(n) + n 次堆分配
//   → 原因：strong exception guarantee
//     中间一个元素移动失败，已经移走的元素回不来了
//     拷贝不破坏源对象 → 失败了直接 drop 新内存即可
```

**物理过程**见上文 strong exception guarantee 的分析。

检验方法：
```cpp
static_assert(std::is_nothrow_move_constructible_v<std::string>);  // 通常通过
static_assert(std::is_nothrow_move_constructible_v<std::list<int>>); // 取决于实现
static_assert(std::is_nothrow_move_constructible_v<MyType>);        // 检查自己的类型
```

**move constructor 可以不声明为 noexcept 吗？**

语言层面完全可以，编译器不强制。但后果因容器而异：

| 容器 | 影响 |
|------|------|
| `vector` 扩容 | 退化为拷贝，每次扩容 n 次堆分配 |
| `vector::resize` | 同上 |
| `deque` 扩容 | 同上 |
| `std::swap` | 通过 `move_if_noexcept` 走分支 |
| `list` / `map` | 不受影响（节点容器不搬迁元素） |

**什么时候不写 noexcept 是正确的？** 移动构造确实可能抛异常时：

```cpp
struct LoggingBuffer {
    std::unique_ptr<char[]> data;
    LoggingBuffer(LoggingBuffer&& other) : data(std::move(other.data)) {
        logger.write("buffer moved");  // 日志写入可能抛异常
    }
    // 不加 noexcept 是诚实的——告诉容器"我的 move 不可靠，请用拷贝"
};
```

**什么时候不写 noexcept 是错误的？** 不可能抛异常但忘了标：

```cpp
struct MyType {
    std::unique_ptr<Impl> p;
    std::vector<int> data;

    MyType(MyType&& other)  // ← 忘了 noexcept！
        : p(std::move(other.p)), data(std::move(other.data)) {}
    // 两个成员都是 noexcept-movable，自身一定不抛
};
// 后果：vector<MyType> 扩容走拷贝，每次拷整个 Impl 对象
```

**判断标准**：

```
手写 move constructor 时：
├─ 所有成员都是 noexcept-movable → 加 noexcept（没理由不加）
├─ 内部有操作可能抛 → 不加，这是正确的
└─ 用 = default → 编译器自动推导，不用管
```

经验法则：**要么 noexcept，要么别手写交给 `= default`**。没有 `noexcept` 的 move 会静默降级为拷贝，是最隐蔽的性能退化之一。

### Q3：什么时候 move 不是优化？

对于**不含堆资源的类型**，move 就是 copy。没有堆 buffer 可偷，移动构造函数做的事就是逐成员拷贝。

```cpp
struct Point { float x, y; };       // move = copy，全是标量
std::array<int, 100> a;             // move = copy，栈上数据必须逐字节拷
std::string s;                       // move ≠ copy，s 有堆 buffer
```

更隐蔽的场景：
```cpp
std::string s = "hello";
std::string t = std::move(s);  // 优化：偷 buffer
// 但如果 "hello" 用 SSO（短字符串优化），buffer 根本不在堆上
// 此时 move 和 copy 开销几乎一样——都只是拷贝 ~15 字节的栈上数组
```

SSO 下小字符串的 move 不会更快。真正受益于 move 的是超出 SSO 阈值（通常 15~22 字符）的字符串，或含 `unique_ptr`/`vector`/`map` 成员的类型。

### Q4：泛型代码中 `std::forward<T>(x)` 和 `std::move(x)` 会搞混怎么办？

```cpp
template <typename T>
void sink(T&& x) {
    // 错误：std::move(x) 无条件转右值 → 左值参数被意外掏空
    vec.push_back(std::move(x));

    // 正确：保留外层的值类别意图
    vec.push_back(std::forward<T>(x));
}
```

但反过来，如果确实要消费 `x`（比如智能指针接管所有权），即使 x 是左值也应该 move。判断标准：**看语义——你是要转发（forward）还是要终结变量（move）**。forward 保留调用者的选择权，move 强制消费。

## Connections

- [[static-cast]]: `std::move` 和 `std::forward` 的底层实现都是 `static_cast`
- [[rtti]]: 运行时类型信息（另一个维度的 type 机制），move/forward 是编译期的值类别操作
