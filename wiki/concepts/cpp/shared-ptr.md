---
title: "shared_ptr 实现"
type: concept
created: 2026-05-29
updated: 2026-05-29
sources: ["raw/interview-problems/手撕题目：实现C++ shared_ptr.md"]
questions: [1]
tags: [c++, smart-pointer, interview, memory, concurrency]
---

## 题目要求

实现简化版 `SharedPtr<T>`：

1. 默认构造、裸指针构造
2. 拷贝构造、拷贝赋值
3. 移动构造、移动赋值
4. 析构自动释放
5. `operator*`、`operator->`、`get()`、`use_count()`
6. 引用计数线程安全（`std::atomic`）
7. 说明 weak_ptr 扩展方案

## 控制块设计

引用计数不能存在 `SharedPtr` 对象内部——多个 `SharedPtr` 共享同一个计数，必须放在堆上独立的**控制块**中。

```cpp
struct ControlBlock {
    std::atomic<size_t> strong_count{1};  // 强引用计数，构造时 = 1
    std::atomic<size_t> weak_count{0};    // 弱引用计数（为 weak_ptr 预留）

    void add_strong() { strong_count.fetch_add(1, std::memory_order_relaxed); }
    bool release_strong() {
        // 减 1，返回减之前的值。若减前 = 1 → 减后 = 0 → 需要释放对象
        return strong_count.fetch_sub(1, std::memory_order_acq_rel) == 1;
    }
    void add_weak()  { weak_count.fetch_add(1, std::memory_order_relaxed); }
    bool release_weak() {
        return weak_count.fetch_sub(1, std::memory_order_acq_rel) == 1;
    }
};
```

**为什么控制块要独立分配？** 因为多个 `SharedPtr` 和 `WeakPtr` 都指向同一个控制块。如果控制块内嵌在 `SharedPtr` 中，拷贝后就各自为政了。

**为什么用 `memory_order_acq_rel` 而不是默认的 `seq_cst`？** 对引用计数的增减，只需要 acquire-release 语义——保证被管理对象的析构在最后一个 `SharedPtr` 的所有写操作之后（acquire），且减操作前的所有写对析构可见（release）。`seq_cst` 更强但更慢。

## 完整实现

```cpp
#include <atomic>
#include <utility>  // std::exchange, std::swap

template <typename T>
class SharedPtr {
public:
    // ── 构造 ────────────────────────────

    SharedPtr() : ptr_(nullptr), ctrl_(nullptr) {}

    explicit SharedPtr(T* raw) : ptr_(raw) {
        if (raw) {
            ctrl_ = new ControlBlock();  // 强引用计数初始 = 1
        }
    }

    // ── 拷贝 ────────────────────────────

    SharedPtr(const SharedPtr& other) : ptr_(other.ptr_), ctrl_(other.ctrl_) {
        if (ctrl_) ctrl_->add_strong();
    }

    SharedPtr& operator=(const SharedPtr& other) {
        if (this == &other) return *this;  // 自赋值检查
        release();                          // 先释放当前资源
        ptr_  = other.ptr_;
        ctrl_ = other.ctrl_;
        if (ctrl_) ctrl_->add_strong();    // 再增加新引用
        return *this;
    }

    // ── 移动 ────────────────────────────

    SharedPtr(SharedPtr&& other) noexcept
        : ptr_(std::exchange(other.ptr_, nullptr))
        , ctrl_(std::exchange(other.ctrl_, nullptr)) {
        // 不增加引用计数——只是转移所有权
    }

    SharedPtr& operator=(SharedPtr&& other) noexcept {
        if (this == &other) return *this;
        release();
        ptr_  = std::exchange(other.ptr_, nullptr);
        ctrl_ = std::exchange(other.ctrl_, nullptr);
        return *this;
    }

    // ── 析构 ────────────────────────────

    ~SharedPtr() { release(); }

    // ── 访问器 ──────────────────────────

    T& operator*()  const { return *ptr_; }
    T* operator->() const { return ptr_; }
    T* get()        const { return ptr_; }

    size_t use_count() const {
        return ctrl_ ? ctrl_->strong_count.load(std::memory_order_relaxed) : 0;
    }

    explicit operator bool() const { return ptr_ != nullptr; }

private:
    T* ptr_ = nullptr;
    ControlBlock* ctrl_ = nullptr;

    void release() {
        if (!ctrl_) return;
        if (ctrl_->release_strong()) {
            // 强引用计数归零 → 释放被管理对象
            delete ptr_;
            ptr_ = nullptr;
            if (ctrl_->release_weak()) {
                // 弱引用也归零 → 释放控制块
                delete ctrl_;
            }
        }
        ctrl_ = nullptr;
    }
};
```

