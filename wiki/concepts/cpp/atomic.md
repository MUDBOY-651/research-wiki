---
title: "std::atomic 与内存序"
type: concept
created: 2026-05-29
updated: 2026-05-29
sources: []
questions: [1]
tags: [c++, concurrency, memory-model, lock-free]
---

## Definition

`std::atomic<T>` 保证对类型 `T` 的操作是**不可分割的**（atomic）——其他线程看到的操作要么完全完成，要么完全未开始，不存在中间状态。同时通过内存序（memory order）参数控制**操作之间的可见性顺序**。

## 为什么需要 atomic

```cpp
// 看似简单的 count++ 实际是三条指令
int count = 0;
count++;  // ① load count → register
          // ② register + 1
          // ③ store register → count
```

两个线程同时执行 `count++` 10000 次，最终结果可能远小于 20000——线程 A 在①和③之间，线程 B 也读了旧值，两个线程基于同一个旧值写入，丢失一次递增。

```cpp
std::atomic<int> count{0};
count.fetch_add(1);  // 原子递增，硬件保证不可分割
// 两个线程各执行 10000 次 → 结果一定是 20000
```

## 核心操作

### 基础类型（int, bool, pointer 等）

```cpp
std::atomic<int> x{0};

// store / load
x.store(42);
int v = x.load();

// RMW（Read-Modify-Write）—— 读-改-写 原子执行
int old = x.fetch_add(1);      // x += 1，返回旧值
old = x.fetch_sub(1);           // x -= 1
old = x.exchange(42);           // x = 42，返回旧值
old = x.fetch_and(0xFF);        // x &= 0xFF
old = x.fetch_or(0x80);         // x |= 0x80

// CAS（Compare-And-Swap）—— 原子级 "如果...则..."
int expected = 0;
bool ok = x.compare_exchange_weak(expected, 1);
// if (x == expected) { x = 1; return true; }
// else { expected = x; return false; }
```

### 原子 flag（最简单的锁）

```cpp
std::atomic_flag lock = ATOMIC_FLAG_INIT;

// 自旋锁
while (lock.test_and_set(std::memory_order_acquire)) {
    // 忙等待——在争用极短的临界区下比 mutex 更快
}
// ... 临界区 ...
lock.clear(std::memory_order_release);
```

`atomic_flag` 保证是无锁的（lock-free），所有平台上都是。其他 `atomic<T>` 是否无锁取决于平台和 `T` 的大小。

### 判断是否无锁

```cpp
std::atomic<int> x;
x.is_lock_free();  // true — int 在所有主流平台无锁

std::atomic<BigStruct> y;  // sizeof 太大，可能用内部 mutex 实现
y.is_lock_free();           // 可能 false
```

## 内存序（Memory Order）

这是 `std::atomic` 最核心也最容易混淆的部分。不同内存序是**性能-安全**的梯度选择。

### 五种内存序

```
约束强度（强 → 弱）：
seq_cst  →  acquire-release  →  relaxed
 (默认)      (acq_rel)           (最弱)
  慢但安全    中等               最快但容易出错
```

### relaxed — 只保证原子性，不保证顺序

```cpp
std::atomic<int> x{0}, y{0};

// 线程 A
x.store(1, std::memory_order_relaxed);
y.store(1, std::memory_order_relaxed);

// 线程 B
if (y.load(std::memory_order_relaxed) == 1) {
    // 即使 y 已经是 1，x 可能还是 0！
    // relaxed 不保证跨变量的操作顺序
    assert(x.load(std::memory_order_relaxed) == 1);  // 可能失败
}
```

**适用场景**：单纯计数，不依赖计数结果推理其他数据。`shared_ptr` 的引用计数增减就可以用 relaxed（只关心计数本身，不通过计数值来推断别的）。

### acquire-release — 成对使用，建立 happens-before

```cpp
std::atomic<bool> ready{false};
int data = 0;  // 非原子，受 ready 保护

// 线程 A（生产者）
data = 42;
ready.store(true, std::memory_order_release);  // ① release：前面的写都对②可见

// 线程 B（消费者）
while (!ready.load(std::memory_order_acquire)) {}  // ② acquire：后面的读看到①之前的所有写
assert(data == 42);  // 一定成立
```

规则：**release store 之前的所有内存写，对看到同一原子变量 release 值的 acquire load 之后的操作可见。**

```
物理含义（CPU 层面）：
  store(release)：确保之前的所有 store 完成后再发布这个 store
                  （阻止 StoreStore 重排 + LoadStore 重排）

  load(acquire)：确保之后的 load/store 发生在这个 load 之后
                  （阻止 LoadLoad 重排 + LoadStore 重排）
```

### acq_rel — 读-改-写操作同时有 acquire 和 release

```cpp
// fetch_add 是 RMW 操作：
// - 读部分需要 acquire（看到之前 release 的所有写）
// - 写部分需要 release（自己的写被后续 acquire 看到）
x.fetch_add(1, std::memory_order_acq_rel);
```

这是 `shared_ptr` 引用计数递减时用的——减之前需要 acquire 看到对象的最新状态，减到 0 后的 delete 需要对后续操作可见（release）。

### seq_cst（默认）— 全局统一顺序

所有标记为 `seq_cst` 的原子操作，在所有线程看来**发生在同一个全局时间线**上。代价最高，但心智负担最低。不确定用哪个时，用默认的 `seq_cst` 不会出错。

