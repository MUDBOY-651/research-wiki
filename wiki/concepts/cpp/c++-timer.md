---
title: "C++ 线程安全定时器设计"
type: concept
created: 2026-05-24
updated: 2026-05-24
sources: ["raw/interview-problems/手撕题目：C++实现一个定时器.md"]
questions: [1]
tags: [c++-concurrency, data-structure, system-design, interview]
---

## 问题

设计一个线程安全的 `TimerManager`，支持 add / cancel，后台线程调度，安全析构，仅用标准库。

## 最小堆方案（推荐起步方案）

### 数据结构

```cpp
struct TimerTask {
    uint64_t id;
    std::chrono::steady_clock::time_point expires;
    std::function<void()> callback;
    bool cancelled = false;  // 惰性删除标记
};

// 最小堆：堆顶是最早过期任务
std::priority_queue<TimerTask, std::vector<TimerTask>,
    decltype([](const auto& a, const auto& b) { return a.expires > b.expires; })> heap_;
std::unordered_map<uint64_t, TimerTask*> id_to_task_;  // O(1) 取消查找
```

### 后台线程循环

```cpp
void schedulerLoop() {
    while (!stopped_) {
        std::unique_lock lk(mu_);
        if (heap_.empty()) {
            cv_.wait(lk);  // 没任务就等
            continue;
        }
        auto& top = heap_.top();
        if (top.cancelled) { heap_.pop(); continue; }
        auto status = cv_.wait_until(lk, top.expires);
        if (status == std::cv_status::timeout) {
            auto cb = std::move(top.callback);
            heap_.pop();
            lk.unlock();  // ★ 回调执行前释放锁
            cb();
        }
        // 否则是被 addTimer 唤醒（新任务更早），重新算时间
    }
}
```

### addTimer

```cpp
uint64_t addTimer(uint64_t delay_ms, std::function<void()> cb) {
    std::lock_guard lk(mu_);
    auto id = next_id_++;
    auto expires = std::chrono::steady_clock::now() + std::chrono::milliseconds(delay_ms);
    heap_.push({id, expires, std::move(cb)});
    id_to_task_[id] = const_cast<TimerTask*>(&heap_.top()); // 注意：std::priority_queue 不返回引用

    // 如果新任务比当前堆顶更早，唤醒后台线程
    if (heap_.top().id == id) {
        cv_.notify_one();
    }
    return id;
}
```

**注意**：`std::priority_queue` 不暴露内部元素引用，上面 `id_to_task_` 的写法有问题。正确做法是用 `std::multimap`（按 expires 排序）替代 priority_queue，或自己维护 vector + make_heap，来获取 stable 引用。

### 更正确的容器：std::multimap

```cpp
std::multimap<std::chrono::steady_clock::time_point, TimerTask> tasks_;
// TimerTask 存 id, callback, cancelled flag
// id_to_task_ 存 unordered_map<uint64_t, decltype(tasks_)::iterator>
// 取消时通过 iterator 直接 O(1) 标记 cancelled
```

### cancelTimer（惰性删除）

```cpp
bool cancelTimer(uint64_t id) {
    std::lock_guard lk(mu_);
    auto it = id_to_task_.find(id);
    if (it == id_to_task_.end()) return false;
    it->second->cancelled = true;  // 惰性删除，不真正移除
    id_to_task_.erase(it);
    // 如果想立即唤醒后台线程跳过呢？不需要——后台线程在循环顶部检查 cancelled
    return true;
}
```

惰性删除的代价：堆中会堆积 cancelled 节点。大量频繁取消时需定期清理。

### 析构安全

```cpp
~TimerManager() {
    {
        std::lock_guard lk(mu_);
        stopped_ = true;
    }
    cv_.notify_all();
    if (worker_.joinable()) worker_.join();
}
```

顺序很重要：先设 stopped，再 notify，再 join。

## 关键设计决策

### 为什么用 steady_clock 而不是 system_clock？

`system_clock` 受系统时间调整（NTP / 用户改时间）影响。`steady_clock` 单调递增，不会被拨回。定时器要的是"过了多久"，不是"现在几点"。

