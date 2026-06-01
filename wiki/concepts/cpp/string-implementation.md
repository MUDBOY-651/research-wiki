---
title: "String 类实现"
type: concept
created: 2026-06-01
updated: 2026-06-01
sources: ["raw/interview-problems/手撕题目：实现类似C++ string类.md"]
questions: [1]
tags: [c++, string, interview, memory, raii, exception-safety]
---

## 题目要求

实现简化版 `String` 类：
1. 默认构造、`const char*` 构造
2. 拷贝构造、拷贝赋值
3. 移动构造、移动赋值（Rule of Five）
4. 析构
5. `size()`、`capacity()`、`c_str()`、`operator[]`
6. `push_back()`、`append()`
7. 保证 `'\0'` 结尾
8. 异常安全 + 自赋值

## 内部表示

```cpp
class String {
private:
    char*   data_ = nullptr;   // 指向堆上的字符串，以 '\0' 结尾
    size_t  size_ = 0;         // 有效字符数（不含 '\0'）
    size_t  capacity_ = 0;     // 可容纳有效字符数（不含 '\0'）

    // 不变式（invariant）：
    // 1. data_[size_] == '\0' 始终成立
    // 2. size_ <= capacity_
    // 3. capacity_ 为 0 时 data_ 为 nullptr（或指向空字符串，取决于实现）
};
```

**为什么 capacity_ 不含 `'\0'`？** `size_` 和 `capacity_` 语义一致——都用"有效字符数"计量。分配时 `new char[capacity_ + 1]`，多出的 1 留给 `'\0'`。这避免了到处写 `+1` `-1` 的混乱。

**关键区分**：
- `size()` 返回字符数
- `capacity()` 返回可存储的字符数上限
- c_str 始终有 `'\0'`：`data_[size()]` 一定是 `'\0'`

## 完整实现

### 构造与析构

```cpp
class String {
    char*  data_;
    size_t size_;
    size_t capacity_;

public:
    // 默认构造
    String() : data_(nullptr), size_(0), capacity_(0) {}

    // const char* 构造
    String(const char* s) {
        if (s == nullptr) {
            data_ = nullptr;
            size_ = 0;
            capacity_ = 0;
            return;
        }
        size_ = std::strlen(s);
        capacity_ = size_;
        data_ = new char[capacity_ + 1];
        std::memcpy(data_, s, size_ + 1);  // +1 拷贝 '\0'
    }

    // 析构
    ~String() {
        delete[] data_;
    }
};
```

### 拷贝（深拷贝）

```cpp
    // 拷贝构造
    String(const String& other) {
        if (other.data_ == nullptr) {
            data_ = nullptr;
            size_ = 0;
            capacity_ = 0;
            return;
        }
        size_ = other.size_;
        capacity_ = other.capacity_;
        data_ = new char[capacity_ + 1];
        std::memcpy(data_, other.data_, size_ + 1);
    }

    // 拷贝赋值 — copy-and-swap（异常安全 + 自赋值安全）
    String& operator=(const String& other) {
        if (this == &other) return *this;      // 自赋值快速路径
        String tmp(other);                      // 拷贝构造（可能抛异常）
        swap(tmp);                               // 不抛异常，接管资源
        return *this;
    }  // tmp 析构，释放旧资源
```

**为什么 copy-and-swap？** 先拷贝到 `tmp`——如果拷贝中途抛异常（`new` 失败），`*this` 保持不变（强异常安全）。swap 只交换三个指针/整数，不抛异常。`tmp` 析构时自然释放旧资源。

### 移动

```cpp
    // 移动构造
    String(String&& other) noexcept
        : data_(std::exchange(other.data_, nullptr))
        , size_(std::exchange(other.size_, 0))
        , capacity_(std::exchange(other.capacity_, 0))
    {}

    // 移动赋值
    String& operator=(String&& other) noexcept {
        if (this == &other) return *this;
        delete[] data_;
        data_ = std::exchange(other.data_, nullptr);
        size_ = std::exchange(other.size_, 0);
        capacity_ = std::exchange(other.capacity_, 0);
        return *this;
    }
```

