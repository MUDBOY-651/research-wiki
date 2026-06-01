---
title: "mmap 内存映射"
type: concept
created: 2026-05-28
updated: 2026-05-28
sources: ["raw/system/内存与文件管理.md"]
questions: [2]
tags: [linux, memory, virtual-memory, io, kernel]
---

## Definition

`mmap` 是 Linux/Unix 的**内存映射机制**，将文件、设备或匿名内存映射到进程的虚拟地址空间。映射后程序像访问普通内存一样读写文件内容，无需显式调用 `read()`/`write()`。

> `mmap` 让文件内容看起来像一段内存。

```
进程虚拟地址空间：

+------------------+
| heap             |
+------------------+
| mmap 区域         |  --->  映射到 data.txt 的页缓存
+------------------+
| stack            |
+------------------+
```

## 核心机制：按需分页（Demand Paging）

调用 `mmap` 时**并不立即把整个文件读进内存**，只建立虚拟地址到文件的映射关系。实际 I/O 发生在首次访问时：

```
访问 p[0]
    ↓
MMU 查页表，发现该页未映射（present bit = 0）
    ↓
触发缺页异常（page fault）
    ↓
内核的 page fault handler：
  1. 分配一个物理页帧
  2. 从磁盘/页缓存读取对应数据到该页帧
  3. 更新页表项，指向新分配的物理页
  4. 设置 present bit = 1
    ↓
CPU 重新执行导致 page fault 的指令，这次成功
```

这就是 mmap 和虚拟内存、页表、缺页异常紧密耦合的原因。

## 关键参数

### MAP_PRIVATE — 私有映射 + COW

```c
void* addr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                  MAP_PRIVATE, fd, 0);
```

- 修改映射内存**不会**写回原文件
- 写操作触发 **COW（Copy-On-Write）**：内核为写入页创建进程私有的副本
- 适用场景：加载配置文件、可执行文件的 .text/.data 段

### MAP_SHARED — 共享映射

```c
void* addr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                  MAP_SHARED, fd, 0);
```

- 修改映射内存**会同步回原文件**（通过页缓存）
- 多进程映射同一文件时，任一进程的写入对其他进程可见
- `msync(addr, size, MS_SYNC)` 强制立即刷回磁盘
- 适用场景：多进程共享内存、数据库持久化

### MAP_ANONYMOUS — 匿名映射

```c
// 不关联任何文件，类似 malloc 大块内存
void* addr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                  MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
```

- `fd = -1`，不关联文件
- 分配的页面初始化为 0
- `malloc` 在大块分配时底层会使用 `mmap(MAP_ANONYMOUS)` 而非 `brk()`

## mmap vs read/write：为什么少一次拷贝

传统 `read()` 的数据路径：

```
磁盘 → 内核页缓存(buffer) → 用户态 buffer
              ↑ 内核空间      ↑ 用户空间
              └── 一次拷贝 ──┘
```

mmap 的数据路径：

```
磁盘 → 内核页缓存(buffer) ← 用户进程直接访问（同一块物理内存映射到用户空间）
              ↑ 零拷贝
```

mmap 让用户进程的虚拟地址直接映射到页缓存的物理页，消除了一次内核→用户的拷贝。对大数据量的随机读写尤其显著。

```c
// read/write 方式：每次访问都要拷贝
char buf[4096];
pread(fd, buf, sizeof(buf), offset);   // 页缓存 → buf
process(buf);

// mmap 方式：直接访问页缓存
char* p = mmap(NULL, file_size, PROT_READ, MAP_PRIVATE, fd, 0);
process(&p[offset]);  // 直接在页缓存上操作，无拷贝
```

## 优点

1. **减少一次用户态拷贝**：直接映射页缓存，省去内核→用户的 memcpy
2. **随机访问方便**：像数组一样 `p[offset]` 直接寻址，不需要 `lseek` + `read`
3. **多进程共享**：`MAP_SHARED` 映射同一文件，天然共享内存
4. **延迟加载**：按需分页，大文件只占实际访问部分的物理内存

## 陷阱与缺点

### 1. Page Fault 开销

首次访问某页会触发缺页异常，延迟不可控（需要磁盘 I/O）。对延迟敏感的路径，可提前做 madvise 预热：

```c
// 预读：建议内核提前加载，减少后续 page fault
madvise(addr, size, MADV_SEQUENTIAL);   // 顺序访问模式
madvise(addr, size, MADV_WILLNEED);     // 马上就需要，立即预读
madvise(addr, size, MADV_RANDOM);       // 随机访问，减少预读浪费
```

### 2. SIGBUS — 文件被截断

如果 mmap 了一个文件，然后其他进程将其截断（truncate），访问超出新长度的映射区域会收到 `SIGBUS`（而非 `SIGSEGV`）。

```c
// 场景：进程 A mmap 了 4KB 文件
char* p = mmap(NULL, 4096, PROT_READ, MAP_SHARED, fd, 0);

// 进程 B 把文件截成 2KB
ftruncate(fd, 2048);

// 进程 A 访问后 2KB
char c = p[3000];  // SIGBUS！页表有映射但文件已不包含这部分数据
```

SIGBUS vs SIGSEGV：SIGSEGV 是页表没有映射（访问非法地址），SIGBUS 是页表有映射但对文件的实际访问不合法。

### 3. 不适合小文件/顺序读写

对 KB 级的小文件或纯顺序读写，mmap 的 page fault 开销和页表维护成本可能超过 `read/write`。`read/write` 可以使用预读（readahead）优化顺序访问。

## 典型使用场景

| 场景 | 方式 | 原因 |
|------|------|------|
| 大文件随机访问（数据库） | mmap | 零拷贝 + 自然寻址 |
| 多进程共享内存 | MAP_SHARED | 内核管理的共享内存 |
| 加载动态库/可执行文件 | mmap（内核内部） | 按需加载 .text/.data，页面可回收 |
| 大块内存分配（>128KB） | MAP_ANONYMOUS | malloc 底层用 mmap 处理大块 |
| 小文件顺序读 | read | 避免 page fault 和页表开销 |
| 流式数据（socket、pipe） | read/write | mmap 只能映射文件/设备 |

## 面试速记

- **mmap 比 read 快在哪？** 消除内核→用户的拷贝，用户直接读写页缓存。
- **mmap 后数据什么时候到内存？** 不是调用时，是首次访问时（page fault + demand paging）。
- **MAP_PRIVATE vs MAP_SHARED？** PRIVATE 写时复制不写回文件，SHARED 修改可见且同步文件。
- **mmap 的致命问题？** 文件被截断后访问导致 SIGBUS，不是 SIGSEGV。
- **什么时候不用 mmap？** 小文件、流式 I/O、延迟敏感且无法预热的路径。

## Connections

- [[virtual-memory]]: mmap 的底层依赖——虚拟地址空间、页表、缺页异常
- [[page-cache]]: mmap 映射的核心数据结构，直接映射的页缓存物理页
- [[copy-on-write]]: MAP_PRIVATE 写操作触发的 COW 机制

## Raw Source

- [[raw/system/内存与文件管理.md]]
