# Wiki Log

## [2026-06-01] ingest | 手撕题目：实现类似C++ string类
- Source: `raw/interview-problems/手撕题目：实现类似C++ string类.md`
- Created concepts: [[string-implementation]]
- Core questions: 1 (现代C++编程)
- Notes: 经典面试手撕题。展开为完整实现页，覆盖内部三成员不变式（data_/size_/capacity_ 不含 '\0' 的语义）、Rule of Five 完整实现、copy-and-swap 的异常安全分析、push_back/append/reserve 扩容逻辑、×2 vs ×1.5 扩容策略比较与伙伴分配器论证、异常安全三层保证的逐行分析、四个陷阱（off-by-one / nullptr / 自赋值 / 缺 noexcept）、与标准 std::string 的差距对照（SSO/COW/allocator/sizeof）。

## [2026-05-30] ingest | 系统基础 — 并发管理（lock-free 编程）
- Source: `raw/system/并发管理.md`
- Created concepts: [[lock-free-programming]]
- Core questions: 2 (计算机系统基础)
- Notes: raw 涵盖 CAS 机制和无锁编程基本介绍。展开为完整概念页，覆盖 CAS 硬件原理（x86 CMPXCHG / ARM LDREX-STREX）、weak vs strong CAS 选择、三级进度保证（blocking/lock-free/wait-free）的严格定义与代码对比、无锁栈 push/pop 完整实现（含 release/acquire 语义）、内存回收难题（Hazard Pointer / Epoch / RCU）、ABA 问题的具体执行轨迹与 tagged pointer 解决方案、使用 vs 不使用的决策矩阵、黄金法则（先 mutex 再优化）。

## [2026-05-29] ingest | 手撕题目：实现C++ shared_ptr
- Source: `raw/interview-problems/手撕题目：实现C++ shared_ptr.md`
- Created concepts: [[shared-ptr]]
- Core questions: 1 (现代C++编程)
- Created concepts (supplementary): [[atomic]]
- Notes: 经典面试手撕题。展开为完整实现页，覆盖控制块设计（strong_count/weak_count + atomic memory_order 选择理由）、完整 SharedPtr 实现（构造/拷贝/移动/析构/访问器）、拷贝赋值顺序陷阱（先 release 再 add 否则 use-after-free）、线程安全粒度（引用计数安全，对象和变量本身不安全）、WeakPtr 完整实现（lock() 的 CAS 循环防止 TOCTOU）、enable_shared_from_this 原理、make_shared 单次分配优化与内存锚定代价。5 个面试追问。

## [2026-05-28] ingest | 系统基础 — 内存与文件管理（mmap）
- Source: `raw/system/内存与文件管理.md`
- Created concepts: [[mmap]], [[virtual-memory]], [[page-cache]], [[copy-on-write]]
- Core questions: 2 (计算机系统基础)
- Notes: raw 涵盖 mmap 内存映射机制的基本介绍。展开为完整概念页，覆盖按需分页的物理过程（MMU→page fault→页表更新）、MAP_PRIVATE/MAP_SHARED/MAP_ANONYMOUS 三种模式、mmap vs read/write 的零拷贝原理（消除内核→用户拷贝）、madvise 预热策略、SIGBUS vs SIGSEGV 的区别、适用场景选择表、面试速记。首个 Systems 类概念。追加了三个 mmap 引用的系统概念页：virtual-memory（四级页表/MMU/TLB/demand paging）、page-cache（radix tree/脏页写回/双LRU淘汰）、copy-on-write（fork 零开销关键/COW page fault 完整流程/refcount 快速路径）。

## [2026-05-28] ingest | C++ cook book - 语法基础（补充 移动语义 & 完美转发）
- Source: `raw/C++ cook book/语法基础.md`
- Created concepts: [[move-semantics]]
- Core questions: 1 (现代C++编程)
- Notes: raw 新增了左值引用/右值引用/std::move/std::forward 板块。展开为完整概念页，覆盖类型 vs 值类别的核心区分、std::move 只是 cast 不移动数据、为什么具名 T&& 是左值（防止隐式重复移动）、std::forward 的引用折叠原理、四种常见错误（move const、use-after-move、forward/move 混用、return std::move 抑制 NRVO）、面试速记。

## [2026-05-26] ingest | C++ cook book - 语法基础（补充 std::string_view）
- Source: `raw/C++ cook book/语法基础.md`
- Created concepts: [[string-view]]
- Core questions: 1 (现代C++编程)
- Notes: raw 新增了 C++17 string_view 小节；展开为完整概念页，覆盖零拷贝机制、O(1) substr、remove_prefix/suffix、致命悬垂陷阱（临时对象、reallocation、表达式亡值）、与 string 对比表、null-terminated 注意事项、split tokenizer 示例。

## [2026-05-24] ingest | 手撕题目：C++实现一个定时器
- Source: `raw/interview-problems/手撕题目：C++实现一个定时器.md`
- Created concepts: [[c++-timer]], [[time-wheel]]
- Core questions: 1 (现代C++编程)
- Notes: 从最小堆方案起步，展开 steady_clock vs system_clock、锁释放时机、cv 唤醒条件、惰性删除、析构顺序等设计细节。time-wheel 单独成页，完整展开单级时间轮的插入/tick/rotation 计算及其致命缺陷、分层时间轮的 cascade 机制与完整实现代码、与最小堆的性能对比。

## [2026-05-23] ingest | C++ cook book - STL 关联容器（std::map & std::unordered_map）
- Source: `raw/C++ cook book/STL.md`
- Created concepts: [[stl-map]], [[stl-unordered-map]], [[red-black-tree]]
- Created connections: [[conn-map-vs-unordered-map]]
- Core questions: 1 (现代C++编程)
- Notes: 从 raw 笔记中几乎为空白的 std::map 章节和简略的 unordered_map 章节出发，补充了红黑树节点结构、迭代器稳定性、emplace/try_emplace/insert_or_assign 演进、extract/merge 机制、透明比较器、hash 表底层结构、迭代器失效规则、reserve vs rehash、缩容、第三方对比、以及 map vs unordered_map vs sorted vector 的三方选择决策。

## [2026-05-23] ingest | C++ cook book - 语法基础（static_cast & dynamic_cast）
- Source: `raw/C++ cook book/语法基础.md`
- Created concepts: [[dynamic-cast]], [[static-cast]], [[rtti]], [[vtable]], [[thunk]]
- Created connections: [[conn-cpp-casts]]
- Created questions: [[q-variant-vs-dynamic-cast]], [[q-final-devirtualization]]
- Core questions: 1 (现代C++编程), 2 (计算机系统基础)
- Notes: 从 raw 笔记中的 static_cast/dynamic_cast 对比出发，深入补充了 dynamic_cast 的 RTTI 实现细节（vtable 布局、继承图遍历、type_info 指针比较机制、交叉转换的指针调整算法），并扩展为四个概念/连接页的完整知识网络。