### 拷贝赋值的顺序不能反过来

```cpp
// 错误顺序：
SharedPtr& operator=(const SharedPtr& other) {
    if (ctrl_) ctrl_->add_strong();  // 如果 other 也是 *this 且是自己的唯一引用
    release();                        // release 可能已经把控制块 delete 了
    // ...
}

// 正确顺序：先 release 当前资源，再 add_strong 新资源
// 原因：other 可能和 *this 共享同一个控制块
// 先 add 再 release 的话，如果 use_count=1，release 会 delete 控制块
```

自赋值检查 `if (this == &other) return *this;` 不仅为避免无意义工作——没有它，`use_count=1` 的自赋值会先 `release`（delete 控制块），然后访问已释放的 `other.ctrl_->add_strong()`，**use-after-free**。

## 线程安全：什么安全，什么不安全

标准 `std::shared_ptr` 的线程安全保证是**引用计数线程安全，被管理对象不保证**：

```cpp
SharedPtr<Widget> sp(new Widget);

// 安全——多个线程同时拷贝/析构同一个 sp（操作的是引用计数，atomic）
std::thread t1([sp]{});  // 拷贝：inc refcount
std::thread t2([sp]{});  // 拷贝：inc refcount
// 两个线程同时修改 ctrl_->strong_count——atomic，安全

// 不安全——多线程同时通过 shared_ptr 写同一对象
SharedPtr<Widget> sp(new Widget);
std::thread t1([&sp]{ sp->value = 1; });  // 写 Widget::value
std::thread t2([&sp]{ sp->value = 2; });  // 写 Widget::value
// Widget::value 不是 atomic——data race

// 不安全——多线程同时读写 shared_ptr 变量本身
std::thread t1([&sp]{ sp = other_sp; });  // 修改 sp 本身（不是 atomic）
std::thread t2([&sp]{ sp = other_sp2; }); // 同时修改 sp 本身——data race
```

记法：**`shared_ptr` 的引用计数是线程安全的，`shared_ptr` 变量本身不是。**

## weak_ptr 扩展

### 为什么需要 weak_ptr

```cpp
// 循环引用问题
struct Node {
    SharedPtr<Node> next;  // 强引用 → 永远不会释放
    ~Node() { std::cout << "freed\n"; }
};

auto a = new SharedPtr<Node>(new Node);
auto b = new SharedPtr<Node>(new Node);
(*a)->next = *b;
(*b)->next = *a;
// a 和 b 出作用域 → use_count 从 2 降到 1（互相持有）
// 两个 Node 都泄露——永远不会析构
```

### 解决方案：weak_ptr 不增加强引用

```cpp
struct Node {
    WeakPtr<Node> next;  // 弱引用 → 打破循环
};
```

### 控制块扩展

```
                    ┌─────────────┐
                    │ ControlBlock │
                    ├─────────────┤
 SharedPtr ────────→│ strong_count │  ← 强引用计数（决定对象何时 delete）
 SharedPtr ────────→│             │
                    │ weak_count  │  ← 弱引用计数（决定控制块何时 delete）
 WeakPtr ──────────→│             │
 WeakPtr ──────────→│             │
                    └──────┬──────┘
                           │
                           ▼ 强引用 = 0 时 delete
                    ┌──────────┐
                    │ T object │
                    └──────────┘
```

释放规则：
- `strong_count` 归零 → `delete` 被管理对象，但**控制块继续存活**（WeakPtr 还在用）
- `weak_count` 也归零 → `delete` 控制块

### WeakPtr 实现

