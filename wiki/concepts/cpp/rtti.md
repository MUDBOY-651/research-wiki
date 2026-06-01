---
title: "RTTI (Run-Time Type Identification)"
type: concept
created: 2026-05-23
updated: 2026-05-23
sources: ["raw/C++ cook book/语法基础.md"]
questions: [1, 2]
tags: [c++, type-system, vtable, compiler-abi]
---

## Definition

RTTI（Run-Time Type Identification）是 C++ 提供的运行时类型信息机制，允许程序在运行时查询对象的动态类型。它由两个核心功能组成：

- `dynamic_cast`：安全的运行时类型转换
- `typeid`：获取对象的动态类型信息

RTTI 的实现依赖于虚表（vtable）中存储的 `type_info` 指针。只有**多态类型**（至少有一个虚函数的类）才支持 RTTI。

## Key Ideas

### 数据存储位置

RTTI 信息存储在程序的数据段（`.rodata` 或 `.data.rel.ro`），每个编译单元为一个类型生成一份 `type_info` 对象，链接器负责合并重复实例。

```
┌──────────────────────────────────────┐
│ .rodata (只读数据段)                   │
│                                      │
│ type_info for A:                     │
│   __vptr → type_info::vtable         │
│   __name → "1A"                      │
│                                      │
│ type_info for Derived:               │
│   __vptr → ...                       │
│   __name → "7Derived"                │
│   __base_info[0] → type_info for A   │
│   __base_info[1] → type_info for B   │
└──────────────────────────────────────┘
```

### type_info 的唯一性保证

C++ 标准通过 ODR（One Definition Rule）和链接器合并保证同一类型的 `type_info` 在程序内全局唯一：

- 每个 `.o` 文件为它用到的类型生成 `type_info`
- 链接时，所有弱符号的 `type_info` 对象合并为一个
- `&typeid(T)` 在程序的任何位置都返回相同的地址

**例外**：使用 `dlopen` + `RTLD_LOCAL` 加载的共享库可能有自己的 `type_info` 副本（取决于链接器实现），此时 dynamic_cast 需要回退到字符串比较。

### type_info 类层次

`type_info` 自身是一个多态的类层级：

```
std::type_info               ← 所有类型的基类
  std::__class_type_info     ← 有基类的类型（class/struct）
    std::__si_class_type_info  ← 单继承
    std::__vmi_class_type_info ← 多继承 / 虚继承
  std::__fundamental_type_info ← 基础类型 (int, float, ...)
  std::__array_type_info       ← 数组类型
  std::__function_type_info    ← 函数类型
  std::__enum_type_info        ← 枚举类型
  std::__pointer_type_info     ← 指针类型
```

`dynamic_cast` 的继承图遍历只会遇到 `__class_type_info` 及其子类，因为非类类型没有继承关系。

### type_id 运算符

```cpp
#include <typeinfo>

Base* bp = new Derived();

typeid(*bp);            // 返回 bp 所指对象的动态类型 (Derived)
                        // 注意：必须解引用！typeid(bp) 返回 Base*

typeid(Derived).name(); // 返回 mangled name，如 "7Derived"
                        // name() 的返回格式因编译器而异
```

`typeid` 的内部实现：

1. 如果类型是编译期已知的静态类型 → 直接返回 `type_info` 的引用（无运行时开销）
2. 如果表达式是多态类型的解引用 → 通过 vptr → vtable → type_info* 获取动态类型

关键区别：

```cpp
typeid(Base);       // 静态，编译期确定，无开销
typeid(*bp);        // 动态（因为 bp 指向多态类型），查 vtable
typeid(Derived);    // 静态，编译期确定
```

### RTTI 的内存开销

每个多态类型额外存储：
- 每个 vtable 中一个 `type_info*`（8 字节在 64 位系统上）
- 每个 type_info 对象：vptr（8 字节）+ name 字符串指针（8 字节）+ name 字符串本身（几字节到几十字节）+ 每个基类的 `__base_class_type_info`（16 字节/基类）

对于有 1000 个类的项目，RTTI 的内存开销通常在几十 KB 量级。主要开销不是内存，而是：
- **代码体积**：每个编译单元为用到的类型生成 type_info 代码
- **运行时开销**：dynamic_cast 的继承图遍历

### 禁用 RTTI

```bash
g++ -fno-rtti main.cpp
```

禁用后的限制：
- `dynamic_cast` 不可用（编译错误）
- `typeid` 仍可用但对多态类型也返回静态类型
- 异常处理仍正常工作（不依赖 RTTI）

适用场景：
- 嵌入式系统（内存/代码体积受限）
- 游戏引擎（性能敏感，类型系统已通过其他方式控制）
- 内核开发

## Example

```cpp
#include <iostream>
#include <typeinfo>

struct A { virtual ~A() {} };
struct B : A {};

int main() {
    A* pa = new B();

    // 动态类型查询
    std::cout << typeid(*pa).name() << "\n";  // 输出 B 的 mangled name

    // 类型比较（使用指针比较，不是字符串比较）
    if (typeid(*pa) == typeid(B)) {
        std::cout << "*pa is a B\n";
    }

    // static type 不查 vtable
    std::cout << typeid(pa).name() << "\n";   // 输出 "P1A" (pointer to A)

    delete pa;
}
```

## Connections

- [[dynamic-cast]]: 依赖 RTTI 实现安全的下行转换和交叉转换
- [[static-cast]]: 不依赖 RTTI，编译期转换
- [[vtable]]: RTTI 数据通过虚表指针访问

## Open Questions

- `-fno-rtti` 下如何替代 `typeid` 的运行时类型判断？（CRTP + 自定义 type tag）
- Windows MSVC 的 RTTI 实现（基于 `RTTICompleteObjectLocator`）与 Itanium ABI 的差异？
