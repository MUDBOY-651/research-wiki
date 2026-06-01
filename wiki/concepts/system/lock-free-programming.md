---
title: "无锁编程（Lock-Free Programming）"
type: concept
created: 2026-05-30
updated: 2026-05-30
sources: ["raw/system/并发管理.md"]
questions: [2]
tags: [concurrency, lock-free, cas, atomic, system]
---

## Definition

无锁编程（lock-free programming）是指**不使用互斥锁阻塞线程**，而是通过 CPU 原子指令完成并发控制。核心工具是 CAS（Compare-And-Swap）等原子原语。

> Lock-free 不是"没有锁"，而是一种**进度保证**：系统整体至少有一个线程能向前推进。单个线程可能反复失败重试，但不会出现一个线程持锁阻塞其他所有线程的情况。

## CAS：无锁编程的原子原语

CAS 是 CPU 指令级别的原子操作。x86 上是 `CMPXCHG`，ARM 上是 `LDREX/STREX`。

```
CAS(addr, expected, desired):
    atomically:
        if *addr == expected:
            *addr = desired
            return true   // 成功
        else:
            expected = *addr
            return false  // 失败，expected 被更新为当前值
```

这解决了**"检查 + 修改"的复合操作**在不加锁情况下的原子性问题。读写之间没有间隙，其他线程无法插入修改。

C++ 提供了两个接口：

```cpp
std::atomic<int> x{0};
int expected = 0;

// weak: 可能虚假失败（spurious failure），但更快，适合循环重试
x.compare_exchange_weak(expected, 1);

// strong: 只在值不匹配时失败，保证"真失败才失败"，但某些平台稍慢
x.compare_exchange_strong(expected, 1);
```

**weak vs strong 的选择**：在重试循环中用 `weak`——循环本身已经处理了重试，虚假失败只是多转一圈。`strong` 用在不需要循环的场合，比如一次性 CAS 判断状态。

## 并发算法三级进度保证

```
Wait-free  ⊂  Lock-free  ⊂  Blocking
 (最强)       (中等)        (最弱)
```

### Blocking（阻塞）

使用 mutex、condition_variable 等。一个线程持锁时，其他线程被操作系统挂起（阻塞）。

```cpp
std::mutex mtx;
void add(int delta) {
    std::lock_guard<std::mutex> lg(mtx);
    counter += delta;  // 临界区内其他线程阻塞
}
```

**问题**：
- 持锁线程被调度走 → 所有等待线程全部停滞（ convoy effect）
- 优先级反转：低优先级线程持锁，高优先级线程等待
- 死锁风险

### Lock-Free（无锁）

**系统整体至少有一个线程能向前推进**。单个线程可能无限重试失败，但不会出现"所有线程被一个挂起的线程阻塞"的情况。

```cpp
std::atomic<int> x{0};
void add(int delta) {
    int old = x.load(std::memory_order_relaxed);
    while (!x.compare_exchange_weak(old, old + delta,
             std::memory_order_release, std::memory_order_relaxed)) {
        // CAS 失败说明别的线程改了 x，old 已被更新，重试即可
    }
}
```

如果一个线程在 CAS 循环中途被调度走，其他线程不受影响——它们在自己的 CAS 循环中继续推进。

### Wait-Free（无等待）

**每个线程在有限步数内完成**，不会出现任何线程被"饿死"的情况。在所有并发层级中实现最困难。

```
lock-free:  "总有线程能推进" → 可能某个倒霉线程一直 CAS 失败
wait-free:  "每个线程都能在有限步内完成" → 没有饿死
```

工程中 wait-free 极少见——通常需要为每个线程分配独立的工作槽，避免竞争共享状态。

## 无锁栈完整实现

```cpp
#include <atomic>
#include <memory>

template <typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
        Node(const T& val) : data(val), next(nullptr) {}
    };

    std::atomic<Node*> head_{nullptr};

public:
    void push(const T& val) {
        Node* node = new Node(val);  // 新节点
        node->next = head_.load(std::memory_order_relaxed);

        // CAS 循环：尝试把 head_ 从 node->next 改为 node
        while (!head_.compare_exchange_weak(
            node->next,           // expected: 当前读到的 head
            node,                 // desired: 新的 head
            std::memory_order_release,   // 成功：写 head，release 语义
            std::memory_order_relaxed    // 失败：读 head，只需原子性
        )) {
            // CAS 失败 → node->next 已被更新为最新 head，循环重试
        }
    }

    std::optional<T> pop() {
        Node* old_head = head_.load(std::memory_order_relaxed);
        while (old_head) {
            if (head_.compare_exchange_weak(
                old_head,
                old_head->next,
                std::memory_order_acquire,
                std::memory_order_relaxed
            )) {
                T result = std::move(old_head->data);
                delete old_head;  // ← 潜在的悬垂问题
                return result;
            }
        }
        return std::nullopt;
    }
};
```