移动后的源对象处于"有效但空"的状态——`data_` 为 `nullptr`，`size_` 和 `capacity_` 为 0。析构时 `delete[] nullptr` 是安全的。

### 访问器

```cpp
    size_t size()     const { return size_; }
    size_t capacity() const { return capacity_; }
    bool   empty()    const { return size_ == 0; }

    const char* c_str() const {
        // 空字符串时返回指向空 C 字符串的指针
        static const char empty_str[] = "";
        return data_ ? data_ : empty_str;
    }

    char& operator[](size_t i)       { return data_[i]; }
    const char& operator[](size_t i) const { return data_[i]; }
```

`c_str()` 对空字符串的处理很重要——不能返回 `nullptr`（C API 期望 `const char*` 永远非空）。返回指向静态空字符串的指针或让 `data_` 永远至少指向一个 `'\0'`。

### push_back 与 append

```cpp
    void push_back(char c) {
        if (size_ == capacity_) {
            // 扩容策略：倍增，最小 16
            size_t new_cap = capacity_ == 0 ? 16 : capacity_ * 2;
            reserve(new_cap);
        }
        data_[size_] = c;
        ++size_;
        data_[size_] = '\0';  // 维护不变式
    }

    void append(const char* s) {
        if (s == nullptr) return;
        size_t len = std::strlen(s);
        if (len == 0) return;
        if (size_ + len > capacity_) {
            size_t new_cap = std::max(capacity_ * 2, size_ + len);
            reserve(new_cap);
        }
        std::memcpy(data_ + size_, s, len);
        size_ += len;
        data_[size_] = '\0';
    }

    String& operator+=(char c) {
        push_back(c);
        return *this;
    }

    String& operator+=(const char* s) {
        append(s);
        return *this;
    }

    void reserve(size_t new_cap) {
        if (new_cap <= capacity_) return;
        char* new_data = new char[new_cap + 1];
        if (data_) {
            std::memcpy(new_data, data_, size_ + 1);
            delete[] data_;
        } else {
            new_data[0] = '\0';
        }
        data_ = new_data;
        capacity_ = new_cap;
    }
```

每次 `push_back` 和 `append` 结尾都写 `data_[size_] = '\0'`，维护"size_ 位置永远是 `'\0'`"的不变式。

### swap

```cpp
    void swap(String& other) noexcept {
        using std::swap;
        swap(data_, other.data_);
        swap(size_, other.size_);
        swap(capacity_, other.capacity_);
    }

    friend void swap(String& a, String& b) noexcept {
        a.swap(b);
    }
```

## 扩容策略分析

### 倍增（×2）vs 1.5 倍

```
倍增 ×2:  cap: 16 → 32 → 64 → 128 → 256 ...
         每一步都是新大小，之前释放的内存块总和永远 < 新块大小
         → 旧块和新块不可能合并 → 内存复用率差

×1.5:     cap: 16 → 24 → 36 → 54 → 81 → 121 ...
         前几个释放的块累加：16+24+36=76 > 54 ✓
         → 内存分配器有机会复用之前释放的块
```

标准库实际：GCC libstdc++ 用 ×2，Clang libc++ 用 ×2，MSVC 用 ×1.5。原因不是 ×1.5 内存更优，而是 ×2 配合 `malloc` 的伙伴分配器表现尚可，两者差异在微基准测试中不显著。

面试中：说自己选择的是 ×2，原因简单明了。如果追问可以说 1.5 倍的内存复用论证。

## 异常安全三层保证

```
nothrow 保证:   操作不可能抛异常。移动构造（标记 noexcept）、swap、析构。
强保证:         失败时状态完全回滚，就像没调用过。copy-and-swap 的拷贝赋值。
基本保证:       失败时不泄露资源，但状态可能改变。push_back 失败（new 抛异常）。
                ——数据没变但 capacity 可能增加（取决于 reserve 实现，实际上
                  capacity 也没有增加，因为 reserve 在修改 size 之前调用）
```

