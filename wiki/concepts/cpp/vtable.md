---
title: "vtable (Virtual Table)"
type: concept
created: 2026-05-23
updated: 2026-05-23
sources: ["raw/C++ cook book/语法基础.md"]
questions: [1, 2]
tags: [c++, polymorphism, compiler-abi, memory-layout]
---

## Definition

vtable（虚表，virtual table）是 C++ 编译器为实现虚函数动态分发而生成的函数指针数组。每个包含虚函数的类有一个独立的 vtable，存储该类所有虚函数的实际地址。每个对象通过其 vptr（虚表指针）找到对应类的 vtable，从而在运行时调用正确的虚函数实现。

vtable 的第二职责是承载 RTTI 信息：vtable 的前几个槽位存储了 `type_info*` 和偏移量信息，供 [[dynamic-cast]] 和 `typeid` 使用。

## Key Ideas

### 基础模型

```
对象实例                     vtable (per-class, 全局唯一)
┌──────────┐               ┌──────────────────────┐
│ vptr ────┼──────────────►│ offset_to_top    (0) │
│ member_1 │               │ type_info*        (8) │  ← RTTI
│ member_2 │               │ &ClassName::f1  (16) │  ← 虚函数槽
└──────────┘               │ &ClassName::f2  (24) │
                           └──────────────────────┘
```

- **vptr**：每个对象内部隐式插入的指针成员（通常位于对象起始位置），占用 8 字节（64 位系统）
- **vtable**：每个多态类一份，存储在 `.rodata` 或 `.data.rel.ro` 段

### 虚函数调用的汇编过程

```cpp
class Base {
public:
    virtual void f() { }
    virtual void g() { }
};

Base* p = new Derived();
p->f();  // 调用的是 Derived::f()
```

编译为类似以下伪代码：

```cpp
// p->f() 的实际执行过程：
void* vptr = *(p);              // 1. 解引用对象首地址，获取 vptr
void** vtable = vptr;           // 2. vptr 指向 vtable
void (*func_ptr)() = vtable[2]; // 3. 读取 vtable[2]（f 的槽位，索引取决于声明顺序）
func_ptr(p);                    // 4. 调用函数指针，this 指针作为隐式参数传入
```

对应的 x86-64 汇编（简化）：

```asm
mov  rax, QWORD PTR [rdi]      ; rax = vptr (rdi = this)
mov  rax, QWORD PTR [rax + 16] ; rax = vtable[2] (f 的地址)
call rax                         ; 调用 Derived::f
```

### vtable 的内存布局（Itanium ABI）

```
vptr 指向 vtable 的第二个 slot（不是开头！）
vptr - 16  → offset_to_top    ← 用于交叉转换的指针调整
vptr - 8   → type_info*        ← RTTI 类型信息
vptr + 0   → func_slot_0       ← 第一个虚函数的地址
vptr + 8   → func_slot_1       ← 第二个虚函数的地址
...

vtable 中的函数按照声明顺序排列，派生类覆写的函数覆盖对应槽位。
```

**Itanium ABI** 是一份独立于硬件的 C++ 二进制接口规范，原名 Itanium C++ ABI。它诞生于 1990 年代末 Intel/HP 为 Itanium 处理器制定统一编译标准时，但后来被移植到 x86-64、ARM64 等所有主流架构，Linux/macOS 上的 GCC、Clang、ICC 都遵循它。它定义了对象布局、name mangling、虚函数分发、RTTI 结构、异常处理 unwinding 等所有编译器需要对齐的底层约定。

**其他布局：MSVC ABI**（Windows）采用不同设计。vptr 直接指向第一个虚函数地址（无负索引槽），RTTI 信息挂在 vtable 末尾而非开头，通过 `RTTICompleteObjectLocator` 链访问：

```
MSVC 布局:
vptr ──► ┌───────────────────┐
          │ &virtual_func_0   │  ← 直接是虚函数，无 offset_to_top
          │ &virtual_func_1   │
          │ ...               │
          │ Complete Locator* │◄── RTTICompleteObjectLocator → type_info
          └───────────────────┘
```

| | Itanium ABI | MSVC ABI |
|---|---|---|
| RTTI 位置 | vptr 负偏移（vptr[-1]） | vtable 末尾 |
| type_info 访问 | 1 层间接（直接到 type_info） | 多层间接（locator → descriptor） |
| 支持平台 | Linux, macOS, BSD 等 | Windows |
| 使用编译器 | GCC, Clang, ICC | MSVC |

ARM C++ ABI 是 Itanium 的简化子集，布局原则相同但结构更紧凑。

### 单继承的 vtable 构造

```cpp
class A {
    virtual void f();
    virtual void g();
};
class B : public A {
    virtual void f() override;  // 覆写 f
    virtual void h();           // 新增虚函数
};
```

```
A 的 vtable:                  B 的 vtable:
┌──────────────────┐         ┌──────────────────┐
│ offset_to_top    │         │ offset_to_top    │
│ type_info: &A    │         │ type_info: &B    │
│ &A::f            │         │ &B::f            │  ← 覆写
│ &A::g            │         │ &A::g            │  ← 继承
└──────────────────┘         │ &B::h            │  ← 新增在末尾
                             └──────────────────┘
```

### 多继承的 vtable 布局

多继承时，派生类有**多个 vptr**，对应每个基类子对象一个 vtable：

```cpp
class A { virtual void f(); };
class B { virtual void g(); };
class C : public A, public B {
    virtual void f() override;
    virtual void g() override;
};
```