```cpp
template <typename T>
class WeakPtr {
public:
    WeakPtr() = default;

    // 从 SharedPtr 构造：不增加强引用，只增加弱引用
    WeakPtr(const SharedPtr<T>& sp)
        : ptr_(sp.ptr_), ctrl_(sp.ctrl_) {
        if (ctrl_) ctrl_->add_weak();
    }

    WeakPtr(const WeakPtr& other)
        : ptr_(other.ptr_), ctrl_(other.ctrl_) {
        if (ctrl_) ctrl_->add_weak();
    }

    WeakPtr& operator=(const WeakPtr& other) {
        if (this == &other) return *this;
        release();
        ptr_  = other.ptr_;
        ctrl_ = other.ctrl_;
        if (ctrl_) ctrl_->add_weak();
        return *this;
    }

    ~WeakPtr() { release(); }

    // lock() — 尝试提升为 SharedPtr
    SharedPtr<T> lock() const {
        if (!ctrl_) return SharedPtr<T>();
        // CAS 自旋：只在 strong_count > 0 时才能提升
        size_t count = ctrl_->strong_count.load(std::memory_order_relaxed);
        do {
            if (count == 0) return SharedPtr<T>();  // 对象已释放
        } while (!ctrl_->strong_count.compare_exchange_weak(
            count, count + 1, std::memory_order_acquire, std::memory_order_relaxed));
        // CAS 成功——引用计数已增加，返回持有该引用的 SharedPtr
        return SharedPtr<T>(ptr_, ctrl_);  // 需要 friend 访问私有构造
    }

    bool expired() const {
        return use_count() == 0;
    }

    size_t use_count() const {
        return ctrl_ ? ctrl_->strong_count.load(std::memory_order_relaxed) : 0;
    }

private:
    T* ptr_ = nullptr;
    ControlBlock* ctrl_ = nullptr;

    void release() {
        if (!ctrl_) return;
        if (ctrl_->release_weak()) {
            // weak_count 归零并且 strong_count 也已归零 → 释放控制块
            delete ctrl_;
        }
        ctrl_ = nullptr;
    }
};
```

`lock()` 是最关键的方法——用 CAS 循环安全地将弱引用"升级"为强引用，同时防止 TOCTOU 问题：在检查 `count > 0` 和增加计数之间，最后一个 `SharedPtr` 可能正好析构。

## enable_shared_from_this

### 问题：为什么不能直接 `SharedPtr<T>(this)`？

这是最高频的 shared_ptr 使用错误之一。假设一个对象需要在成员函数中返回指向自己的 shared_ptr：

```cpp
class Widget {
public:
    SharedPtr<Widget> getPtr() {
        return SharedPtr<Widget>(this);  // 致命错误！
    }
};

SharedPtr<Widget> sp(new Widget);
SharedPtr<Widget> sp2 = sp->getPtr();
// sp 和 sp2 各自认为自己是 Widget 的唯一 owner
// sp 析构 → delete Widget
// sp2 析构 → delete Widget → double free
```

本质原因是 `SharedPtr<T>(this)` **每次调用都创建新的控制块**。两个 SharedPtr 共享同一个对象指针但各自维护独立的控制块，各自数自己的引用计数到 0，各自 delete 一次对象。

**唯一正确的方式**：第一个 SharedPtr 创建控制块后，所有后续的 SharedPtr 共享同一个控制块。`enable_shared_from_this` 就是为此设计的——它让对象内部保存一个指向"唯一控制块"的弱引用。

### 正确用法

```cpp
class Widget : public std::enable_shared_from_this<Widget> {
public:
    SharedPtr<Widget> getPtr() {
        return shared_from_this();  // 共享已有控制块，safe
    }

    ~Widget() { std::cout << "dtor\n"; }
};

auto sp = std::make_shared<Widget>();
auto sp2 = sp->getPtr();
// sp.use_count() == 2 — 共享同一个控制块
// 两个 shared_ptr 都析构后 Widget 被释放一次
```

### 内部实现原理

`enable_shared_from_this<T>` 内部持有一个 `weak_ptr<T>`：

```cpp
template <typename T>
class enable_shared_from_this {
public:
    SharedPtr<T> shared_from_this() {
        return weak_this_.lock();  // weak_ptr → shared_ptr
    }

    // 只观测不提升
    WeakPtr<T> weak_from_this() const noexcept {
        return weak_this_;
    }

protected:
    // CRTP: Derived 继承 enable_shared_from_this<Derived>
    // 构造函数什么也不做——weak_this_ 此时为空
    enable_shared_from_this() noexcept = default;
    ~enable_shared_from_this() = default;

    // 禁止拷贝——不同对象的 weak_this_ 不能混淆
    enable_shared_from_this(const enable_shared_from_this&) noexcept {}
    enable_shared_from_this& operator=(const enable_shared_from_this&) noexcept {
        return *this;
    }
    // 拷贝构造和赋值故意不复制 weak_this_——新对象属于新的 shared_ptr

private:
    template <typename U>
    friend class SharedPtr;  // 让 SharedPtr 能直接写 weak_this_

    mutable WeakPtr<T> weak_this_;  // mutable: 在 shared_ptr 构造函数里赋初值
};
```