### 为什么回调执行前要释放锁？

```
addTimer 拿锁 → 等后台线程释放锁 → 后台线程在执行回调时持锁 → 回调内部调 addTimer → 死锁
```

释放锁后执行回调是最安全的方式。

### 唤醒条件：什么时候需要 notify？

```
addTimer 添加的任务比当前堆顶更早到期 → 需要 notify_one() 打断 wait_until
否则 → 不需要，当前堆顶先过期，后台线程到时会自动醒来
```

### id 分配必须有序吗？

不需要。用 `std::atomic<uint64_t>` 递增即可。不需要在锁内分配（即使多线程 atomic 也是安全的），但要在插入 map 前赋值。

## 数据结构的进阶选择

| 结构                 | add      | cancel   | tick/唤醒 | 适用                      |
| ------------------ | -------- | -------- | ------- | ----------------------- |
| 最小堆               | O(log n) | O(1) 惰性  | O(1)    | 通用，n < 1000             |
| 红黑树 (multimap)     | O(log n) | O(log n) | O(1)    | 需要遍历过期范围                |
| 时间轮               | O(1)     | O(1)     | O(1)    | 大 n（>10k），精度不极端         |
| 分层时间轮             | O(1)     | O(1)     | O(1)    | 跨度大（ms 到 day）           |
| Deadline Scheduling | O(log n) | O(1)     | OS 按需   | 少量高精度（μs 级），零空转，Linux only |

### 时间轮与分层时间轮

详见 [[time-wheel]]，完整展开单级时间轮的插入/tick/rotation 计算、单级方案的致命缺陷、分层时间轮的 cascade 机制、完整实现代码、与最小堆的性能对比、以及 Linux 内核参考实现。

## 单线程 vs 多线程

如果所有定时器回调都在同一个线程执行（如 libevent），就不需要锁，性能更高。多线程方案在于 addTimer 来自不同调用者但执行在单独线程。这种模式本质是生产者-消费者：

- 生产者（add/cancel 线程）：入队、修改标记、通知
- 消费者（后台线程）：等待、弹出、执行回调

## Deadline Scheduling：OS 级别的精确唤醒

时间轮和最小堆的共同假设是"主动轮询"——定时器自己维护 tick 循环，到了时间点触发。另一种思路是**把唤醒责任交给 OS**：告诉内核"最近的任务在 T 时刻到期，到点叫我"，线程阻塞，OS 精确到时唤醒。

本质区别：
```
轮询模式：每隔固定间隔 tick，检查有没有到期任务 → 精度受限于 tick 粒度
Deadline 模式：告诉 OS 下一个 deadline，阻塞等待 → OS 在 T 时刻直接唤醒，微秒级精度
```

### timerfd + epoll 实现