**注意**：上面的 `pop` 实现有内存回收问题。线程 A 读到 `old_head` 后，在线程 A 执行 CAS 之前，线程 B 可能已经 pop 了同一个节点并 delete 了它。线程 A 的 CAS 访问已释放内存 → use-after-free。

这是无锁编程的一个核心难题：**不能简单地 delete 被 CAS 替换掉的节点**，因为其他线程可能还在读它。解决方案包括：
- Hazard Pointer（危险指针）：线程声明"我正在读这个指针"，回收者跳过被声明的指针
- Epoch-Based Reclamation（基于纪元的回收）：维护读临界区，延迟回收
- RCU（Read-Copy-Update）：读取方零开销，更新方负责延迟回收

## ABA 问题

无锁栈的经典陷阱：

```
初始栈: A → B → C

线程 T1 正要 pop A（读到 head=A, next=B）
线程 T2 在这期间：
  pop A  →  delete A
  pop B  →  delete B
  push A'（新节点，地址恰与 A 相同）→  push B'
  栈变为: B' → A' → C

T1 执行 CAS(head, A, B):
  head 当前确实是 A（地址匹配），CAS 成功！
  但 A->next 现在是 B' 而不是 B
  栈变成 B → C，B' 和 A' 全部丢失
```

CAS 只看"地址是否相等"，不关心"这个地址在此期间经历过什么"。值从 A → B → A，CAS 误判为"什么都没变"。

**解决方案**——带 tag 的指针：

```cpp
struct TaggedPtr {
    Node* ptr;
    uintptr_t tag;  // 每次 CAS 成功后递增
};

std::atomic<TaggedPtr> head;

// push 时：
TaggedPtr old = head.load();
TaggedPtr desired{node, old.tag + 1};
while (!head.compare_exchange_weak(old, desired)) {
    desired.tag = old.tag + 1;  // 每次重试都用新的 tag
}
// 即使地址复用，tag 不同 → CAS 不会误成功
```

x86-64 上可用 cmpxchg16b（16 字节 CAS）或 ARMv8.1 的 LSE 扩展来原子操作 128 位的 `{ptr, tag}` 组合。

## 使用决策

### 适合无锁编程

| 条件 | 例子 |
|------|------|
| 操作极简单（几条指令） | 计数器递增、状态标志位翻转 |
| 临界区极小 | 单变量 CAS 更新 |
| 高频调用 | 引用计数、内存分配器内部结构 |
| 延迟敏感 | 中断处理、音频线程、交易系统 |
| 底层基础设施 | 无锁队列（SPSC/MPMC）、调度器 |

### 不适合无锁编程

| 条件 | 原因 |
|------|------|
| 普通业务逻辑 | 出 bug 成本远高于性能收益 |
| 临界区复杂 | 尝试无锁实现会极端复杂且容易错 |
| 多对象一致性 | 需要同时原子修改两个对象，CAS 做不到 |
| 团队不熟悉内存模型 | 错误的 `memory_order_relaxed` 比 mutex 更难调试 |
| 可维护性优先 | 未来维护者不一定是并发专家 |

### 黄金法则

> 先用 mutex 写清楚，确认锁竞争是性能瓶颈后，再考虑 lock-free 优化。

一把无竞争的 mutex（futex）大概 25ns。如果你的操作本身也是这个量级，lock-free 的收益不大。只有当锁竞争导致上下文切换（微秒级）时才值得优化。

## 面试速记

- **CAS 原理？** 原子地比较并交换。相等则写入新值返回成功，不等则更新 expected 返回失败。CPU 指令级支持。
- **lock-free vs wait-free？** lock-free 保证整体有线程推进（不保证单线程）；wait-free 保证每个线程有限步完成。
- **ABA 问题？** 地址被释放后复用，CAS 只看值不察觉中间变化 → 链表结构损坏。解决方案：tagged pointer。
- **无锁栈 push 为什么用 release，pop 用 acquire？** push 是生产者（release 之前的所有写入对后续 acquire 可见），pop 是消费者（acquire 看到生产者所有写入）。
- **什么时候不用无锁？** 临界区复杂、多对象一致性、团队不懂内存模型、性能瓶颈不确定是锁时。

## Connections

- [[atomic]]: CAS 的 C++ 接口（compare_exchange_weak/strong）与五种内存序
- [[shared-ptr]]: atomic 引用计数是 lock-free 的典型实际应用
- [[copy-on-write]]: COW 的引用计数增减也是 lock-free 模式