关键设计细节：

1. **拷贝构造/赋值不复制 `weak_this_`**：拷贝出来的新对象不属于已有的 shared_ptr，需要新的 shared_ptr 来初始化它的 `weak_this_`。如果复制了，新对象会错误地认为自己也关联到旧对象的控制块。

2. **`weak_ptr<T>` 而非 `shared_ptr<T>`**：如果用 `shared_ptr`，会永久增加一个强引用，即使外部所有 shared_ptr 都析构了对象也不会释放——这就是循环引用。`weak_ptr` 正确地反映了"对象自身不应该拥有自己"。

3. **`mutable`**：`shared_ptr` 构造函数需要修改 `weak_this_`，但 `shared_ptr` 的构造函数接收 `T*`，不要求 `T` 是可修改的。

### shared_ptr 构造函数的"检测"机制

`shared_ptr` 的构造函数如何知道 `T` 继承了 `enable_shared_from_this<T>`？通过编译期的 `is_convertible` 检测 + SFINAE：

```cpp
// 简化：shared_ptr 某个构造函数的逻辑
template <typename T>
class SharedPtr {
    // 从裸指针构造时，检测 enable_shared_from_this
    template <typename Y>
    explicit SharedPtr(Y* ptr) : ptr_(ptr), ctrl_(new ControlBlock()) {
        // 编译期检测：Y 是否可转为 enable_shared_from_this<Y> 的基类？
        if constexpr (std::is_convertible_v<Y*, enable_shared_from_this<Y>*>) {
            // 是 → 初始化 weak_this_
            enable_shared_from_this<Y>& base = *ptr;
            if (base.weak_this_.expired()) {
                // 从当前 shared_ptr 建一个 weak_ptr 存进去
                base.weak_this_ = /* 从 *this 构造 weak_ptr */;
            }
        }
    }
};
```

实际标准库中，这个初始化逻辑是在 `shared_ptr` 内部的 `_Enable_shared_from_this` 帮助函数中完成的，但核心思想一样：编译期检测继承关系，运行时给 `weak_this_` 赋值。

### 初始化时机：weak_this_ 何时被赋值

```
Widget* w = new Widget;
// w->weak_this_ 为空（expired）
// shared_from_this() 会抛 bad_weak_ptr 或导致 UB

SharedPtr<Widget> sp(w);
// shared_ptr 构造函数检测到 Widget 继承 enable_shared_from_this
// → 从 sp 构造一个 weak_ptr 赋值给 w->weak_this_
// w->weak_this_.lock() 现在返回 sp 的 shared_ptr 副本

auto sp2 = w->shared_from_this();  // OK，返回 sp 的副本
// sp.use_count() == 2
```

**共享控制块的关键**：`shared_ptr` 构造函数把"自己持有的控制块信息"包装成 weak_ptr 存进对象内部。此后对象内部的 `weak_this_` 和外部所有 shared_ptr 指向**同一个控制块**。

### 陷阱与常见错误

#### 1. 没有 shared_ptr 就直接调用 shared_from_this()

```cpp
class Widget : public std::enable_shared_from_this<Widget> {
public:
    void doSomething() {
        auto sp = shared_from_this();  // 未定义行为！
        // weak_this_ 从未被初始化（没有 shared_ptr 拥有过这个对象）
    }
};

Widget w;               // 栈上对象，没有 shared_ptr 拥有它
w.doSomething();        // UB！标准库实现通常抛 std::bad_weak_ptr
```

这是最常见的错误——`shared_from_this()` 的前提是**对象已经由某个 shared_ptr 管理**。换句话说，必须先有 `SharedPtr<Widget> sp(new Widget)`，只有这之后 `share_from_this()` 才合法。

**最佳实践**：将构造函数设为 private，只允许通过静态工厂方法创建，确保对象永远在 shared_ptr 中：

```cpp
class Widget : public std::enable_shared_from_this<Widget> {
public:
    static SharedPtr<Widget> create() {
        return SharedPtr<Widget>(new Widget());  // C++17 前用这个
        // 或 return std::make_shared<Widget>();   // C++17 起 make_shared 也行
    }

    SharedPtr<Widget> getPtr() {
        return shared_from_this();
    }

private:
    Widget() = default;  // 禁止外部直接构造
};
```