```
C 对象的内存布局:            A-in-C 的 vtable:        B-in-C 的 vtable:
┌──────────────┐            ┌─────────────────┐      ┌─────────────────────┐
│ A::vptr ─────┼───────────►│ offset_to_top:0 │      │ offset_to_top: -16  │
│ A 数据成员    │            │ type_info: &C   │      │ type_info: &C       │
├──────────────┤            │ &C::f           │      │ &C::g               │
│ B::vptr ─────┼───────┐    └─────────────────┘      └─────────────────────┘
│ B 数据成员    │       │
├──────────────┤       └──► B-in-C 的 vtable（见右）
│ C 数据成员    │
└──────────────┘
```

**关键细节**：`B-in-C 的 vtable` 中 `offset_to_top = -16`，因为 B 子对象在 C 对象内部偏移 +16 处，回到 C 的首地址需要 -16。这个偏移量被 [[dynamic-cast]] 在交叉转换时使用。

非覆写的虚函数在 `B-in-C` 的 vtable 中需要**this 指针调整 thunk**：

```cpp
// 编译器生成的 thunk（简化）
void B::g_thunk(B* this) {
    // this 指向 B 子对象，需要调整到 C 对象
    C* real_this = (C*)((char*)this - 16);
    // 然后调用真正的 C::g，但 g 是 B 的虚函数，C 覆写时需要调整
}
```

### 虚继承的 vtable 扩展

虚继承让问题更复杂。虚基类子对象在派生类中的位置在编译期未知（由最终派生类统一放置），需要通过 vtable 中的 **vbase offset** 间接定位：

```
vtable (虚继承时还有额外条目):
┌──────────────────────┐
│ offset_to_top        │
│ type_info*           │
│ vbase_offset[0]      │  ← 当前对象到第一个虚基类的偏移
│ vbase_offset[1]      │  ← 当前对象到第二个虚基类的偏移
│ func_slot_0          │
│ func_slot_1          │
└──────────────────────┘
```

这也是 diamond 菱形继承下 vtable 复杂度爆炸的来源。

### vtable 的生成时机

- **编译期**：编译器为每个多态类生成 vtable 的内容（在 `.rodata` 的弱符号）
- **链接期**：链接器合并同名类的 vtable（ODR 规则），合并 `type_info` 对象
- **运行时**：构造函数执行期间逐步设置 vptr：
  1. 进入构造函数体之前 → vptr 指向当前构造中的类的 vtable
  2. 基类构造完成，进入派生类构造 → vptr 更新为派生类的 vtable
  3. 析构时反之，逐层恢复为基类的 vtable

这意味着在基类构造函数中调用虚函数时，不会发生多态分发——虚函数调用的是当前类（基类）的版本，而非最终派生类的覆写。

```cpp
class A {
public:
    A() { f(); }  // 调用 A::f，不是 B::f！此时 vptr 指向 A 的 vtable
    virtual void f() { std::cout << "A::f\n"; }
};
class B : public A {
public:
    B() { f(); }  // 此时 vptr 已更新到 B 的 vtable，调用 B::f
    void f() override { std::cout << "B::f\n"; }
};

B b;  // 输出: A::f（A 构造函数中） B::f（B 构造函数中）
```

### 性能考量

| 操作 | 开销 |
|------|------|
| 虚函数调用 | 两次间接寻址（对象→vptr→vtable→func），约 2-3 个 CPU 周期额外的 load |
| 非虚函数调用 | 直接跳转，零额外开销 |
| 虚函数调用 + branch predictor | 现代 CPU 的间接分支预测器通常能预测，但若虚函数实际目标频繁变化则预测失败 |
| 多继承下的虚函数调用 | 额外的 this 指针调整（thunk），可能多 1-2 条指令 |

**虚函数的真正成本不是那几次间接寻址**，而是：
1. 丧失了内联优化的机会（编译器不知道运行时实际调用哪个函数）
2. L1i cache miss（如果虚函数调用的目标在工作负载中频繁切换）
3. 阻止编译器做更多跨函数调用优化

换言之：虚函数的成本主要不是几纳秒的指针追踪，而是丧失了编译期优化的可能。

## Example

```cpp
#include <iostream>

class Animal {
public:
    virtual void speak() { std::cout << "Animal sound\n"; }
    virtual ~Animal() {}
};

class Dog : public Animal {
public:
    void speak() override { std::cout << "Woof!\n"; }
};

class Cat : public Animal {
public:
    void speak() override { std::cout << "Meow!\n"; }
};

int main() {
    Animal* a = new Dog();
    a->speak();  // Woof! — 通过 vtable 分派到 Dog::speak

    a = new Cat();
    a->speak();  // Meow! — 同一个调用点，不同目标

    // 演示构造/析构期间的 vptr 变化：见上方的代码示例

    delete a;
}
```

## Connections

- [[dynamic-cast]]: 通过 vtable 中的 `type_info*` 实现运行时类型检查
- [[rtti]]: RTTI 数据（type_info）通过 vtable 的第二个槽位存储
- [[static-cast]]: 上行转换不需要查 vtable（编译期已知偏移），效率更高
- [[conn-cpp-casts]]: 四种转换的不同底层依赖（vtable vs 编译期偏移 vs 位级别重解释）
- [[thunk]]: 多继承场景下存储在虚表中的 this 指针调整代码

## Open Questions

- MSVC 的 vtable 布局（带有完整的 RTTI 定位器链）与 Itanium ABI 的具体差异？
- LTO（Link-Time Optimization）+ PGO（Profile-Guided Optimization）能在多大程度上为间接调用做去虚拟化（devirtualization）？
- `final` 关键字的去虚拟化效果：编译器真的会利用 `final` 消除间接调用吗？→ 详见 [[q-final-devirtualization]]
