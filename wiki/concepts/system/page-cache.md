---
title: "页缓存（Page Cache）"
type: concept
created: 2026-05-28
updated: 2026-05-28
sources: []
questions: [2]
tags: [linux, memory, io, kernel, performance]
---

## Definition

页缓存（Page Cache）是 Linux 内核维护的**文件数据在物理内存中的缓存**。对文件的 `read`/`write` 和 mmap 访问，实际的数据都流经页缓存。它是磁盘和用户进程之间的唯一桥梁。

```
进程A                      进程B
  │  mmap / read              │  read
  ▼                           ▼
+-----------------------------------+
|          页缓存 (page cache)        |
|  物理页帧，按文件 inode+offset 索引  |
+-----------------------------------+
              │
              ▼
           磁盘
```

## 为什么需要页缓存

磁盘 I/O 比内存慢 4-5 个数量级。页缓存的核心思想：**读写文件先走内存缓存，由内核异步写回磁盘**。

- **读缓存**：同一文件块的重复读取直接从内存返回
- **写缓冲**：`write()` 只写到页缓存就返回（write-back），内核后台刷盘
- **预读（readahead）**：顺序读时内核提前把后续页面加载进缓存

## 核心数据结构

Linux 用 `struct address_space`（关联一个 inode）管理该文件所有缓存页。每个缓存页用 `struct page` 表示，通过 **radix tree**（现在用 xarray）按 offset 索引。

```
inode → address_space → radix tree (key=page offset)
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                 page[0]   page[1]   page[2]
                 (4KB)     (4KB)     (4KB)
```

## 缓存命中与未命中

### read() 的流程

```
用户调用 read(fd, buf, 4096, offset=0)
  │
  ▼
内核：在 address_space 的 radix tree 中查 offset=0 的 page
  │
  ├── 命中 → 将 page 内容 copy_to_user → 返回
  │
  └── 未命中 → 分配新 page → 加入 radix tree
                   → 发起磁盘 I/O 读取对应块
                   → I/O 完成后 → copy_to_user → 返回
```

### mmap 的流程

```
mmap 映射文件 → 建立 VMA（virtual memory area）
  │
  ▼
首次访问 p[0] → page fault
  │
  ▼
内核检查 VMA → 发现是文件映射
  │
  ├── 页缓存中有对应 page → 直接映射到进程页表（零 I/O）
  │
  └── 页缓存中无对应 page → 分配 page → 发起磁盘 I/O → I/O 完成后映射
```

`mmap` 和 `read` 共享同一个页缓存。先用 read 读到页缓存中的数据，后续 mmap 同一文件时直接命中。

## 脏页与写回（Writeback）

被写入但尚未同步到磁盘的页称为**脏页（dirty page）**。内核通过后台线程定期将脏页写回磁盘。

```bash
# 查看脏页情况
cat /proc/meminfo | grep Dirty

# 控制写回行为的参数
/proc/sys/vm/dirty_ratio          # 脏页占总内存比例达到此值时强制写回（默认 10%）
/proc/sys/vm/dirty_background_ratio # 脏页比例达到此值时启动后台写回（默认 5%）
/proc/sys/vm/dirty_expire_centisecs # 脏页超过此时长必须写回（默认 30s）
```

写回由 **pdflush/flush 内核线程** 周期性执行，或脏页比例超过阈值时触发。

## 页缓存与 OOM 的隐蔽问题

页缓存在 `free` 命令中属于 `cache`（可用内存）。但**脏页不能被直接回收**——必须先写回磁盘。在高 I/O 负载下，脏页写回慢，`cache` 内存实质上不可用，可能导致 OOM：

```bash
free -h
#               total   used    free   shared   buff/cache   available
# Mem:           16G    2.0G    1.0G    100M       13G          13G
#                                                    ↑ 大部分是页缓存，可回收
#                                               但如果其中大量是脏页，实际可回收的远少于此
```

## 页缓存淘汰：双 LRU

Linux 使用**活跃/非活跃双 LRU 链表**管理页缓存页面：

```
访问顺序 (新→旧):
活跃链表:   [page_f] → [page_e] → [page_d]
非活跃链表: [page_c] → [page_b] → [page_a]

新分配 → 非活跃链表头部
被访问 → 提升到活跃链表
内存压力 → 淘汰非活跃链表尾部的冷页
```

## Direct I/O：绕过页缓存

```c
// O_DIRECT：绕过页缓存，直接读写磁盘
int fd = open("file", O_RDONLY | O_DIRECT);
```

- 数据不经过页缓存，直接到用户 buffer
- 适用于：数据库自己管理缓存（避免内核双缓存）、一次性大文件读写
- 代价：失去预读和写缓冲，必须对齐到扇区大小

## Connections

- [[virtual-memory]]: 页缓存是虚拟内存中"文件页"的物理实现
- [[mmap]]: mmap 直接映射页缓存的物理页到用户空间，消除拷贝
- [[copy-on-write]]: MAP_PRIVATE 的写操作用 COW 创建页缓存的私有副本