#### 2. 构造函数/析构函数中调用 shared_from_this()

```cpp
class Widget : public std::enable_shared_from_this<Widget> {
public:
    Widget() {
        auto sp = shared_from_this();  // UB! weak_this_ 还未初始化
    }
    ~Widget() {
        auto sp = shared_from_this();  // UB! shared_ptr 正在析构，use_count 已是 0
    }
};
```

构造函数中 `weak_this_` 尚未被 shared_ptr 初始化（`shared_ptr` 的构造函数先执行 `T` 的构造，再初始化 `weak_this_`）。析构函数中 `shared_ptr` 的析构已将强引用计数减到 0，`weak_this_.lock()` 返回空。

#### 3. 多重继承的 enable_shared_from_this

```cpp
class Base : public std::enable_shared_from_this<Base> {};
class Derived : public Base {};

Derived* d = new Derived;
SharedPtr<Derived> sp(d);
// shared_ptr 构造时会检测 Base::enable_shared_from_this<Base>
// Base 的 weak_this_ 被正确初始化 ✓

d->shared_from_this();  // 返回 SharedPtr<Base>，不是 SharedPtr<Derived>
```

如果两个基类都继承 `enable_shared_from_this`，只有一个能生效——两个独立的 `weak_this_` 指向同一个控制块会导致混乱。标准库规定了这种情形下的行为（最后一个被检测到的有效），但实际使用中应避免。

#### 4. shared_from_this() 返回的是 shared_ptr，不是裸指针引用

```cpp
void someFunction(const SharedPtr<Widget>& sp) { ... }

// 错误：
someFunction(shared_from_this());  // 创建临时 shared_ptr，函数返回后析构
// 这不是永久的引用——函数返回后 shared_ptr 计数减 1

// 正确——只在需要传递所有权或延长生命周期时用 shared_from_this
SharedPtr<Widget> sp = shared_from_this();  // 显式绑定延长生命周期
```

### 与 make_shared 的交互

```cpp
// C++17 起，make_shared 和 enable_shared_from_this 完全兼容
auto sp = std::make_shared<Widget>();
sp->shared_from_this();  // OK

// C++14 及之前：make_shared 对 enable_shared_from_this 的支持因实现而异
// 部分旧标准库中 make_shared 不会检测 enable_shared_from_this
// 安全做法：C++17 前用 shared_ptr<T>(new T) 而非 make_shared
```

### 面试速记

- **为什么不能 `SharedPtr<T>(this)`？** 每次调用创建新控制块 → double free。需要共享第一个 shared_ptr 的控制块。
- **内部存什么？** 存一个 `mutable weak_ptr<T>`。用 weak_ptr 是为了不额外增加强引用计数（不然对象永远不会释放）。
- **什么时候赋值？** `shared_ptr` 构造时，通过 `is_convertible` 编译期检测继承关系，runtime 把 weak_ptr 写进 `weak_this_`。
- **什么时候能用 `shared_from_this()`？** 对象已被 shared_ptr 管理之后。否则 UB。
- **构造函数/析构函数里能用吗？** 不能。构造时 weak_this_ 未初始化，析构时已过期。

## make_shared 的单次分配优化

```cpp
auto sp = std::make_shared<Widget>(args...);
```

`make_shared` 一次分配同时容纳控制块和对象：

```
普通构造（两次堆分配）：
  new Widget     → Widget object
  new ControlBlock → ControlBlock

make_shared（一次堆分配）：
  new char[sizeof(ControlBlock) + sizeof(Widget)]
  → [ControlBlock | Widget object]  ← 同一个内存块
```

好处：少一次堆分配，更好的缓存局部性。代价：`weak_count` 未归零前整个内存块不能释放——即使对象已析构，其占用的内存仍在（被控制块"锚定"）。

## 常见面试追问

### Q1：为什么控制块需要用 `new` 而不能内嵌在 SharedPtr 里？

内嵌的控制块在拷贝时会复制，导致多个 SharedPtr 各自维护独立的计数，无法共享。控制块必须是一个堆上对象，让所有 SharedPtr 指向同一个。

### Q2：移动构造为什么不改引用计数？

移动语义的本质是**所有权转移**，不是共享。移动后源 `SharedPtr` 放弃所有权（ptr_ 和 ctrl_ 置空），目标接过所有权。总引用数不变，无需操作原子变量。