```cpp
#include <sys/timerfd.h>
#include <sys/epoll.h>

class DeadlineScheduler {
public:
    DeadlineScheduler() {
        epfd_ = epoll_create1(0);
        worker_ = std::thread(&DeadlineScheduler::run, this);
    }

    // 添加定时器：返回 timerfd 的 fd
    int addTimer(uint64_t delay_ms, std::function<void()> cb) {
        int tfd = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK);

        struct itimerspec ts{};
        ts.it_value.tv_sec  = delay_ms / 1000;
        ts.it_value.tv_nsec = (delay_ms % 1000) * 1'000'000;
        ts.it_interval = {0};  // 0 = 一次性定时器

        timerfd_settime(tfd, 0, &ts, nullptr);

        struct epoll_event ev{};
        ev.events = EPOLLIN;
        ev.data.fd = tfd;
        epoll_ctl(epfd_, EPOLL_CTL_ADD, tfd, &ev);

        {
            std::lock_guard lk(mu_);
            fds_[tfd] = std::move(cb);
        }
        return tfd;
    }

    // 重新调度已有定时器（比删除+创建更高效）
    void reschedule(int tfd, uint64_t delay_ms) {
        struct itimerspec ts{};
        ts.it_value.tv_sec  = delay_ms / 1000;
        ts.it_value.tv_nsec = (delay_ms % 1000) * 1'000'000;
        timerfd_settime(tfd, 0, &ts, nullptr);
    }

    void cancel(int tfd) {
        epoll_ctl(epfd_, EPOLL_CTL_DEL, tfd, nullptr);
        close(tfd);
        std::lock_guard lk(mu_);
        fds_.erase(tfd);
    }

private:
    void run() {
        struct epoll_event events[64];
        while (!stopped_) {
            // -1 = 无限期阻塞，直到最早定时器到期
            int n = epoll_wait(epfd_, events, 64, -1);
            for (int i = 0; i < n; i++) {
                int fd = events[i].data.fd;
                uint64_t expirations;
                read(fd, &expirations, sizeof(expirations));  // 必须读掉，否则 LT 模式反复触发

                std::function<void()> cb;
                {
                    std::lock_guard lk(mu_);
                    cb = std::move(fds_[fd]);
                }
                if (cb) cb();
            }
        }
    }

    std::unordered_map<int, std::function<void()>> fds_;
    int epfd_;
    std::thread worker_;
    std::mutex mu_;
    std::atomic<bool> stopped_{false};
};
```

### 关键实现细节

**为什么要 `read(fd, &expirations, sizeof(expirations))`？**

`timerfd` 到期后变为可读，`epoll` 在 level-triggered 模式下会反复通知。必须 `read()` 消费掉这个事件，读出的 `uint64_t` 是到期次数（对于重复定时器可能 > 1）。

**reschedule 的效率**：修改已有 timerfd 的时间用 `timerfd_settime`，不需要 `epoll_ctl` 删了再加。比 delete + create 少两个系统调用。

**取消定时器**：`epoll_ctl(..., EPOLL_CTL_DEL, ...)` 从 epoll 移除，然后 `close(fd)`。顺序不重要，epoll 会自动清理已关闭的 fd。

### 多 fd vs 单 fd：两种 Deadline Scheduling 实现

前面的代码是"每个定时器一个 timerfd"，好处是唤醒时 `events[i].data.fd` 直接标识到期者，不需要扫描/弹堆：

```
addTimer → timerfd_create + timerfd_settime + epoll_ctl  (3 syscall)
wakeup   → epoll 直接告诉你哪个 fd 到了，精准 O(1)
cancel   → epoll_ctl(DEL) + close(fd)                     (2 syscall)
```

另一种是"复用单个 timerfd + 最小堆"，timerfd 始终设为堆顶的到期时间：

