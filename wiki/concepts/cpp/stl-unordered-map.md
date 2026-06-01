---
title: "std::unordered_map"
type: concept
created: 2026-05-23
updated: 2026-05-23
sources: ["raw/C++ cook book/STL.md"]
questions: [1]
tags: [c++, stl, unordered-map, hash-table, data-structure]
---

## Definition

`std::unordered_map<Key, T, Hash, KeyEqual, Allocator>` 是 C++11 引入的无序关联容器，基于**链地址法哈希表**实现。平均 O(1) 查找，最坏 O(n)（大量碰撞时）。key 唯一；`std::unordered_multimap` 允许重复 key。

```cpp
#include <unordered_map>
std::unordered_map<std::string, int> m;
m["alice"] = 30;
m.emplace("bob", 25);
```

## Key Ideas

### 底层结构：链地址法（Separate Chaining）

C++ 标准库的哈希表通过 bucket 数组 + 单向链表实现：

```
bucket 数组:          每个 bucket 指向一个单向链表:
┌───┐                bucket[0] → (key1,val1) → (key2,val2) → nullptr
│ 0 │─┐              bucket[1] → nullptr      ← 空桶
├───┤ │              bucket[2] → (key3,val3) → nullptr
│ 1 │─┼──→
├───┤ │
│ 2 │─┘
├───┤
│...│
└───┘

hash(key) % bucket_count → bucket 索引
同一 bucket 内的冲突元素串成单向链表
```

标准没有规定具体实现方式，但主流编译器（GCC libstdc++、Clang libc++、MSVC）都使用链地址法。这意味每个元素都是**独立堆分配**，通过指针链接。

### 迭代器失效规则（最容易被坑的点）

| 操作 | 是否失效 | 说明 |
|------|----------|------|
| insert | 可能全部失效 | 若触发 rehash（load_factor > max_load_factor），所有迭代器失效 |
| erase | 仅被删元素的失效 | |
| rehash / reserve | 全部失效 | 重建 bucket 数组 |
| operator[] | 可能全部失效 | 若插入新元素触发 rehash |

与 std::map 的对照：

```cpp
// unordered_map：危险
for (auto it = um.begin(); it != um.end(); ++it) {
    if (condition)
        um.insert(...);  // 可能 rehash → 全部迭代器失效 → UB
}

// map：安全
for (auto it = m.begin(); it != m.end(); ++it) {
    if (condition)
        m.insert(...);   // 安全，已有迭代器不受影响
}
```

### load_factor 与 max_load_factor

```cpp
float lf    = m.load_factor();   // size() / bucket_count()，当前负载
float maxlf = m.max_load_factor(); // 触发 rehash 的阈值，默认 1.0
m.max_load_factor(0.5);          // 调低阈值，提前 rehash，用空间换查找速度

// 超过 max_load_factor 时，insert 内部触发 rehash
```

默认值各编译器不同：
- libstdc++ (GCC): 1.0
- libc++ (Clang): 1.0
- MSVC: 1.0

**max_load_factor 调低的代价**：rehash 更频繁，内存占用更大，但 bucket 链表更短 → 查找更快。

**max_load_factor 调高的代价**：rehash 更少，内存更省，但 bucket 链表更长 → 碰撞时线性扫描更长。

### reserve vs rehash

两个函数名字相似但语义不同：

```cpp
// reserve(n): 保证 bucket_count >= ceil(n / max_load_factor())
// 即预留容量能容纳 n 个元素而不触发 rehash
m.reserve(1000);  // "我要放 1000 个元素，提前把空间准备好"

// rehash(n): 显式设置 bucket_count >= n
m.rehash(2000);   // "我要 2000 个桶"
```

区别：

```cpp
// max_load_factor = 1.0 时
m.rehash(1000);   // bucket_count >= 1000 → 可容纳 1000 个元素
m.reserve(1000);  // bucket_count >= 1000 → 同上（因为 max_load_factor=1.0）

// max_load_factor = 0.5 时
m.rehash(1000);   // bucket_count >= 1000 → 可容纳 500 个元素
m.reserve(1000);  // bucket_count >= 2000 → 可容纳 1000 个元素（考虑 0.5 因子）
```

**日常使用 prefer `reserve`**——语义是"我要放 n 个元素"，不需要操心 max_load_factor。

### 缩容

unordered_map **不会自动缩容**。大量 erase 后 bucket_count 不变，load_factor 很低（内存浪费但查找快）。

主动缩容的方法：

```cpp
// 方法1：rehash 到当前元素数
m.rehash(m.size());  // 强制重建 bucket 数组为 size() 个

// 方法2：setter + 拷贝
m = std::unordered_map<Key, Value>(m.begin(), m.end());

// 方法3：C++17 的 reserve 也可用于缩容
m.reserve(m.size());  // 实际上等价于 rehash(size())
```

### 自定义哈希函数与等价判断

```cpp
struct MyHash {
    size_t operator()(const std::string& s) const {
        // FNV-1a hash，比默认的 std::hash 在某些平台上更快
        size_t h = 14695981039346656037ULL;
        for (char c : s) {
            h ^= static_cast<size_t>(c);
            h *= 1099511628211ULL;
        }
        return h;
    }
};

struct MyEqual {
    bool operator()(const std::string& a, const std::string& b) const {
        return a.size() == b.size() && std::equal(a.begin(), a.end(), b.begin());
    }
};

std::unordered_map<std::string, int, MyHash, MyEqual> m;
```

**透明哈希（C++20）**：允许异构查找（类似 map 的透明比较器）：

