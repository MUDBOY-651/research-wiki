---
title: "红黑树 (Red-Black Tree)"
type: concept
created: 2026-05-23
updated: 2026-05-23
sources: ["raw/C++ cook book/STL.md"]
questions: [1, 2]
tags: [c++, data-structure, balanced-tree, stl]
---

## Definition

红黑树是一种**自平衡二叉搜索树**，通过每个节点附加一个颜色位（RED/BLACK）和四条不变式约束，保证树高始终为 O(log n)，从而确保插入、删除、查找的最坏时间复杂度为 O(log n)。它是 `std::map`、`std::set` 以及 Linux CFS（完全公平调度器）、epoll、Java `TreeMap` 等系统的底层数据结构。

与 AVL 树不同，红黑树在平衡条件上做了妥协（更宽松的平衡 → 更少的旋转），在插入/删除频繁的场景下优于 AVL。

## Key Ideas

### 四条不变式与直觉

1. 每个节点要么是红色，要么是黑色
2. 根节点是黑色
3. 红色节点的子节点必须是黑色（不能有两个连续的红色）
4. 从任意节点到其所有后代 NIL 节点的路径上，黑色节点数量相同

不变式 3 和 4 是核心。4 保证了黑色高度一致，3 保证了红色节点被"隔开"——最坏情况就是红黑交替（一条全黑路径 vs 一条红黑交替路径），所以最长路径 ≤ 2 × 最短路径。

```
一条红黑树路径（从根到叶）:
●─→●─→●             ← 3个黑节点，路径长度=3 (最短路径)
●─→●─→◎─→●─→◎─→●   ← 3个黑节点+2个红节点，路径长度=5 (最长可能路径=3×2-1)
```

### 节点结构

```cpp
enum class Color { RED, BLACK };

struct RBTreeNode {
    Key key;
    Value value;
    Color color;
    RBTreeNode* parent;
    RBTreeNode* left;
    RBTreeNode* right;
};
```

每个节点 3 个指针（parent/left/right）、1 个枚举/布尔、key + value。64 位系统上指针操作开销约 (3×8 + 8 + 16-32) ≈ 48-56 字节/节点，不含分配器开销。

### 插入：不超过 2 次旋转

插入分两步：
1. **按 BST 规则插入新节点**，染红色
2. **修复红红冲突**（新节点 + 其父节点都是红色）

修复有三种情况，取决于**叔叔节点的颜色**：

```
情形1: 叔叔是红的
      ●(grand,black)         ◎(grand,red)
     / \          →         / \
  ◎(p) ◎(uncle)          ●(p)  ●(uncle)
  /                          /
◎(new,red)                ◎(new,red)
→ 变色即可，不旋转。grand 变红后可能引发新的红红冲突，向上递归

情形2: 叔叔是黑的，且新节点在父节点的内侧
      ●(grand,black)         ●(grand)
     /             →        /
  ◎(p)                   ◎(new)
     \                   /
     ◎(new)            ◎(p)
→ 旋转父节点，变成情形3

情形3: 叔叔是黑的，且新节点在父节点的外侧
      ●(grand,black)         ◎(p,new root of subtree)
     /             →        / \
  ◎(p)                   ●(new) ●(grand)
   /
◎(new)
→ 旋转 grand + 变色，完成。最多 2 次旋转，不会向上递归
```

这就是为什么插入是 O(log n) 但常数极小——大多数插入只需要变色，最坏才 2 次旋转。

### 删除：不超过 3 次旋转

删除比插入复杂，分四种情况，但最多 3 次旋转。核心思路：
- 若被删节点有两个子节点 → 用后继节点替换，转换为删除后继（至多一个子节点）
- 若被删节点是红色 → 直接删，不影响黑高
- 若被删节点是黑色 → 需要在兄弟子树"借"一个黑色节点

### 红黑树 vs AVL 树 vs B 树

| | 红黑树 | AVL 树 | B 树 |
|---|---|---|---|
| 平衡因子 | 宽松（最长 ≤ 2×最短） | 严格（高度差 ≤ 1） | 所有叶子同高度 |
| 插入旋转 | ≤ 2 次 | 可能 O(log n) 次 | 分裂（上推） |
| 查找 | 稍慢（树略深） | 更快（更紧凑） | 更快（更矮更宽） |
| 节点开销 | 1 bit (color) | 2 bits (平衡因子) | 数组 + 子指针 |
| 使用场景 | 插入/删除多 | 查找远多于修改 | 磁盘 I/O 敏感 |

**为什么 std::map 选红黑树而非 AVL？** 核心原因：插入/删除的旋转次数差异。

```
插入：
  AVL:  可能向上回溯到根，每层都可能旋转 → O(log n) 次旋转
  RB:   最多 2 次旋转（情形3 直接结束），不向上回溯

删除：
  AVL:  可能需要 O(log n) 次旋转
  RB:   最多 3 次旋转
```

