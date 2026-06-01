---
title: "std::map vs std::unordered_map 选择决策"
type: connection
created: 2026-05-23
updated: 2026-05-23
sources: ["raw/C++ cook book/STL.md"]
questions: [1]
tags: [c++, stl, map, unordered-map, performance, comparison]
---

## Overview

`std::map` 和 `std::unordered_map` 解决了同一个问题（键值对存储和查找），但底层数据结构完全不同，导致它们在性能、内存、迭代器行为上有系统性差异。选择取决于对**有序性、查找模式、内存布局、插入/删除频率**的权衡。

还存在第三个选项：**sorted vector / `std::flat_map`（C++23）**，在特定场景下是最优解。

## 对比总表

| 维度                        | std::map            | std::unordered_map | flat_map / sorted vector     |
| ------------------------- | ------------------- | ------------------ | ---------------------------- |
| 底层结构                      | 红黑树                 | 链地址法哈希表            | `std::vector<std::pair>`     |
| 查找                        | O(log n)            | 平均 O(1)，最坏 O(n)    | O(log n) (binary_search)     |
| 插入/删除                     | O(log n)            | 平均 O(1)，最坏 O(n)    | O(n) (需移动元素)                 |
| 有序性                       | 始终按 key 排序          | 无序                 | 是（需手动维护）                     |
| lower_bound / upper_bound | O(log n) 天然支持       | 不支持                | O(log n)                     |
| 迭代器稳定性                    | insert 不失效          | rehash 全部失效        | insert/erase 全部失效            |
| 内存                        | 节点分散（每节点 ~40B 额外开销） | 节点分散 + bucket 数组   | 连续内存                         |
| 遍历性能                      | 中等（指针追踪）            | 中等（bucket 跳跃）      | 极好（连续，一次 load 一行 cache line） |
| 引用稳定性                     | key 地址始终稳定          | key 地址始终稳定         | 不保证（vector 扩容时移动）            |
| 适用数据量                     | 任意                  | 任意                 | 几十到几十万（插入极少时）                |

## 选择决策流程

```
需要键值对存储
├─ 需要元素始终按 key 排序？
│   ├─ 插入/删除频繁 → std::map
│   └─ 构建后只查找/遍历 → std::flat_map（C++23）或 sorted vector
│
├─ 需要 lower_bound / upper_bound / equal_range？
│   └─ 是 → std::map
│
├─ 需要在遍历过程中安全地插入新元素？
│   └─ 是 → std::map（unordered_map 可能 rehash 使迭代器失效）
│
├─ 查找是关键路径且插入/删除很少？
│   └─ 是 → std::flat_map（需排序）或 unordered_map（无需排序）
│
├─ 需要引用稳定性？（有外部持有 key/value 的指针/引用）
│   └─ 是 → std::map 或 unordered_map（sorted vector 不适用）
│
├─ 元素数量大（> 10k）且插入/删除频繁？
│   └─ 是 → std::unordered_map（预 reserve 后 O(1) 插入）
│
└─ 默认选择
    └─ std::unordered_map（多数场景 O(1) 查找比 O(log n) 更优）
```

## 常见误区

### 误区1："unordered_map 总是比 map 快"

小数据量（< 100 个元素）下，红黑树 O(log n) 的常数因子小，可能比哈希表的 hash 计算 + bucket 定位更快。而且 map 的有序遍历比 unordered_map 的 bucket-hopping 遍历更紧凑（中序遍历 vs 跳 bucket + 扫链表）。

```cpp
// 100个元素的遍历：
// map: 中序遍历，节点间指针跳转
// unordered_map: 跳过空 bucket + 遍历链表 → 可能更慢
```

### 误区2："map 内存占用比 unordered_map 大"

map 的节点有 3 个指针（parent/left/right）+ color + key/value，约 40-48 字节。unordered_map 的节点有 next 指针 + key/value，约 32-40 字节，但还有 bucket 数组（通常 16-32 字节/bucket）。实际内存占用取决于 load_factor——load_factor 越低（空桶多），unordered_map 越浪费。

### 误区3："operator[] 只是查找操作"

```cpp
std::map<int, Expensive> m;
auto val = m[42];  // 若 42 不存在 → 默认构造 Expensive 并插入 → O(log n)
// 这不仅仅是查找，还可能是插入！

// 只查找不插入：
auto it = m.find(42);
if (it != m.end()) { auto val = it->second; }
```

`operator[]` 在 key 不存在时会**插入一个默认值**，对 unordered_map 同理。

## sorted vector / std::flat_map 方案

对于"构建后只查找，几乎不修改"的数据，sorted vector + binary search 是最优选择。C++23 的 `std::flat_map` 把这个模式标准化了：

```
std::map:                      std::flat_map:
┌──┐   ┌──┐   ┌──┐            ┌──────┬──────┬──────┬──────┐
│A │──→│B │──→│C │            │ A,va │ B,vb │ C,vc │ D,vd │  连续内存
└──┘   └──┘   └──┘            └──────┴──────┴──────┴──────┘
 节点分散                              cache 友好，零额外指针开销
```

`flat_map` 本质就是把 key-value pair 平铺在连续 buffer 中，靠二分查找定位（O(log n)），接口与 map 高度一致。遍历/查找比 map 快 2-10x，但插入/删除是 O(n)（移动后续所有元素）。

手写 sorted vector：

```cpp
std::vector<std::pair<int, std::string>> data = {{3,"c"}, {1,"a"}, {2,"b"}};
std::sort(data.begin(), data.end());  // O(n log n) 排序

// 查找：O(log n)，实际比 map 快（连续内存，cache 友好）
auto it = std::lower_bound(data.begin(), data.end(), 2,
    [](const auto& p, int key) { return p.first < key; });
if (it != data.end() && it->first == 2) {
    std::cout << it->second;  // 找到了
}
```

| 操作  | map                      | sorted vector         |
| --- | ------------------------ | --------------------- |
| 构建  | O(n log n)（逐个插入）         | O(n log n)（sort）      |
| 查找  | O(log n)，pointer chasing | O(log n)，连续内存，常数因子小很多 |
| 插入  | O(log n)                 | O(n)（需移动元素）           |
| 内存  | 每元素 ~40B 额外开销            | 零额外开销（pair 紧邻）        |
| 缓存  | 差                        | 极好                    |

**阈值**：有 benchmark 显示 100 万元素 sorted vector 查找比 map 快 5-10x。但只要有一次插入/删除需要 O(n) 移动元素，优势就会被抹去。

## Connections

- [[stl-map]]: 红黑树实现，有序，迭代器稳定
- [[stl-unordered-map]]: 哈希表实现，平均 O(1)，迭代器不稳定
- [[conn-cpp-casts]]: 类似的"不同场景不同选择"的对比模式
