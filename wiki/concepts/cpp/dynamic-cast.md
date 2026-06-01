---
title: dynamic_cast
type: concept
created: 2026-05-23
updated: 2026-05-23
sources:
  - raw/C++ cook book/语法基础.md
questions:
  - 1
  - 2
tags:
  - c++
  - rtti
  - type-system
  - polymorphism
---

## Definition

`dynamic_cast` 是 C++ 中唯一依赖运行时类型信息（RTTI）的转换操作符，用于在类继承层次中进行**安全的**下行转换（down-cast）和交叉转换（cross-cast）。它通过虚表中的 `type_info` 指针在运行时验证转换的合法性。

```cpp
Derived* d = dynamic_cast<Derived*>(base_ptr);
// 成功 → 返回合法的 Derived*；失败 → 返回 nullptr

Derived& d = dynamic_cast<Derived&>(base_ref);
// 成功 → 返回合法引用；失败 → throw std::bad_cast
```

## Key Ideas

### 依赖的运行时数据结构

dynamic_cast 依赖一套链式数据结构完成运行时类型判定：

```
对象实例                      虚表 (vtable)                  type_info
┌──────────────┐            ┌─────────────────┐          ┌──────────────────┐
│ vptr ────────┼───────────►│ offset_to_top   │          │ __vptr           │
│ member1      │            │ type_info* ─────┼─────────►│ __name = "7Derived"│
│ member2      │            │ func_slot_0     │          │ __base_type*     │
└──────────────┘            │ func_slot_1     │          │ __base_class[]   │
                            └─────────────────┘          └──────────────────┘
```

在 Itanium C++ ABI（GCC/Clang）中，vtable 的内存布局：

```
vptr ──► ┌─────────────────────┐
          │ offset_to_top  (0)  │  ← 用于交叉转换时调整 this 指针
          │ type_info*      (8)  │  ← 指向 RTTI 类型描述
          │ func_slot_0    (16)  │  ← 第一个虚函数
          │ func_slot_1    (24)  │
          └─────────────────────┘
```

### 三种转换类型的实现细节

**上行转换（Up-cast）：Derived* → Base***

完全不需要 RTTI。编译器在编译期就知道继承链中的偏移量，行为和 `static_cast` 一致，零运行时开销。

> 例外：虚继承涉及上行转换时，虚基类子对象的位置在编译期未知，需要通过 vbase offset 查表——但这是虚继承本身的开销，不是 dynamic_cast 独有的。

**下行转换（Down-cast）：Base* → Derived***

这是 dynamic_cast 的核心价值。运行时算法：

1. 从对象的 vptr 获取 `type_info*`
2. 调用 `__dynamic_cast()` 运行时库函数
3. 在 type_info 的继承图（`__base_class` 数组）中查找目标类型
4. 匹配成功 → 计算指针偏移并返回；失败 → 返回 nullptr

**交叉转换（Cross-cast）：BaseA* → BaseB***

最复杂的转换类型，需要指针的物理地址调整。

```cpp
class A { virtual ~A() {} int ax; };
class B { virtual ~B() {} int bx; };
class Derived : public A, public B { int dx; };

B* bp = new Derived();       // bp 指向 Derived 内的 B 子对象
A* ap = dynamic_cast<A*>(bp); // 交叉转换
```

内存布局与转换过程：

```
Derived 对象:              ┌──────────┐
                           │ A::vptr  │ ◄── &d = 0x1000
                           │ A::ax    │
                           │ B::vptr  │ ◄── bp  = 0x1010 (偏移 +16)
                           │ B::bx    │
                           │ dx       │
                           └──────────┘
```

1. `bp` = 0x1010（指向 B 子对象）
2. 从 B 的 vtable 读取 `offset_to_top` = -16
3. 计算最派生对象地址：`(char*)bp + (-16)` = 0x1000
4. 通过最派生对象的 type_info 确认类型是 Derived
5. 在 Derived 的基类列表中查找 A，偏移量 = 0
6. 返回 `(A*)(0x1000 + 0)` = 0x1000

### 多继承下的虚表结构

多继承时，派生类对每个基类子对象维护一个独立的 vtable，但全部指向同一个 `type_info`（最派生类型的）：

```
┌──────────────────────────┐
│ A-in-Derived 的 vtable    │  ← A::vptr 指向此处
│   offset_to_top: 0       │
│   type_info: &Derived ◄──│──── 指向 Derived 的 type_info
│   A::f() override        │
├──────────────────────────┤
│ B-in-Derived 的 vtable    │  ← B::vptr 指向此处
│   offset_to_top: -16     │
│   type_info: &Derived ◄──│──── 同样指向 Derived 的 type_info
│   B::g() override        │
└──────────────────────────┘
```

### type_info 的比较机制

常见误解是 dynamic_cast 使用字符串比较（`strcmp`），但实际实现使用**指针比较**：

