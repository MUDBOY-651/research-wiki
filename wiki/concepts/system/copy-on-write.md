---
title: "写时复制（Copy-On-Write）"
type: concept
created: 2026-05-28
updated: 2026-05-28
sources: []
questions: [2]
tags: [os, memory, optimization, linux]
---

## Definition

Copy-On-Write（COW）是一种**延迟拷贝**策略：多个引用共享同一份数据，只有当其中一方尝试修改时，才真正创建一份私有副本。核心思想是用"共享 + 按需复制"替代"立即深拷贝"。

## 机制

COW 依赖虚拟内存的页表权限位实现：

```
初始状态（父子进程共享同一物理页）：
  父进程页表         物理页        子进程页表
  PTE[K] ──────→  [shared_page] ←────── PTE[K]
  R/W: Read-Only                    R/W: Read-Only
  物理页 refcount = 2

写操作发生时：
  父进程: *p = new_value
    ↓
  MMU 检查 PTE: Read-Only → 写操作违反权限 → page fault
    ↓
  内核 page fault handler:
    1. 检查 refcount > 1 → 需要 COW
    2. 分配新物理页
    3. memcpy(新页, 原页) 
    4. 更新父进程 PTE 指向新页，R/W = Read-Write
    5. 原子递减原页 refcount
    6. 返回用户态重试写操作
    ↓
  现在：父进程有私有副本，子进程仍指向原页
```

## 三个经典场景

### 1. fork()

```c
pid_t pid = fork();
```

`fork()` 后，父子进程的**所有非共享页面**都设为 COW——父子共享同一物理页，页表都标为只读。谁先写谁触发 COW 获得私有副本。这就是 `fork()` 能在 O(1) 时间"复制"整个地址空间的原因——实际上什么都没复制，只增加了页表项和页的引用计数。

```c
// 典型使用模式：fork 后直接 exec
pid = fork();
if (pid == 0) {
    execve("/bin/ls", ...);  // exec 会创建全新地址空间，COW 页全被丢弃
}
// 如果 fork 做深拷贝，上面的模式会极度浪费
// COW 使得 fork + exec 几乎零开销
```

### 2. mmap MAP_PRIVATE

```c
void* addr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                  MAP_PRIVATE, fd, 0);
```

`MAP_PRIVATE` 映射的页面初始为只读，共享页缓存。进程第一次写时触发 COW，得到页缓存的私有副本。原文件不受影响。

### 3. 应用层 COW — 典型的 `std::string` SSO

```cpp
// 旧版 libstdc++ 的 std::string 曾使用 COW（现已废弃，C++11 禁止）
// 但原理是相通的：
std::string a = "hello";
std::string b = a;       // 不拷贝 buffer，a 和 b 共享同一块内存，refcount=2
b[0] = 'H';              // 写操作！触发 COW，b 得到自己的 buffer 副本
```

**注意**：C++11 标准禁止了 `std::string` 的 COW 实现（因为 `operator[]` 返回引用无法判断是否要写），现代实现使用 SSO。

## 性能考量

### COW 的收益

```
fork() 不触发 COW：
  fork() 时间 = O(虚拟地址空间大小) × 深拷贝
  例子：4GB 进程 fork → 4GB memcpy = 数百毫秒

fork() + COW：
  fork() 时间 = O(页表项数量) × 复制页表 + 增加 refcount
  例子：4GB 进程 fork → 复制页表（~8MB）+ 设置只读 → 几毫秒
```

### COW 的代价

每次 COW 触发都有 page fault 开销：
- 用户态→内核态切换
- 物理页分配（可能触发内存回收）
- memcpy 4KB
- 页表更新 + TLB 刷新

大量写操作下，COW 的总开销 ≈ 深拷贝 + N × page fault。如果 `fork()` 后双方都大量写，不如直接做深拷贝。

## COW 与引用计数

COW 的正确性依赖**物理页的引用计数**。内核为每个物理页维护 `_refcount`：

```c
// 简化的内核逻辑
struct page {
    atomic_t _refcount;  // 有多少进程共享这个物理页
};

// fork 时：
for each shared page:
    atomic_inc(&page->_refcount);
    pte_make_readonly(ptep);  // 双方页表都标为只读

// COW page fault 时：
if (atomic_read(&page->_refcount) == 1) {
    // 只有一个引用，不用拷贝，直接设可写
    pte_make_writable(ptep);
} else {
    // 多个引用，得拷贝
    new_page = alloc_page();
    copy_page(new_page, old_page);
    atomic_dec(&old_page->_refcount);
    pte_update(ptep, new_page, writable);
}
```

refcount == 1 时的快速路径避免了不必要的拷贝——这就是为什么 `fork() + exec()` 几乎零开销：exec 前没有写操作，exec 时直接替换整个地址空间，COW 完全未触发。

## Connections

- [[virtual-memory]]: COW 依赖页表的权限位和 page fault 机制
- [[mmap]]: MAP_PRIVATE 通过 COW 实现私有映射
- [[page-cache]]: MAP_PRIVATE 的 COW 创建的是页缓存页的私有副本
