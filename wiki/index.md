# Wiki Index

## C++

### Concepts
- [[move-semantics]]: 移动语义与完美转发，std::move/std::forward 本质、值类别 vs 类型、引用折叠、常见误区
- [[c++-timer]]: C++ 线程安全定时器设计，最小堆/时间轮/分层时间轮，惰性删除与析构安全
- [[time-wheel]]: 时间轮定时器完整设计，单级旋转与分层 cascade 机制，O(1) add/cancel/tick
- [[string-view]]: std::string_view 非拥有型字符串视图，零拷贝 substr，替代 const std::string& 的陷阱与最佳实践
- [[string-implementation]]: String 类完整实现，Rule of Five，copy-and-swap 异常安全，扩容策略与 SSO 对比
- [[dynamic-cast]]: C++ 运行时类型转换操作符，依赖 RTTI 实现安全的下行转换和交叉转换
- [[static-cast]]: C++ 编译期类型转换操作符，零开销，替代 C 风格强制转换
- [[rtti]]: C++ 运行时类型识别机制，dynamic_cast 与 typeid 的底层依赖
- [[vtable]]: 虚表，虚函数动态分发的底层机制，同时承载 RTTI 信息
- [[thunk]]: 多继承下编译器自动生成的 this 指针调整胶水代码
- [[stl-map]]: std::map 有序关联容器，基于红黑树实现，迭代器稳定
- [[stl-unordered-map]]: std::unordered_map 无序关联容器，基于链地址法哈希表实现
- [[red-black-tree]]: 红黑树自平衡二叉搜索树，std::map 的底层数据结构
- [[shared-ptr]]: shared_ptr 完整实现，控制块与 atomic 引用计数，weak_ptr 扩展，make_shared 单次分配优化
- [[atomic]]: std::atomic 原子操作与五种内存序，relaxed/acquire-release/seq_cst 的语义与物理屏障，CAS 与 ABA，自旋锁 vs mutex

### Questions
- [[q-variant-vs-dynamic-cast]]: C++17 variant + visit 能否替代 dynamic_cast？封闭集合多态 vs 开放集合多态
- [[q-final-devirtualization]]: final 关键字的去虚拟化效果，编译器在哪些场景真正消除了间接调用

### Connections
- [[conn-cpp-casts]]: C++ 四种类型转换操作符的对比总表与选择决策流程
- [[conn-map-vs-unordered-map]]: std::map vs unordered_map vs sorted vector 的选择决策

## Systems

### Concepts
- [[mmap]]: Linux 内存映射机制，零拷贝文件 I/O，MAP_PRIVATE/SHARED，page fault 与 SIGBUS
- [[virtual-memory]]: 虚拟内存抽象，页表与 MMU 地址转换，TLB，demand paging，swap 与 overcommit
- [[page-cache]]: Linux 页缓存机制，文件 I/O 的唯一桥梁，脏页写回，双 LRU 淘汰策略
- [[copy-on-write]]: 写时复制延迟拷贝策略，fork() 零开销的关键，依赖页表权限位 + page fault
- [[lock-free-programming]]: CAS 无锁编程范式，lock-free/wait-free 进度保证层级，无锁栈实现，ABA 问题与 tagged pointer，使用决策

## Papers
*(empty)*