- C++ ODR（One Definition Rule）保证每个类型在程序中只有一份 `type_info` 对象
- 链接器会合并各编译单元中相同类型的 `type_info` 实例
- `type_info::operator==` 直接比较地址，而非字符串
- 字符串比较仅在动态加载共享库（`dlopen` + `RTLD_LOCAL`）时作为 fallback

因此主要开销来自**继承图的遍历查找**，而非字符串比较。

### 性能代价

| 操作 | 开销 |
|------|------|
| 上行转换（非虚继承） | 零开销，与 static_cast 一致 |
| 上行转换（虚继承） | 查 vtable 取 vbase offset |
| 下行转换（单继承） | 一次 type_info 指针比较 |
| 下行转换（多继承） | 遍历基类图，O(n)，n = 基类数量 |
| 交叉转换 | 遍历 + 指针调整 |
| 失败路径 | 遍历完整继承图仍找不到，最坏情况 |

具体代价：`__dynamic_cast` 函数调用 + 继承图遍历 + 潜在的 cache miss（type_info 节点分散在内存各处）。

### RTTI 信息的数据结构（Itanium ABI）

```cpp
// <cxxabi.h> 中的关键类型
class type_info {
    const char* __type_name;  // mangled name，如 "7Derived"
};

class __class_type_info : public type_info {
    // 标记为"有基类"的类型
};

class __si_class_type_info : public __class_type_info {
    // 单继承
    const __class_type_info* __base_type;
};

class __vmi_class_type_info : public __class_type_info {
    // 多继承 / 虚继承
    unsigned int __flags;
    unsigned int __base_count;
    __base_class_type_info __base_info[];  // 每个基类的详细信息
};

struct __base_class_type_info {
    const __class_type_info* __base_type;
    long __offset_flags;  // 偏移量或虚基类标记
};
```

这些数据结构在编译后构成一个静态全局的有向图，dynamic_cast 的运行时查找就是对这张图做 BFS/DFS。

### 引用版本的实现

```cpp
Derived& d = dynamic_cast<Derived&>(base_ref);
// 内部：先调指针版 dynamic_cast，结果 == nullptr 则 throw std::bad_cast
```

并非编译器黑魔法，可以直接阅读 libc++abi 或 libstdc++ 源码中的 `__dynamic_cast` 验证。

### -fno-rtti

GCC/Clang 提供 `-fno-rtti` 编译选项，完全禁用 RTTI（dynamic_cast + typeid 均不可用）。许多嵌入式系统和游戏引擎（Unreal Engine）出于性能和内存考虑使用此选项。禁用后只能使用 `static_cast` 进行下行转换，由开发者自行保证类型安全。

## Example

```cpp
#include <iostream>
#include <typeinfo>

struct B { virtual ~B() {} };
struct D : B {};

int main() {
    B* b = new D();

    // 下行转换
    D* d = dynamic_cast<D*>(b);
    std::cout << "dynamic_cast: " << (d ? "success" : "null") << "\n";

    // type_info 唯一性验证
    std::cout << "typeid(*b).name(): " << typeid(*b).name() << "\n";
    std::cout << "same type_info? "
              << (&typeid(*b) == &typeid(D{})) << "\n";  // 输出 1

    // 失败示例
    B* b2 = new B();
    D* d2 = dynamic_cast<D*>(b2);  // d2 == nullptr

    delete b;
    delete b2;
}
```

## Connections

- [[static-cast]]: 编译期转换，下行转换不做运行时检查，效率更高但危险
- [[rtti]]: dynamic_cast 依赖的运行时基础设施
- [[vtable]]: 虚表是 RTTI 数据的载体
- [[conn-cpp-casts]]: C++ 四种转换操作符的全局对比

## Open Questions

- 虚继承 + diamond 菱形继承场景下，dynamic_cast 的性能退化有多大？

  相对非虚多继承退化约 **2-3x**，绝对代价仍很小（单次 30-80ns）。退化来自三个额外开销：
  1. **虚基类偏移查 vtable**：非虚继承的基类偏移是编译期立即数，虚继承需运行时从 vtable 读取，每经过一个虚基类节点多两次 load
  2. **去重标记（visited set）**：虚继承图中同一虚基类被多条路径引用（共享节点），遍历时需要跟踪已访问节点以防重复遍历，引入 O(v²) 开销（v = 已访问节点数）
  3. **`__base_info` 数组更长**：虚基类条目直接挂在最终派生类 type_info 中（带 `__virtual_mask`），遍历空间更大

  上行转换（D* → B*）不受影响，不走 `__dynamic_cast`，开销与 static_cast 一致。真正的性能杀手不是虚继承，而是**热循环中频繁做 dynamic_cast 本身**——应优先避免。

- Clang 的 Control Flow Integrity (CFI) 与 dynamic_cast 如何交互？
- C++17 `std::variant` + `std::visit` 是否可以在项目层面替代 dynamic_cast？Trade-offs 如何？→ 详见 [[q-variant-vs-dynamic-cast]]