### Q3：拷贝赋值先 release 再 add 和先 add 再 release 有什么区别？

考虑 `sp1 = sp2` 这个拷贝赋值操作。目标是把 sp2 的 ptr_ 和 ctrl_ 复制到 sp1，同时维护引用计数。两种实现方案：

**方案 A：「先 release 再 add」**

```cpp
SharedPtr& operator=(const SharedPtr& other) {
    if (this == &other) return *this;  // 必须检查自赋值！
    release();                          // ① 先放弃当前资源
    ptr_  = other.ptr_;
    ctrl_ = other.ctrl_;
    if (ctrl_) ctrl_->add_strong();    // ② 再增加新引用
    return *this;
}
```

**方案 B：「先 add 再 release」**

```cpp
SharedPtr& operator=(const SharedPtr& other) {
    if (other.ctrl_) other.ctrl_->add_strong();  // ① 先增加新引用
    release();                                     // ② 再放弃旧资源
    ptr_  = other.ptr_;
    ctrl_ = other.ctrl_;
    return *this;
    // 即使 *this == other 也正确，不需要自赋值检查
}
```

为什么这两种写法有区别？关键在**自赋值的边界情况**。

**场景**：`sp1 = sp1`（自赋值）或 `sp1 = sp2`，且 sp1 和 sp2 指向同一对象，`use_count == 1`。

**方案 A 的执行过程**（没有自赋值检查的话）：

```
初始: sp1.ctrl_->strong_count = 1

① release():
   ctrl_->release_strong() → strong_count 从 1 减到 0
   → "减之前是 1，归零了，释放对象"
   → delete ptr_;        // 对象被销毁
   → delete ctrl_;        // 控制块被销毁！

② other.ctrl_->add_strong():
   // other 就是 *this（自赋值），other.ctrl_ 就是刚才 delete 的那个地址
   // 访问已释放内存 → use-after-free
```

这就是为什么方案 A **必须**在开头做 `if (this == &other) return *this;`。自赋值检查不仅仅是"避免无意义工作"的优化——没有它，`use_count == 1` 的自赋值会直接访问已释放的内存。

**方案 B 的执行过程**（同样场景）：

```
初始:  both.ctrl_->strong_count = 1

① add_strong():
   strong_count 从 1 增到 2

② release():
   release_strong() → strong_count 从 2 减到 1
   → "减之前是 2，没归零，不释放"

最终: strong_count = 1，对象和控制块都完好
```

方案 B 天然免疫自赋值问题——因为**先增后减，减之前计数至少是 1 + 1 = 2**，永远不会在这个调用路径中归零。

**代价对比**：

| | 方案 A（先 release） | 方案 B（先 add） |
|---|---|---|
| 自赋值检查 | 必须 | 不需要 |
| 自赋值的 atomic 操作 | 0 次（直接 return） | 2 次（add + release） |
| 非自赋值的 atomic 操作 | 2 次（release + add） | 2 次（add + release） |
| 安全性 | 依赖 `this == &other` 检查的正确性 | 天然安全 |

方案 A 在自赋值上更快（0 次原子操作），但正确性依赖自赋值检查。方案 B 多做了一次无意义的 add+release，但不需要自赋值检查就能正确工作。标准库实现通常用方案 B 或 copy-and-swap 惯用法（间接等价于方案 B）。

### Q4：什么时候 weak_count 不归零？

如果存在 `weak_ptr` 指向已过期对象，`strong_count==0` 但 `weak_count>0` → 控制块继续存活，对象的内存（如果是 `make_shared` 分配的）也继续占据。大量 `weak_ptr` 长期不销毁可能造成"逻辑上的内存泄漏"。

### Q5：shared_ptr 是零开销的吗？

引用计数的 atomic 操作有开销，但通常可接受。真正可见的开销在：
- 两次堆分配（除非用 `make_shared`）
- 每次拷贝/析构的 atomic inc/dec（对高度竞争的场景可能成为瓶颈）
- 控制块的内存开销（两个 `atomic<size_t>` + deleter + allocator，通常 24-32 字节）

## Connections

- [[move-semantics]]: 移动构造/赋值实现依赖 `std::exchange` 转移所有权
- [[stl-unordered-map]]: 容器中存储 shared_ptr 的线程安全考量
- [[atomic]]: 引用计数的底层机制

## Raw Source

- [[raw/interview-problems/手撕题目：实现C++ shared_ptr.md]]