每次旋转改动 3-6 个指针。AVL 的 O(log n) 次旋转不仅指令多，每层还可能触发 cache miss（节点分散在堆中），实际差距比纸面更大。STL 的关联容器 `insert` 和 `erase` 是高频操作——省旋转比省一点树高更划算。

**查找差距不大。** AVL 更严格平衡（高度差 ≤ 1），树更紧凑，查找比红黑树快约 20-30%。但查找都是 O(log n)，这个常数差距在大多数场景下被插入/删除的优势覆盖。除非数据是"构建后几乎不修改"（此时 `std::flat_map` 是更优选择，与红黑树 vs AVL 无关）。

**历史因素。** STL（HP/SGI 版本）最早选了红黑树，Alexander Stepanov 的设计目标就是插入/删除和查找的平衡。后来 Java `TreeMap`、.NET `SortedDictionary` 都是跟随 STL 的选择。Linux 内核 CFS 调度器也选了红黑树同因——进程调度的 vruntime 插入/删除极其频繁，最多 2-3 次旋转是硬需求。

与 AVL 树对比：

### 红黑树的关键属性：迭代器不依赖旋转

红黑树的另一个优势：**旋转操作不使任何迭代器失效**。旋转只改变指针指向，不重新分配节点，所以遍历期间插入/删除另一个元素不会把当前迭代器搞废。这也是 std::map 迭代器稳定性的底层原因。

## Example

```cpp
// 手动实现红黑树（简化版，仅演示插入）
#include <iostream>

enum Color { RED, BLACK };

template<typename K, typename V>
struct Node {
    K key; V value;
    Color color = RED;
    Node *left = nullptr, *right = nullptr, *parent = nullptr;
};

template<typename K, typename V>
class RBTree {
    Node<K,V>* root = nullptr;

    void rotateLeft(Node<K,V>* x) {
        auto y = x->right;
        x->right = y->left;
        if (y->left) y->left->parent = x;
        y->parent = x->parent;
        if (!x->parent) root = y;
        else if (x == x->parent->left) x->parent->left = y;
        else x->parent->right = y;
        y->left = x;
        x->parent = y;
    }

    void rotateRight(Node<K,V>* y) {
        auto x = y->left;
        y->left = x->right;
        if (x->right) x->right->parent = y;
        x->parent = y->parent;
        if (!y->parent) root = x;
        else if (y == y->parent->right) y->parent->right = x;
        else y->parent->left = x;
        x->right = y;
        y->parent = x;
    }

    void insertFixup(Node<K,V>* z) {
        while (z->parent && z->parent->color == RED) {
            auto grand = z->parent->parent;
            if (z->parent == grand->left) {
                auto uncle = grand->right;
                if (uncle && uncle->color == RED) {
                    // 情形1：叔叔红色 → 变色
                    z->parent->color = BLACK;
                    uncle->color = BLACK;
                    grand->color = RED;
                    z = grand;
                } else {
                    if (z == z->parent->right) {
                        // 情形2：节点在内侧 → 旋转父节点
                        z = z->parent;
                        rotateLeft(z);
                    }
                    // 情形3：节点在外侧 → 旋转爷节点 + 变色
                    z->parent->color = BLACK;
                    grand->color = RED;
                    rotateRight(grand);
                }
            } else {
                // 对称（父节点是右子）
                auto uncle = grand->left;
                if (uncle && uncle->color == RED) {
                    z->parent->color = BLACK;
                    uncle->color = BLACK;
                    grand->color = RED;
                    z = grand;
                } else {
                    if (z == z->parent->left) {
                        z = z->parent;
                        rotateRight(z);
                    }
                    z->parent->color = BLACK;
                    grand->color = RED;
                    rotateLeft(grand);
                }
            }
        }
        root->color = BLACK;
    }

    void inorder(Node<K,V>* n) {
        if (!n) return;
        inorder(n->left);
        std::cout << n->key << " ";
        inorder(n->right);
    }

public:
    void insert(const K& key, const V& value) {
        auto z = new Node<K,V>{key, value};
        Node<K,V>* y = nullptr;
        auto x = root;
        while (x) {
            y = x;
            x = (z->key < x->key) ? x->left : x->right;
        }
        z->parent = y;
        if (!y) root = z;
        else if (z->key < y->key) y->left = z;
        else y->right = z;
        insertFixup(z);
    }

    void print() { inorder(root); std::cout << "\n"; }
};

int main() {
    RBTree<int, std::string> tree;
    tree.insert(10, "a");
    tree.insert(5, "b");
    tree.insert(15, "c");
    tree.insert(3, "d");
    tree.insert(7, "e");
    tree.print();  // 3 5 7 10 15
}
```

## Connections

- [[stl-map]]: std::map 的底层数据结构
- [[stl-unordered-map]]: 哈希表，红黑树的替代方案，平均 O(1) 查找但无序
- [[conn-map-vs-unordered-map]]: 有序 vs 无序容器的选择决策

## Open Questions

- 红黑树在 Linux 内核 CFS（Completely Fair Scheduler）中的具体应用——如何用 vruntime 做红黑树 key 实现 O(log n) 调度？
