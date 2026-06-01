---
title: "C++ 四种类型转换操作符"
type: connection
created: 2026-05-23
updated: 2026-05-23
sources: ["raw/C++ cook book/语法基础.md"]
questions: [1, 2]
tags: [c++, type-system, comparison, cheatsheet]
---

## Overview

C++ 提供四种命名的类型转换操作符，替代 C 风格 `(type)expr`，每种有明确的语义和使用场景。这条连接页建立四种转换之间的关系和选择决策。

## 对比总表

| 操作符                | 阶段  | 典型场景                       | 安全性                 | 开销               |
| ------------------ | --- | -------------------------- | ------------------- | ---------------- |
| `static_cast`      | 编译期 | 隐式转换的显式标注、上行/下行转换、void* 恢复 | 下行转换不安全             | 零                |
| `dynamic_cast`     | 运行期 | 多态类型的下行转换、交叉转换             | 安全（失败返回 null/throw） | 查 vtable + 遍历继承图 |
| `const_cast`       | 编译期 | 移除/添加 const/volatile 限定    | 修改原本为 const 的对象是 UB | 零                |
| `reinterpret_cast` | 编译期 | 位级别重新解释（指针↔整数、无关类型指针互转）    | 几乎完全不安全             | 零                |

## 选择决策流程

```
需要做类型转换吗？
  ├─ 在同一继承层次内，且需要运行时安全保证
  │   → dynamic_cast（前提：有多态类型可用）
  │
  ├─ 在同一继承层次内，且编译期确定类型安全
  │   → static_cast
  │
  ├─ 基本类型间转换（int↔float, enum↔int, ...）
  │   → static_cast
  │
  ├─ 需要移除 const / volatile
  │   → const_cast
  │
  ├─ 需要在无关指针类型间转换或指针↔整数
  │   → reinterpret_cast（最后手段，极度谨慎）
  │
  └─ 不确定
      → 优先用 static_cast，只在不满足需求时考虑其他
```

## 四种转换的具体关系

### static_cast ↔ dynamic_cast

两者最常见的"选择冲突"出现在继承层次的下行转换中：

- `static_cast<Derived*>(base_ptr)`：你知道 base_ptr 确实指向 Derived，不想为 RTTI 付费
- `dynamic_cast<Derived*>(base_ptr)`：你不确定，需要运行时验证

参见 [[static-cast]] 和 [[dynamic-cast]] 的详细对比表。

### const_cast ↔ 其他转换

`const_cast` 是 C++ 中**唯一**能修改 cv 限定符的转换。常出现在：

1. 兼容不接受 const 的 C API：
```cpp
void legacy_c_api(char* buf);  // 实际不修改 buf
const char* data = "...";
legacy_c_api(const_cast<char*>(data));  // 仅在确认 API 不修改时使用
```

2. 实现 const 成员函数的 non-const 重载复用时：
```cpp
const T& get() const { /* 实际逻辑 */ }
T& get() { return const_cast<T&>(static_cast<const MyClass&>(*this).get()); }
```

### reinterpret_cast ↔ static_cast 的 void* 转换

- `static_cast<B*>(a_ptr)`：不相关类型之间编译错误
- `reinterpret_cast<B*>(a_ptr)`：编译通过，行为完全依赖实现

从 void* 恢复类型应使用 `static_cast`：
```cpp
void* vp = some_int_ptr;
int* ip = static_cast<int*>(vp);  // ✓ 明确、标准行为
int* ip2 = reinterpret_cast<int*>(vp);  // 行为与上面相同但语义不对：这不是"重新解释"，而是"恢复"
```

## 为何不用 C 风格强制转换

C 风格 `(type)expr` 的问题：

1. **语义不清**：实际执行的是 static_cast + const_cast + reinterpret_cast 的混合，代码意图不可见
2. **难以 grep**：搜索 `(int)` 无法定位所有类型转换
3. **无意间移除 const**：`(int*)(const_ptr)` 静默去除了 const，编译器无警告
4. **重构风险**：当继承层次变化时，C 风格强制转换不会触发编译错误，而 `static_cast` 可能会（无关类型间转换会报错）

## Connections

- [[dynamic-cast]]: 运行时安全转换
- [[static-cast]]: 编译期转换的主力
- [[rtti]]: dynamic_cast 的运行时依赖
- [[vtable]]: RTTI 信息的物理载体
