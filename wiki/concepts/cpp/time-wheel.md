---
title: "时间轮定时器"
type: concept
created: 2026-05-24
updated: 2026-05-24
sources: ["raw/interview-problems/手撕题目：C++实现一个定时器.md"]
questions: [1]
tags: [data-structure, timer, c++, system-programming]
---

## Definition

时间轮（Timing Wheel）是一种 O(1) 插入、O(1) 取消、O(1) tick 的定时器数据结构。核心思想是把时间轴离散化为固定大小的 slot，用环形数组 + 链表存储定时器，每 tick 推进一格执行当前 slot 内所有到期任务。

## 单级时间轮

### 结构

```
slot_count = 8, tick_interval = 10ms (一轮覆盖 0~80ms)

     ┌───┬───┬───┬───┬───┬───┬───┬───┐
     │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │   ← 环形数组
     └───┴─┬─┴───┴───┴───┴───┴───┴───┘
           │
           ▼
        [T1] → [T2]   ← 每个 slot 是链表的头指针
```

- `tick_interval`：每次 tick 推进的时间粒度
- `slot_count`：槽数量，通常为 2 的幂（用位运算取模）
- 第 k 个 slot 存放 `[tick_interval * k, tick_interval * (k+1))` 区间到期的定时器

### 核心参数计算

```cpp
class SimpleTimeWheel {
    static constexpr int WHEEL_SIZE = 256;  // 256 个 slot，位运算取模
    static constexpr int TICK_MS = 10;      // 每个 tick 10ms
    // 一轮覆盖: 256 * 10ms = 2.56s

    struct TimerNode {
        uint64_t id;
        uint64_t expires;          // 绝对过期时间 (ms 时间戳)
        int rotation = 0;          // 还需要转多少圈
        std::function<void()> cb;
        TimerNode* next = nullptr;
    };

    TimerNode* slots_[WHEEL_SIZE] = {};  // 全初始化为 nullptr
    int current_slot_ = 0;
    uint64_t tick_count_ = 0;            // 总 tick 计数，用于计算绝对位置
};
```

### 插入：核心计算

给定一个 delay_ms 的定时器，决定它放在哪个 slot：

```cpp
void addTimer(uint64_t id, uint64_t delay_ms, std::function<void()> cb) {
    auto* node = new TimerNode{id, delay_ms + tick_count_ * TICK_MS, 0, std::move(cb)};

    // 过期时刻离现在还有多少 tick
    uint64_t idx = node->expires / TICK_MS;
    // 放在哪个 slot
    int slot = idx & (WHEEL_SIZE - 1);   // 等价于 idx % WHEEL_SIZE

    // 需要转几圈？每一轮 = WHEEL_SIZE 个 tick
    uint64_t cur_tick = tick_count_;
    node->rotation = (idx - cur_tick) / WHEEL_SIZE;

    // 头插法放入链表
    node->next = slots_[slot];
    slots_[slot] = node;
}
```

**关键计算**：
- `idx = expires / TICK_MS`：定时器在第几个 tick 到期
- `slot = idx % WHEEL_SIZE`：落在哪个槽
- `rotation = (idx - cur_tick) / WHEEL_SIZE`：当前 tick 到到期 tick 之间要转多少整轮

**例子**：WHEEL_SIZE=256，每个 tick=10ms。当前 tick_count=1000。插入一个 3s 后的定时器。
- 到期 tick = 1000 + 3000/10 = 1000 + 300 = 1300
- slot = 1300 & 255 = 1300 % 256 = 20
- rotation = (1300 - 1000) / 256 = 300 / 256 = 1

即这个定时器放在 slot 20，需要等轮子转 1 圈后再次到达 slot 20 时才触发。

### 推进：tick 逻辑

```cpp
void tick() {
    int slot = current_slot_;
    TimerNode* curr = slots_[slot];
    slots_[slot] = nullptr;  // 清空当前槽

    // 遍历链表中所有节点
    while (curr) {
        TimerNode* next = curr->next;
        if (curr->rotation == 0) {
            // 到期了：触发回调，删除节点
            curr->cb();
            delete curr;
        } else {
            // 还没到期：rotation--，放回槽里
            curr->rotation--;
            curr->next = slots_[slot];
            slots_[slot] = curr;
        }
        curr = next;
    }

    current_slot_ = (current_slot_ + 1) & (WHEEL_SIZE - 1);
    tick_count_++;
}
```