```cpp
class SingleFdDeadlineScheduler {
    int tfd_;   // 永远只有这一个 timerfd
    int epfd_;
    std::priority_queue<TimerTask, std::vector<TimerTask>,
        decltype([](const auto& a, const auto& b) { return a.expires > b.expires; })> heap_;
    std::unordered_set<uint64_t> cancel_set_;

    void rescheduleFd(std::chrono::steady_clock::time_point t) {
        auto delta = std::chrono::duration_cast<std::chrono::milliseconds>(
            t - std::chrono::steady_clock::now()).count();
        if (delta < 0) delta = 0;
        struct itimerspec ts{};
        ts.it_value.tv_sec  = delta / 1000;
        ts.it_value.tv_nsec = (delta % 1000) * 1'000'000;
        timerfd_settime(tfd_, 0, &ts, nullptr);
    }

    void addTimer(uint64_t delay_ms, std::function<void()> cb) {
        auto expires = std::chrono::steady_clock::now() + std::chrono::milliseconds(delay_ms);
        heap_.push({next_id_++, expires, std::move(cb)});
        if (heap_.top().id == next_id_ - 1) {
            rescheduleFd(expires);  // 新任务最早，提前唤醒
        }
    }

    void cancel(uint64_t id) {
        cancel_set_.insert(id);  // 惰性删除，零 syscall
    }

    void run() {
        struct epoll_event events[1];
        while (!stopped_) {
            epoll_wait(epfd_, events, 1, -1);
            uint64_t exp;
            read(tfd_, &exp, sizeof(exp));

            auto now = std::chrono::steady_clock::now();
            while (!heap_.empty()) {
                auto& top = heap_.top();
                if (cancel_set_.count(top.id)) {
                    cancel_set_.erase(top.id);
                    heap_.pop();
                    continue;
                }
                if (top.expires > now) {
                    rescheduleFd(top.expires);  // 还没到，重设
                    break;
                }
                auto cb = std::move(top.callback);
                heap_.pop();
                cb();
            }
        }
    }
};

/** 对比总结 */

┌──────────────────┬──────────────────────────────┬───────────────────────────┐
│                  │ 每定时器一个 timerfd           │ 单 timerfd + 最小堆         │
├──────────────────┼──────────────────────────────┼───────────────────────────┤
│ add 开销          │ 3 syscall (create+settime+ctl)│ 0~1 syscall               │
│ cancel 开销       │ 2 syscall (ctl+close)         │ 0 syscall（惰性标记）       │
│ 唤醒定位           │ O(1)，fd 直接标识              │ 必须弹堆遍历                │
│ fd 消耗            │ 1 per timer，有 ulimit 硬上限   │ 永远 1 个                 │
│ 定时器上限          │ ~数万（受 epoll fd 数限制）      │ 无上限，百万级可行           │
│ 依赖内核            │ hrtimer 树 = 你的存储          │ 堆在用户态，可随时 inspect    │
│ reschedule 效率    │ 1 syscall (timerfd_settime)    │ O(log n) 堆 sift          │
│ 跨平台              │ Linux only                   │ timerfd 部分 Linux only    │
└──────────────────┴──────────────────────────────┴───────────────────────────┘
```

核心矛盾：**fd 是稀缺资源，syscall 是昂贵的，堆操作是便宜的——设计就是在这三者之间做 trade-off。**

### 时间轮 vs Deadline Scheduling 的定位

```
时间轮适合的：海量连接超时（10万个连接的 idle timeout），精度要求 ms 级
                ↑ 百万定时器下 O(1) 的 add/tick 是关键

Deadline 适合的：少量高精度任务（音视频帧的 pts 调度、游戏主循环的 vsync），精度 μs 级
                ↑ 按需唤醒，零空转 CPU
                
两者互补，不是互斥。很多网络框架混用：
  - 连接超时管理 → 时间轮
  - 心跳/keepalive → 单 timerfd + epoll
```

### BSD/macOS 的 kqueue 等价写法

```cpp
// macOS 上用 kqueue + EVFILT_TIMER
int kq = kqueue();
struct kevent ev;
EV_SET(&ev, ident, EVFILT_TIMER, EV_ADD | EV_ONESHOT, 0, delay_ms, nullptr);
kevent(kq, &ev, 1, nullptr, 0, nullptr);

// 阻塞等待
struct kevent triggered;
int n = kevent(kq, nullptr, 0, &triggered, 1, nullptr);
// triggered.ident 就是到期的定时器标识
```

## 适用场景总梳理

把最小堆、时间轮、Deadline Scheduling 放入同一个决策框架。

### 按变量维度

**定时器数量** 是第一个分叉：

```
定时器数量
│
├─ < 100 ──── 最小堆 (cv.wait_until)，实现最简单
│             或 每定时器一个 timerfd（需要精准 reschedule 时）
│
├─ 100 ~ 10,000 ──── 最小堆 或 单 timerfd + 最小堆
│                    （如果需要 OS 级精度用后者，否则 cv 够用）
│
├─ 10,000 ~ 1,000,000 ──── 时间轮（单级或分层）
│
└─ > 1,000,000 ──── 分层时间轮
```

**精度要求** 是第二个分叉：

```
精度要求
│
├─ ms 级 (~10ms) ──── 最小堆 或 时间轮
│
├─ μs 级 ──── timerfd（每定时器或单 timerfd+堆）
│
└─ ns 级 ──── 用户态忙等 (busy-spin)，定时器机制本身达不到
```