```cpp
struct StringHash {
    using is_transparent = std::true_type;  // 标记为透明
    size_t operator()(std::string_view sv) const { ... }
    size_t operator()(const std::string& s) const { ... }
};
std::unordered_map<std::string, int, StringHash, std::equal_to<>> m;
auto it = m.find("hello");  // C++20: 用 const char* 查找，不构造临时 string
```

### rehash 过程详解

```
扩容前: bucket_count=4, size=4, load_factor=1.0

bucket[0] → (k0,v0) → (k4,v4) → nullptr
bucket[1] → (k1,v1) → nullptr
bucket[2] → (k5,v5) → nullptr
bucket[3] → (k3,v3) → nullptr

插入第5个元素，load_factor > 1.0 → 触发 rehash

扩容后: bucket_count=8  (通常翻倍到质数或2的幂)

1. 分配新的 bucket 数组
2. 遍历旧 bucket 的每个链表，重新计算 hash(k) % 8
3. 把节点插入新 bucket 对应的链表末尾
4. 释放旧 bucket 数组
5. 所有迭代器失效
```

rehash 的复杂度 O(n)，发生在 insert 内部，**不可预测**——这是 unordered_map 最大的性能隐患。

### 性能优化

**1. 预估大小，提前 reserve**

```cpp
std::unordered_map<K, V> m;
m.reserve(expected_size);  // 避免多次 rehash
for (...) {
    m.emplace(k, v);
}
```

如果有 100 万条记录，不 reserve 可能触发 ~20 次 rehash（每次翻倍），每次重建都是 O(n) 的拷贝。reserve 后只有一次分配。

**2. 选择合适的哈希函数**

默认 `std::hash` 在不同平台表现差异大。对于常用 key 类型（int, string），默认哈希通常够用。对于自定义类型，考虑 FNV-1a 或 xxHash。

**3. 合适的 load_factor**

- 查找密集型且内存充足 → `max_load_factor(0.5)`（桶多链表短，碰撞少）
- 内存约束 → 默认 1.0 即可
- 大多数场景默认值就够，不需要手动调

### 第三方替代：为什么比标准库快 2-3x

|       | std::unordered_map     | absl::flat_hash_map | robin_hood              |
| ----- | ---------------------- | ------------------- | ----------------------- |
| 存储方式  | 链地址法（链表）               | 开放寻址法（连续数组）         | 开放寻址 + Robin Hood       |
| 内存    | 每元素独立分配                | 一整块连续内存             | 一整块连续内存                 |
| Cache | 差（指针追踪）                | 好（连续，预取友好）          | 好                       |
| 迭代    | 跳 bucket + 走链表         | 连续扫描，很快             | 连续扫描，很快                 |
| 查找    | hash → bucket → 链表线性扫描 | hash → 位置 → 线性探测    | hash → 位置 → 比距离 reorder |
| 内存占用  | 高（指针 + 桶开销）            | 较低                  | 较低                      |

关键差异：开放寻址法把整个哈希表放在一块连续内存中，遍历时 cache line 利用率极高。标准库不能改用这个设计是因为它要求**引用稳定性**（元素的地址在 insert 后不变），而开放寻址的 rehash 会移动元素。标准委员会选了链地址法来保证引用稳定性，代价是性能。

## Example

```cpp
#include <iostream>
#include <unordered_map>
#include <string>

int main() {
    std::unordered_map<std::string, int> m;

    m.reserve(100);  // 预分配

    m.try_emplace("alice", 30);        // C++17
    m.insert_or_assign("bob", 25);     // C++17

    // 查找
    if (auto it = m.find("alice"); it != m.end()) {
        std::cout << it->first << " = " << it->second << "\n";
    }
    // C++20: if (m.contains("alice")) { ... }

    // bucket 信息
    std::cout << "buckets: " << m.bucket_count()
              << ", load_factor: " << m.load_factor()
              << ", max_load_factor: " << m.max_load_factor() << "\n";

    // 主动缩容
    m.reserve(m.size());

    // 遍历同一个 bucket 的所有元素
    size_t bucket_idx = m.bucket("alice");
    for (auto it = m.begin(bucket_idx); it != m.end(bucket_idx); ++it) {
        std::cout << it->first << "\n";
    }
}
```

## Connections

- [[stl-map]]: 红黑树实现，有序，O(log n)，迭代器稳定
- [[conn-map-vs-unordered-map]]: 两种关联容器的选择决策

## Open Questions

- C++26 是否有计划引入开放寻址的哈希表容器？`std::flat_map` 的哈希版本？

  C++26 没有此计划。C++23 的 `std::flat_map` 是排序连续数组 + 二分查找（O(log n) 有序），而非哈希。开放寻址与标准库"元素地址在 insert 后不变"的契约冲突，委员会倾向于先观察 `flat_map` 的使用反馈。目前替代：`absl::flat_hash_map`、`boost::unordered_flat_map`（Boost 1.81+）等第三方实现。
- 不同编译器对 `std::hash<std::string>` 的实现差异（SipHash vs MurmurHash vs 简单多项式哈希）对性能的实际影响？
- 透明哈希（C++20 `is_transparent`）的支持在主流编译器中是否完整？

  GCC 12+ / Clang 16+ / MSVC VS 2022 17.6+ 均已完整支持（宏 `__cpp_lib_generic_unordered_lookup >= 201811L`）。但需注意与 map 透明比较器的区别：**`std::hash` 标准特化均无 `is_transparent`**，必须自行提供带 `is_transparent` 标记的 hash 函子才能启用异构查找，不像 `std::less<>` 那样换一个比较器即可。