**rotation > 0 时为什么放回同一个 slot？** 因为 rotation 表示"还需要转完整的 WHEEL_SIZE 个 tick 整圈"。每过一个 tick，转动一格。等转完一整圈回到这个 slot 时，再检查 rotation——rotation > 0 就减 1 继续等，rotation == 0 就触发。

这是单级时间轮最简实现，可以工作但有明显问题。

### 单级时间轮的致命缺陷

WHEEL_SIZE = 256，TICK_MS = 10ms，一轮覆盖 2.56s。

如果要插入一个 **60s 后**的定时器：

```
idx_delta = 60000/10 = 6000 ticks
slot = (1000 + 6000) & 255 = 88
rotation = 6000 / 256 = 23 圈
```

tick() 每 10ms 执行一次。这个定时器在 slot 88 的链表里，每 2.56s 被遍历到一次，rotation 从 23 递减。要等 23 × 2.56s 才触发——正确性没问题，但**每次 tick 到 slot 88 时都要遍历到这个节点做 rotation--**，浪费 CPU。

更糟的是：如果有 10000 个长延迟定时器分散在所有 slot 里，每次 tick 都要做大量无意义的 rotation 递减。

**结论**：单级时间轮适用于所有定时器延迟都不超过一轮覆盖范围（`WHEEL_SIZE * TICK_MS`）的场景。

## 分层时间轮（Hierarchical Timing Wheel）

解决长延迟定时器的方法：像时钟一样分层。

### 直观类比

```
时钟：
- 秒针每 60s 转一圈
- 秒针转一圈 → 分针走一格
- 分针转一圈 → 时针走一格

分层时间轮：
- L0（毫秒轮）：128 slots，每 slot=10ms，覆盖 0~1.28s
- L1（秒轮）：    64 slots，每 slot=1.28s（= L0 一整轮），覆盖 1.28s~81.92s
- L2（分钟轮）：  64 slots，每 slot=81.92s（= L1 一整轮），覆盖 ~87min
```

### 级联（Cascade）：核心操作

当 L0 转完一圈回到 slot 0 时，触发一次 cascade：**把 L1 当前 slot 的所有定时器分散插入到 L0**。

```
L0 转完一圈 ──→ L1 当前槽的定时器 ──cascade──→ 重新插入 L0（甚至 L1/L2）
                                                 L1 指针前进一格
```

同理，L1 转完一圈回到 slot 0 时，把 L2 当前 slot cascade 到 L0+L1。

### 为什么这解决了长延迟问题？

一个 60s 后的定时器：
1. 插入时：计算发现超过了 L0 范围 → 放入 L1
2. 秒级粒度下它在 L1 里安静待着，每 1.28s cascade 时才动一次（而不是每 10ms）
3. 等 cascade 到 L0 后，才进入毫秒级精度

**每个 level 的 slot 粒度是该 level 的某一整轮**，所以 cascade 频率随 level 指数递减。

### 完整实现