**延迟跨度** 是第三个分叉：

```
延迟跨度
│
├─ 全部 < 1s ──── 单级时间轮 或 最小堆
├─ ms ~ min ──── 分层时间轮（2~3 层）
├─ ms ~ hour ──── 分层时间轮（4~5 层）
└─ 跨度不定 ──── 最小堆（不依赖时间离散化，最灵活）
```

### 按具体场景

| 场景                   | 定时器数     | 精度需求         | 推荐方案          | 原因                                            |
| -------------------- | -------- | ------------ | ------------- | --------------------------------------------- |
| 网络连接 idle 超时         | 10k~100k | 秒级           | 时间轮           | O(1) add/cancel/tick，精度不敏感                    |
| HTTP 请求超时            | 1k~10k   | ms 级         | 最小堆           | 数量不大，实现简单，精确到期                                |
| TCP 重传定时器            | 每连接 1 个  | ms 级         | 每连接 timerfd   | 每个定时器独立调整（RTT 变化），reschedule 只需 1 syscall     |
| 游戏主循环 tick           | 1        | ~16ms(vsync) | timerfd       | 精准帧率，OS 级唤醒                                   |
| 音视频帧 pts 调度          | < 30     | 微秒级          | 单 timerfd + 堆 | 少量高精度，按需唤醒                                    |
| 连接池心跳/keepalive      | < 1k     | 秒级           | 单 timerfd + 堆 | 定期批量触发，零空转 CPU                                |
| 限流器令牌桶补充             | 每桶 1 个   | ms 级         | 单 timerfd + 堆 | 定时器数少，需要精确间隔                                  |
| Redis 过期键淘汰          | 10k~100k | 100ms        | 时间轮           | 数量大，精度低，惰性删除天然配合                              |
| epoll/kqueue reactor | < 1k fd  | ms 级         | 单 timerfd + 堆 | 配合 event loop，timerfd 和 socket fd 统一 epoll 管理 |
| 面试手写场景               | < 100    | 无所谓          | 最小堆           | 代码量小，逻辑清晰，纯标准库                                |

### 决策草图（面试速记版）

```
面试官问：设计一个定时器

第一句话：先确认场景——定时器多不多？精度要求？跨度？

如果对方说"通用" → 最小堆 + cv.wait_until + 惰性删除
   → "如果定时器量到万级，我会换时间轮"
   → "如果精度要求微秒级，我换 timerfd + epoll"

如果对方说"海量长连接超时" → 分层时间轮
   → 解释单级时间轮的 rotation 问题
   → 讲清楚 cascade 是什么

如果对方说"高精度" → 讨论 cv vs timerfd 的精度差异
   → timerfd 是 OS hrtimer，cv 受调度器 tick 影响
   → 提出单 timerfd + 最小堆 的折中方案
```

## 常见延伸问题

1. **定时器回调里调 addTimer 会死锁吗？** 不会，如果回调前释放了锁。
2. **高精度定时器怎么实现？** 时间轮/timerfd 二选一：海量粗粒度用时间轮，少量高精度用 timerfd + epoll（Deadline Scheduling）。
3. **大量一次性定时器的内存优化？** 用 `shared_ptr` 让调用方持有取消权，定时器自身不管理生命周期。
4. **惰性删除什么时候做真正清理？** 在堆顶遇到 cancelled 时顺便 pop。也可以加 epoch 计数，超过阈值做全量清理。
5. **timerfd 的 fd 会耗尽吗？** 每个定时器占用一个 fd，大量定时器时应该复用 timerfd（用重复模式 + 最近到期时间管理），或回到时间轮方案。

## Connections

- [[time-wheel]]: 时间轮与分层时间轮的完整设计、实现与复杂度分析
- [[stl-map]]: std::multimap 实现方案的数据结构基础
- [[stl-unordered-map]]: id→task 的快速查找
- [[red-black-tree]]: multimap 底层是红黑树，决定了 O(log n) 的增删特性

- C++ 标准库实现文件见 raw source