```
seq_cst 比 acq_rel 多一条约束：
所有 seq_cst 操作之间有一个单一全局顺序（single total order）
——所有线程一致同意这些操作的发生顺序。
```

## 物理本质：缓存一致性 + 内存屏障

```
多核 CPU 的缓存结构：
Core 0              Core 1
  L1 cache            L1 cache
    │                    │
  L2 cache            L2 cache
    │                    │
    └──────┬─────────────┘
           │
        L3 cache (shared)
           │
         Memory
```

每个核有自己的 L1/L2 缓存。一个核的 `store` 不会立刻被另一个核的 `load` 看到。**缓存一致性协议**（MESI/MOESI）保证最终一致性，但**不保证瞬间可见**。

**内存屏障**（memory barrier/fence）强制特定的可见性顺序：

```cpp
// 编译器屏障：阻止编译器重排指令
std::atomic_signal_fence(std::memory_order_release);

// CPU 屏障：阻止 CPU 重排指令（发出对应 CPU 指令如 mfence/lwsync）
std::atomic_thread_fence(std::memory_order_release);
```

这就是内存序的物理基础：不同 memory_order 参数控制生成了什么级别的屏障指令。

```
x86 vs ARM：
- x86 是 TSO（Total Store Order）模型 → 硬件层面已有强顺序保证
  → acquire 几乎零开销（只是阻止编译器重排），release 也只是阻止 StoreLoad 重排
  → seq_cst 需要 mfence（较贵），但比 ARM 还是便宜得多

- ARM 是弱内存模型 → 任何非 relaxed 操作都需要显式屏障指令（dmb/dsb）
  → acquire/release 有明显开销
  → seq_cst 尤其贵
```

这意味着：对 x86 用 relaxed 和 acquire 性能差异极小，但在 ARM 上差异显著。写跨平台代码时仍应按语义需求选择正确的内存序——不要因为 "x86 上跑着没问题" 就用 relaxed。

## 自旋锁 vs mutex：什么时候用 atomic 自己做锁

```cpp
// atomic 自旋锁
std::atomic_flag lock = ATOMIC_FLAG_INIT;
void critical_section() {
    while (lock.test_and_set(std::memory_order_acquire)) {
        // 可选：多次自旋失败后用 pause / yield
        // x86: _mm_pause()   (避免忙等待占用执行单元)
        // ARM: __yield()
    }
    // ... 临界区（必须极短，几个指令）...
    lock.clear(std::memory_order_release);
}

// mutex 等价写法
std::mutex mtx;
void critical_section() {
    std::lock_guard<std::mutex> lg(mtx);
    // ... 临界区 ...
}
```

选择标准：
- `mutex`：临界区可能阻塞、持续时间不确定、有 I/O
- `atomic_flag` 自旋锁：临界区只有几个指令（比如更新单个变量），争用极短
- 如果临界区做的是 `x++` 这种单操作，直接用 `atomic<int>` 的 `fetch_add`，不需要锁

## 常见陷阱

### 1. atomic 不等于全局即时可见

```cpp
std::atomic<bool> flag{false};
// 线程 A 设 true 后，线程 B 可能要过几十纳秒才看到——不是"立即可见"
// atomic 保证的是顺序和最终可见，不是瞬时同步
```

### 2. 两个 atomic 操作不组成原子块

```cpp
std::atomic<int> x{0}, y{0};
// 即使 x 和 y 各自是 atomic 的，x++ 和 y++ 之间仍可插入其他线程的操作
x.fetch_add(1);  // ← 间隙，另一个线程可以修改 x 或 y
y.fetch_add(1);
```

### 3. relaxed 不能用于同步非原子数据

```cpp
int data = 0;
std::atomic<bool> ready{false};

// 错误：用 relaxed store 发布 data
data = 42;
ready.store(true, std::memory_order_relaxed);  // 另一个线程可能看到 ready=true 但 data=0
```

### 4. CAS 的 ABA 问题

```cpp
// 链表栈 pop 的经典 ABA：
// 线程 A：读到 head=A，A->next=B，准备 CAS(head, A, B)
// 线程 B（在 A 的读和 CAS 之间）：pop A, pop B, push A（新节点，地址巧合相同）
// 线程 A 执行 CAS：head 确实是 A！CAS 成功，但 A->next 现在不是 B 了
// → 栈结构损坏
```

无锁数据结构中通常用**带 tag 的指针**（`std::atomic<PtrWithTag>` 或 `compare_exchange_weak` 的双宽度 CAS）来解决。

## 面试速记

- **atomic 解决什么？** 操作的不可分割性（无中间状态）+ 跨线程操作可见性顺序。
- **五种内存序？** relaxed（纯原子）、acquire（读屏障）、release（写屏障）、acq_rel（RMW 用）、seq_cst（默认，全局一致顺序）。
- **relaxed vs acquire-release？** relaxed 不建立 happens-before，不能用于同步数据。acquire-release 成对形成同步关系。
- **atomic 自旋锁 vs mutex？** 临界区极短（几个指令）用自旋锁；可能阻塞/有 I/O 用 mutex。
- **CAS 的 ABA 是什么？** 值从 A→B→A，CAS 只看值不看版本号，误判为"没变过"。

## Connections

- [[shared-ptr]]: atomic 引用计数的实际应用，relaxed 增减 vs acq_rel 释放
- [[virtual-memory]]: 缓存一致性协议是 atomic 在硬件层面的实现基础
