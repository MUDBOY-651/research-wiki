---
title: "std::map"
type: concept
created: 2026-05-23
updated: 2026-05-23
sources: ["raw/C++ cook book/STL.md"]
questions: [1]
tags: [c++, stl, ordered-map, red-black-tree, data-structure]
---

## Definition

`std::map<Key, T, Compare, Allocator>` 是 C++ 标准库中的有序关联容器，基于**红黑树**实现。元素按 key 排序存储，增删查均为 O(log n)。key 唯一；`std::multimap` 允许重复 key，接口和行为类似。

```cpp
#include <map>
std::map<std::string, int> m;
m["alice"] = 30;            // operator[] 不存在时会插入默认值
m.emplace("bob", 25);       // 就地构造，无临时对象
```

## Key Ideas

### 底层结构：红黑树

每个节点包含：key、value、color（RED/BLACK）、parent/left/right 三个指针。

```
节点结构（简化）:
┌──────────────────┐
│ parent*           │
│ left*  │ right*   │
│ color (RED/BLACK) │
│ key               │
│ value             │
└──────────────────┘
```

红黑树的四条不变式：
1. 节点为红色或黑色
2. NIL 节点（空叶子节点）为黑色
3. 红色节点的子节点为黑色
4. 从根到任意 NIL 节点的路径上，黑色节点数量相同

这些约束保证了**最长路径 ≤ 2 × 最短路径**，即树始终近似平衡。

与 AVL 树对比：

|       | 红黑树 (map)     | AVL 树           |
| ----- | ------------- | --------------- |
| 平衡条件  | 宽松（最长 ≤ 2×最短） | 严格（左右子树高度差 ≤ 1） |
| 插入/删除 | 最多 2-3 次旋转    | 可能 O(log n) 次旋转 |
| 查找    | 稍慢（树略深）       | 稍快（更紧凑）         |
| 适用场景  | 插入/删除频繁       | 查找远多于修改         |

### 迭代器稳定性

**这是 std::map 相比 std::unordered_map 的最大结构性优势。**

- `insert`：所有已有迭代器保持有效
- `erase`：仅被删除节点的迭代器失效，其他不变
- 迭代顺序：始终按 key 升序（可通过 Compare 自定义）

这意味着你可以在遍历过程中安全地插入或删除其他元素（不删当前元素时）。unordered_map 的 rehash 会使所有迭代器失效，做不到这一点。

### lower_bound / upper_bound / equal_range

基于有序树结构，这三个操作都是 O(log n)：

```cpp
std::map<int, std::string> m = {{1,"a"}, {3,"c"}, {5,"e"}};

auto lb = m.lower_bound(3);   // 第一个 key >= 3 → 3
auto ub = m.upper_bound(3);   // 第一个 key > 3  → 5
auto [lo, hi] = m.equal_range(3);  // [lower_bound, upper_bound)
```

这是 unordered_map 完全做不到的——无序容器只能做点查找。

### 插入操作的演进：C++11 → C++17

```cpp
std::map<std::string, int> m;

// C++03：总是构造临时 pair
m.insert(std::make_pair("key", 42));

// C++11：emplace 就地构造，避免临时对象
m.emplace("key", 42);

// C++17：键已存在时不移动 value（比 emplace 更高效）
m.try_emplace("key", 42);
// try_emplace 与 emplace 的关键区别：
//   若 key 已存在，try_emplace 不移动 value 参数，emplace 可能已移动

// C++17：存在则更新，不存在则插入
m.insert_or_assign("key", 42);
//   等价于: auto [it, ok] = m.insert_or_assign(k, v);
//   返回 pair<iterator, bool>：iterator 指向元素，bool 表示是否插入
```

`try_emplace` 比 `emplace` 好的场景：

```cpp
std::map<std::string, std::unique_ptr<Expensive>> m;
m.emplace("key", std::make_unique<Expensive>(...));  
// 即使 "key" 已存在，unique_ptr 已经被构造然后被销毁——浪费

m.try_emplace("key", std::make_unique<Expensive>(...));
// 如果 "key" 已存在，根本不构造 unique_ptr
```

### C++17 extract / merge

O(1) 把节点从一个 map 转移到另一个 map（不重新分配，不拷贝 key 和 value）：

```cpp
std::map<int, std::string> src = {{1,"a"}, {2,"b"}};
std::map<int, std::string> dst = {{3,"c"}};

// 方式1：extract 出 node_handle，再 insert 到另一个 map
auto node = src.extract(1);      // O(1)，src 中删除，node 持有元素
dst.insert(std::move(node));     // O(log n)，node 插入 dst

// 方式2：merge 直接合并整个 map
dst.merge(src);  // O(n log n)，src 中的元素逐一移入 dst
// merge 后 src 中 key 与 dst 冲突的元素仍留在 src 中

// 方式3：从 map 转 multimap
std::multimap<int, std::string> mm;
mm.merge(src);  // 所有元素都移过去，不会冲突（multimap 允许重复 key）
```

`extract` 的本质：node_handle 持有了树的节点指针，insert 时只是把节点指针重新挂到另一棵树上，**key 和 value 自始至终没有移动过**。

### C++20 contains()

```cpp
// C++17 及以前
if (m.find(key) != m.end()) { ... }  // 啰嗦

// C++20
if (m.contains(key)) { ... }         // 简洁
```

### 自定义比较器