```cpp
class HierarchicalTimeWheel {
public:
    // 配置
    static constexpr int TVR_BITS = 8;   // L0: 256 slots
    static constexpr int TVN_BITS = 6;   // L1~L4: 64 slots per level
    static constexpr int TVR_SIZE = 1 << TVR_BITS;  // 256
    static constexpr int TVN_SIZE = 1 << TVN_BITS;  // 64
    static constexpr int TVR_MASK = TVR_SIZE - 1;
    static constexpr int TVN_MASK = TVN_SIZE - 1;
    static constexpr int TICK_MS = 10;

    struct TimerNode {
        uint64_t id;
        uint64_t expires;       // 绝对到期 tick 编号
        std::function<void()> cb;
        TimerNode* next = nullptr;
        bool cancelled = false;
    };

    // L0: 256 slots，粒度=1 tick
    TimerNode* tv1_[TVR_SIZE] = {};
    // L1~L4: 各 64 slots，粒度递增
    TimerNode* tv2_[TVN_SIZE] = {};
    TimerNode* tv3_[TVN_SIZE] = {};
    TimerNode* tv4_[TVN_SIZE] = {};
    TimerNode* tv5_[TVN_SIZE] = {};

    uint64_t jiffies_ = 0;  // 当前 tick 计数

    // ============ 内部：查找应该插入到哪层 ============
    // 返回 (wheel 指针, index)
    static constexpr int TVR_GRANULARITY = 1;
    static constexpr int TVN1_GRANULARITY = TVR_SIZE;           // 256  tick
    static constexpr int TVN2_GRANULARITY = TVN1_GRANULARITY * TVN_SIZE; // 16384 tick
    static constexpr int TVN3_GRANULARITY = TVN2_GRANULARITY * TVN_SIZE; // 1M tick
    static constexpr int TVN4_GRANULARITY = TVN3_GRANULARITY * TVN_SIZE; // 64M tick

    void addTimer(uint64_t id, uint64_t delay_ms, std::function<void()> cb) {
        auto* node = new TimerNode{id, jiffies_ + delay_ms / TICK_MS, std::move(cb)};
        internalAdd(node);
    }

    void internalAdd(TimerNode* node) {
        uint64_t expires = node->expires;
        uint64_t idx = expires - jiffies_;  // 距离现在多少 tick

        if (idx < TVR_SIZE) {
            // 放入 L0
            int slot = expires & TVR_MASK;
            node->next = tv1_[slot];
            tv1_[slot] = node;
        } else if (idx < (1ULL << (TVR_BITS + TVN_BITS))) {
            // 放入 L1
            int slot = (expires >> TVR_BITS) & TVN_MASK;
            node->next = tv2_[slot];
            tv2_[slot] = node;
        } else if (idx < (1ULL << (TVR_BITS + 2 * TVN_BITS))) {
            // 放入 L2
            int slot = (expires >> (TVR_BITS + TVN_BITS)) & TVN_MASK;
            node->next = tv3_[slot];
            tv3_[slot] = node;
        } else if (idx < (1ULL << (TVR_BITS + 3 * TVN_BITS))) {
            // 放入 L3
            int slot = (expires >> (TVR_BITS + 2 * TVN_BITS)) & TVN_MASK;
            node->next = tv4_[slot];
            tv4_[slot] = node;
        } else {
            // 放入 L4（最粗粒度层）
            int slot = (expires >> (TVR_BITS + 3 * TVN_BITS)) & TVN_MASK;
            node->next = tv5_[slot];
            tv5_[slot] = node;
        }
    }

    // ============ 级联 ============
    void cascade(TimerNode** from_wheel, int idx) {
        // 把 from_wheel[idx] 链表中所有节点重新 internalAdd
        TimerNode* curr = from_wheel[idx];
        from_wheel[idx] = nullptr;
        while (curr) {
            TimerNode* next = curr->next;
            if (!curr->cancelled) {
                internalAdd(curr);  // 重新插入，此时 expires - jiffies 更小，可能降级
            } else {
                delete curr;        // 级联时顺便清理 cancelled 节点
            }
            curr = next;
        }
    }

    // ============ 每 tick 推进 ============
    void tick() {
        uint64_t idx = jiffies_ & TVR_MASK;  // L0 当前槽

        // cascade 操作：当 L0 指针转回到 slot 0 时，从 L1 cascade 一批过来
        if (idx == 0) {
            // L0 转完一圈：cascade L1 当前槽
            int idx1 = (jiffies_ >> TVR_BITS) & TVN_MASK;
            cascade(tv2_, idx1);

            if (idx1 == 0) {
                // L1 也转完一圈：cascade L2
                int idx2 = (jiffies_ >> (TVR_BITS + TVN_BITS)) & TVN_MASK;
                cascade(tv3_, idx2);

                if (idx2 == 0) {
                    int idx3 = (jiffies_ >> (TVR_BITS + 2 * TVN_BITS)) & TVN_MASK;
                    cascade(tv4_, idx3);

                    if (idx3 == 0) {
                        int idx4 = (jiffies_ >> (TVR_BITS + 3 * TVN_BITS)) & TVN_MASK;
                        cascade(tv5_, idx4);
                    }
                }
            }
        }

        // 执行 L0 当前槽的所有定时器
        TimerNode* curr = tv1_[idx];
        tv1_[idx] = nullptr;
        while (curr) {
            TimerNode* next = curr->next;
            if (!curr->cancelled) {
                curr->cb();
            }
            delete curr;
            curr = next;
        }

        jiffies_++;
    }
};
```