`push_back` 的异常安全分析：
```cpp
void push_back(char c) {
    if (size_ == capacity_) {
        reserve(new_cap);  // ① 扩容。失败抛异常 → 退出，size_ 没变。
    }                      //    reserve 内 new 成功则内存已换，失败则原样。
    data_[size_] = c;      // ② 写字符
    ++size_;               // ③ 更新 size_
    data_[size_] = '\0';   // ④ 写 '\0'
}
// ①在最后也可能失败，②③④不会抛异常（只是内存写入和整数运算）
// → 强异常安全（失败时无任何可观察的变化）
```

## 常见陷阱

### 1. off-by-one：容量差一的混乱

```cpp
// 错误：
data_ = new char[size_];       // 没有 '\0' 的空间
data_ = new char[capacity_];   // 没有 '\0' 的空间
data_[size_ + 1] = '\0';       // 下标越界

// 正确：
data_ = new char[capacity_ + 1]; // +1 给 '\0'
data_[size_] = '\0';             // size_ 位置正好是 '\0' 的位置
```

### 2. nullptr 输入

```cpp
// 危险：
String s = nullptr;   // ctor 中 strlen(nullptr) → crash

// 安全：
String(const char* s) {
    if (s == nullptr) { /* 初始化为空字符串 */ }
}
```

### 3. 自赋值

```cpp
// copy-and-swap 天然安全，但用下面的写法就有问题：
String& operator=(const String& other) {
    delete[] data_;             // 如果 this == &other，删了自己的数据
    data_ = new char[...];      // other.data_ 现在指向已释放的内存
    memcpy(data_, other.data_, ...);  // 拷贝已释放内存 → UB
}
// copy-and-swap 或自赋值检查两者取一即可
```

### 4. 缺少 noexcept 标记

移动构造和移动赋值的 `noexcept` 不是形式主义——`std::vector<String>` 扩容时，如果 `String` 的移动构造没有 `noexcept`，vector 会退化用拷贝。详情见 [[move-semantics]] Q2。

## SSO（Short String Optimization）

绝大多数程序中的字符串都很短。不开 SSO 时，即使 `std::string s = "hi"` 也要 HeapAlloc + memcpy。SSO 让小字符串直接存在 string 对象内部，零堆分配。

### 存储 layout

string 对象内部是一个 union——长模式存指针+size+capacity，短模式复用同一块内存存字符数组：

```
libc++ (Clang) — sizeof(string) = 24 字节，SSO 阈值 = 22 字符

长字符串模式（堆分配）:
  [ ptr* 8B ][ size 7B + flag 1B ][ capacity 7B + flag 1B ]
       │
       └──→ 堆上的 "hello world, this is a long string\0"

短字符串模式（SSO，≤22 字符）:
  [ "hello\0_________________"  ][ len 1B + flags ][ unused ]
    └───── 22 字节缓冲区 ──────┘
    全部在对象内部，无堆分配
```

三个主流实现的对比：

|          | sizeof  | SSO 阈值  |
| -------- | ------- | ------- |
| libc++   | 24 B    | 22 字符   |
| libstdc++| 32 B    | 15 字符   |
| MSVC     | 32 B    | 15 字符   |

GCC 的 string 虽然 32 字节，SSO 却只有 15 字符——因为 layout 不同（`char*` + `size_t` + 16 字节 SSO buffer）。

### SSO_BIT：怎么知道当前是长还是短？

核心 trick：`capacity_` 的真实值永远 < 2^63（8 EB），最高位一定是空闲的。借用这个 bit 做模式标记：

```
capacity_ 的 bit 63:
  0 → 短模式（对象内部存储）
  1 → 长模式（堆分配）
```

**短模式**下，对应 `capacity_` 那块内存存的是字符串数据。一个 uint8_t 存的长度值最高位自然为 0：

```cpp
// 构造 std::string s = "hi";

// 24 字节对象布局（libc++，简化）:
// [0..21]: 'h', 'i', '\0', ...  ← 22 字节字符 buffer
// [22]:    0x02                  ← 长度 = 2，uint8_t，最高位 = 0
// [23]:    (padding)
//
// 如果你"误读"这块内存为 Long 模式：
//   capacity_ 位置的最高 byte 是 0x02 → 最高 bit = 0 → 短模式！
```

**长模式**下，分配时主动把标志位置 1，读回时抹掉：