```cpp
// 降序
std::map<int, std::string, std::greater<int>> m;

// 自定义 functor
struct CaseInsensitive {
    bool operator()(const std::string& a, const std::string& b) const {
        return std::lexicographical_compare(
            a.begin(), a.end(), b.begin(), b.end(),
            [](char c1, char c2) { return std::tolower(c1) < std::tolower(c2); }
        );
    }
};
std::map<std::string, int, CaseInsensitive> m;

// C++14：透明比较器，允许异构查找
std::map<std::string, int, std::less<>> m;  // std::less<> → std::less<void>
auto it = m.find("hello");  // 不需要构造 std::string！直接用 const char* 查找
// 无透明比较器时，find("hello") 会隐式构造一个 std::string 临时对象
```

透明比较器对性能的影响很容易被忽略——以 `const char*` 或 `std::string_view` 查找时，省去了一次临时 `std::string` 的构造和销毁。

**原理**：`std::less<>`（即 `std::less<void>`）内部有一个特殊标记 `is_transparent`：

```cpp
namespace std {
    template <>
    struct less<void> {
        using is_transparent = void;  // 告诉容器"我支持异构查找"

        template <typename T, typename U>
        bool operator()(T&& a, U&& b) const {
            return std::forward<T>(a) < std::forward<U>(b);
        }
    };
}
```

当 map 检测到比较器有 `is_transparent` 时，`find()` 等成员函数的参数类型不再限制为 `Key`，而是接受**任何能与 Key 比较的类型**。无透明比较器时，`find("hello")` 会先把 `const char*` 隐式构造成临时 `std::string`（堆分配 + 拷贝 + 析构）；有透明比较器时，比较直接在 `const char*` 和树节点中的 `std::string` 之间进行。

**不止 find，所有查找类函数都受益**：

```cpp
std::map<std::string, int, std::less<>> m;

m.find("hello");            // ✓ 都可以用 const char* 或 string_view
m.lower_bound("hello");     // ✓
m.upper_bound("hello");     // ✓
m.equal_range("hello");     // ✓
m.contains("hello");        // ✓ (C++20)
m.erase("hello");           // ✓ (C++14)
m.count("hello");           // ✓
```

**自定义类的透明比较器**：为自定义 key 类型支持异构查找：

```cpp
struct Employee {
    int id;
    std::string name;
};

struct CompareEmployee {
    using is_transparent = void;  // ← 关键标记

    bool operator()(const Employee& a, const Employee& b) const {
        return a.id < b.id;
    }
    // 重载：允许用 int 直接查找
    bool operator()(const Employee& a, int id) const { return a.id < id; }
    bool operator()(int id, const Employee& a) const { return id < a.id; }
};

std::map<Employee, std::string, CompareEmployee> m;
auto it = m.find(42);  // 用 int 查找，不构造 Employee 对象
```

内存分配是容器操作中最慢的部分之一。**热点路径上的透明比较器省下来的临时对象构造，可能比红黑树遍历本身更显著。** 但要注意：如果查找时传的本来就是 `std::string`（而非 `const char*`），那就没有区别——透明比较器优化的是"查找参数类型 ≠ Key 类型"的场景。

### 内存布局与 Cache 行为

每个节点是**独立堆分配**的，节点之间通过指针链接。遍历 map 时会跳跃访问内存，cache miss 率高。对比：

```
std::map:                    std::vector + sort:
┌──┐   ┌──┐   ┌──┐          ┌──┬──┬──┬──┬──┬──┐
│A │──→│B │──→│C │          │A │B │C │D │E │F │  连续内存
└──┘   └──┘   └──┘          └──┴──┴──┴──┴──┴──┘
 ↑ 分散在堆各处                   ↑ 一次 load 一行 cache line

遍历 100 万元素的典型时间：
  std::map:             ~15-30ms  ← 指针追踪，cache miss 多
  std::vector (sorted): ~2-5ms    ← 连续内存，预取友好
  std::unordered_map:   ~8-12ms   ← bucket 间跳跃，但 bucket 内连续
```

### 时间复杂度一览

| 操作                        | 复杂度        | 备注             |
| ------------------------- | ---------- | -------------- |
| insert / emplace          | O(log n)   |                |
| erase                     | O(log n)   | 摊销常数（节点析构）     |
| find / contains           | O(log n)   |                |
| lower_bound / upper_bound | O(log n)   | unordered 无法提供 |
| operator[]                | O(log n)   | 不存在时插入默认值      |
| extract                   | 摊销 O(1)    | 只取节点指针         |
| merge                     | O(n log n) | n = 源容器元素数     |
| 遍历全部元素                    | O(n)       | 中序遍历           |

## Example

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::map<std::string, int> m;

    // 插入
    m.try_emplace("alice", 30);    // C++17，key 不存在时插入
    m.insert_or_assign("bob", 25); // C++17，存在则更新

    // 查找
    if (auto it = m.find("alice"); it != m.end()) {
        std::cout << it->first << " = " << it->second << "\n";
    }
    // C++20: if (m.contains("alice")) { ... }

    // range
    auto [lo, hi] = m.equal_range("alice");
    for (auto it = lo; it != hi; ++it) {
        std::cout << it->first << "\n";
    }

    // extract + re-insert
    auto node = m.extract("alice");
    node.key() = "ALICE";           // 修改 key（C++17 才允许）
    m.insert(std::move(node));
}
```

## Connections

- [[stl-unordered-map]]: 哈希表实现，O(1) 查找但无序，迭代器不稳定
- [[conn-map-vs-unordered-map]]: 两种关联容器的选择决策
- [[red-black-tree]]: std::map 的底层数据结构

## Open Questions

- 红黑树节点分配器的优化：`std::pmr::map` + memory pool 能多大程度缓解分散分配？
- 什么场景下 `std::vector<std::pair>` + `std::sort` + `std::binary_search` 比 `std::map` 更高效？具体的数据量阈值？