### 关键点解读

**为什么 idx < TVR_SIZE 放 L0？**

距离到期时间 < 256 个 tick → 放入 L0（最细粒度 1 tick/slot），确保精度。

**为什么用 `expires >> N` 做 L1~L4 的 slot 索引？**

L0 一个 slot = 1 tick。L1 一个 slot = TVR_SIZE ticks（256 tick = 一轮 L0）。所以 L1 的第 k 个 slot 覆盖 `[k * 256, (k+1) * 256)` tick。对于定时器在 tick 时刻 T 到期：`T / 256` 的商决定在 L1 的哪个 slot。位运算就是 `T >> 8`。

**为什么 cascade 只发生在 L0 指针回到 slot 0？**

因为此时 L0 刚转完完整一圈，L1 一个 slot 的覆盖范围恰好等于这一圈。L1 当前 slot 里的定时器应该在这一圈内到期 → cascasde 到 L0 获取更精确调度。

**cascade 时的删除**：级联时 cancelled 节点直接被 free，不用单独清理逻辑。

### 时间复杂度分析

| 操作 | 复杂度 | 说明 |
|------|--------|------|
| add | O(1) | 头插法，常数时间 |
| cancel | O(1) | 标记 cancelled=true |
| tick (L0 触发) | O(1) | 遍历当前槽链表 |
| cascade | O(m) | m = 被 cascade 的定时器数，均摊 O(1) |

**add 的 O(1) 是关键优势**。最小堆的 O(log n) 在 n 达到数万时变为瓶颈，而时间轮保持常数。

cascade 均摊 O(1) 的直觉：每个定时器只被 cascade 一次（或少数几次），分摊到每次 tick 就是常数。

### 取消定时器

```cpp
std::unordered_map<uint64_t, TimerNode*> id_map_;

bool cancel(uint64_t id) {
    auto it = id_map_.find(id);
    if (it == id_map_.end()) return false;
    it->second->cancelled = true;  // 惰性删除
    id_map_.erase(it);
    return true;
}
```

cancelled 节点在 cascade 时被释放，或在 tick 到 L0 槽时被跳过。不会内存泄漏。

## 与最小堆的对比

```
场景：10,000 个定时器，每秒新增 1,000 个，tick 频率 = 100Hz

最小堆：
  add: 1000 * log2(10000) ≈ 13,000 次比较/秒
  tick: 100 * 1 次 top 检查 = 100 次
  总开销 ≈ 13,100 次操作/秒

时间轮 (L0=256):
  add: 1000 * 1 头插 = 1,000 次
  tick: 100 * (遍历当前槽 ~40个节点) = 4,000 次
  cascade: 100 * 均摊极小 ≈ 可忽略
  总开销 ≈ 5,000 次操作/秒
```

当定时器数量增长，最小堆的 O(log n) 持续劣化，时间轮保持常数。

## 适用场景总结

| 场景                      | 推荐方案                        |
| ----------------------- | --------------------------- |
| 定时器 < 100，不需要高精度        | 最小堆（实现最简单）                  |
| 定时器 1k~10k，精度要求 ms 级    | 最小堆或小时间轮                    |
| 定时器 > 10k，高频 add/cancel | 单级时间轮（如果延迟不超过一轮）            |
| 定时器跨度大（ms 到分钟/小时）       | 分层时间轮                       |
| 超大规模（百万级连接）             | 分层时间轮 + 每连接独立定时器池           |
| 实时系统，绝对精度要求             | 不用时间轮，用 deadline scheduling |

## Linux 内核实现参考

Linux 内核的 timer wheel（`kernel/time/timer.c`）采用类似设计但做了实用简化：

- 用 `jiffies` 做时间基准（类似 tick_count）
- 按 `expires - jiffies` 的差值决定插入哪级
- 差值 0~255 → L0，256~16383 → L1，16384~1048575 → L2，...
- 每级粒度和范围都以 2 的幂为边界

内核版不维护 id_map（调用方自己记住），cancel 通过 `del_timer()` 直接操作节点。

## Connections

- [[c++-timer]]: 定时器设计的完整方案对比，最小堆、红黑树等替代方案
- [[red-black-tree]]: 最小堆方案的底层数据结构