```cpp
// ── 写（构造/扩容） ──
long_.ptr_      = new char[len + 1];
long_.size_     = len;
long_.capacity_ = len | SSO_MASK;       // 真实容量 | 标志位
// 例: len=64 → capacity_ = 64 | 0x80000000'00000000

// ── 读（capacity()） ──
size_t capacity() const {
    return long_.capacity_ & ~SSO_MASK;  // 抹掉标志位 → 64
}

// ── 判断模式 ──
bool is_long() const {
    return long_.capacity_ & SSO_MASK;   // 最高位 = 1?
}
// 条件: sizeof(size_t) == 8, SSO_MASK = 1ull << 63
```

### 简化实现骨架

```cpp
class string_sso {
    static constexpr size_t SSO_MASK = size_t(1) << (sizeof(size_t) * 8 - 1);
    static constexpr size_t SSO_CAP  = sizeof(Long) - 2;  // 留 1 字节存长度

    struct Long {
        char*  ptr_;
        size_t size_;
        size_t capacity_;  // 最高位 = 1
    };
    struct Short {
        char    buf_[sizeof(Long) - 1];  // 23 字节 buffer
        uint8_t len_;                     // 短字符串长度（≤22）
    };
    // sizeof(Long) == sizeof(Short) == 24
    // short_.len_ 的位置恰好等于 long_.capacity_ 的最高 byte
    // → len_ ≤ 22 < 128，最高位自然为 0 → 区分长/短

    union { Long long_; Short short_; };

public:
    string_sso(const char* s) {
        size_t len = strlen(s);
        if (len <= SSO_CAP) {
            memcpy(short_.buf_, s, len);
            short_.buf_[len] = '\0';
            short_.len_ = (uint8_t)len;        // 最高位 = 0
        } else {
            long_.ptr_ = new char[len + 1];
            memcpy(long_.ptr_, s, len + 1);
            long_.size_ = len;
            long_.capacity_ = len | SSO_MASK;  // 最高位 = 1
        }
    }

    const char* data() const {
        return is_long() ? long_.ptr_ : short_.buf_;
    }

    size_t size() const {
        return is_long() ? long_.size_ : short_.len_;
    }

    ~string_sso() {
        if (is_long()) delete[] long_.ptr_;
    }

private:
    bool is_long() const { return long_.capacity_ & SSO_MASK; }
};
```

### SSO 对 move 的影响

```cpp
std::string s = "hi";           // SSO 内存储
std::string t = std::move(s);   // 并非"偷指针"——只是 memcpy 了 24 字节的 SSO buffer
```

SSO 字符串的数据在对象内部。move 不能"偷"内部数据（源对象析构时它还在原地），只能逐字节拷贝。小字符串的 move 和 copy 开销一样——没有堆 buffer 可偷。详见 [[move-semantics]] Q3。

### 其他差异

| 特性           | 简化实现              | 标准 std::string    |
| ------------ | ----------------- | ----------------- |
| COW（写时复制）    | 无                 | C++11 前 GCC 用，现已禁止 |
| allocator 支持 | 无                 | 模板参数              |
| 小内存分配        | 一律 new            | SSO 短字符串零分配       |

面试中如果被问 "你的实现和标准库的差距"，SSO 是第一个该提的点。

## 面试速记

- **内部三个成员？** `char* data_` + `size_t size_` + `size_t capacity_`，都不含 `'\0'`。
- **为什么 copy-and-swap？** 一次搞定异常安全 + 自赋值安全 + 少写代码。
- **capacity 要加 1 吗？** `new char[capacity_ + 1]`，分配时 +1 给 `'\0'`。
- **push_back 的异常安全？** 先扩容（失败不影响），后写入（不抛异常）→ 强异常安全。
- **移动后源对象什么状态？** `data_ = nullptr`，可安全析构。

## Connections

- [[move-semantics]]: 移动构造/赋值、noexcept 对容器的意义、Rule of Five
- [[string-view]]: String 的只读视图，O(1) substr 替代品，与本实现的交互
- [[shared-ptr]]: 同属 Rule of Five 手撕题，对比内存管理策略

## Raw Source

- [[raw/interview-problems/手撕题目：实现类似C++ string类.md]]
