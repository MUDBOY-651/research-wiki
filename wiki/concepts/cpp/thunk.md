---
title: "thunk (this 指针调整)"
type: concept
created: 2026-05-23
updated: 2026-05-23
sources: []
questions: [1, 2]
tags: [c++, polymorphism, vtable, compiler, abi]
---

## Definition

thunk 是编译器在多继承场景下自动生成的一小段胶水代码，在进入虚函数之前对 `this` 指针做一次加减偏移，确保函数内部看到的 `this` 指向正确的对象地址。

从指令层面看，thunk 通常就两条指令：一条改 this，一条跳转到真正的函数，**零额外调用开销**。

```asm
; 典型 thunk（Itanium ABI）
sub  rdi, 16     ; this -= 16
jmp  target_func  ; 跳到真正的函数实现
```

## Key Ideas

### 问题：同一个函数，不同的 this

```cpp
class B {
    int bx;
public:
    virtual void g() { /* 假设 this 指向 B 对象的地址 */ }
};

class C : public A, public B {
    int cx;
public:
    void g() override { /* 假设 this 指向 C 对象的地址 */ }
};
```

```
C 对象内存:
0x1000 ┌──────────┐
       │ A 的东西  │
0x1010 ├──────────┤
       │ B 的东西  │  ← B 子对象从这里开始
       ├──────────┤
       │ C 的东西  │
       └──────────┘
```

`C::g` 只有一个实现，它期望 `this = 0x1000`（C 对象首地址）。但通过 B 的指针调用时：

```cpp
B* b = &c;    // b = 0x1010（B 子对象的地址，不是 C 的首地址！）
b->g();       // 如果直接把 0x1010 当 this 传给 C::g，内部访问 this->cx 时读到的就是乱数据
```

### thunk 做的事：进去之前调一下指针

编译器不在 B-in-C 虚表中直接放 `&C::g`，而是放一个 thunk 的地址：

```
B-in-C 虚表:
┌─────────┐
│  thunk ──┼──→ sub rdi, 16    ← this 减 16（0x1010 → 0x1000）
└─────────┘    jmp C::g         ← 跳到真正的 C::g
```

调用过程：

```
b->g()
  → this 目前是 0x1010
  → thunk: this = 0x1010 - 16 = 0x1000    ← 就这一件事
  → 跳到 C::g
  → C::g 内部看见的 this = 0x1000         ← 正确
```

### 为什么 `jmp` 而不是 `call`

`jmp` 不压栈，C::g 的 `ret` 会直接返回给最初的调用者 b->g()。thunk 本身不占栈帧，"胶水"开销就是那条 `sub` 指令。

### 需要 thunk vs 不需要

```
A* a = &c;    // a = 0x1000，恰好和 C 首地址一样
a->f();       // 不需要 thunk，this 直接就是对的 ✓

B* b = &c;    // b = 0x1010，和 C 首地址不同
b->g();       // 需要 thunk，减掉 16 字节 ✓
```

A 子对象在 C 的起始位置，`A*` = `C*`，不需要调。B 子对象在偏移处，`B*` ≠ `C*`，需要调。**B 虚表中 `offset_to_top = -16` 就是 thunk 要减去的那个偏移量**。

### 虚表中函数的放置规则

```cpp
class A { virtual void f() {} };
class B { virtual void g() {} };
class C : public A, public B {
    void f() override {}               // 覆写 A::f
    void g() override {}               // 覆写 B::g
    virtual void h() {}                // C 新增
};
```

```
A-in-C 主虚表:                B-in-C 次虚表:
┌──────────────────────┐     ┌──────────────────────┐
│ offset_to_top: 0     │     │ offset_to_top: -16   │
│ type_info: &C        │     │ type_info: &C        │
│ &C::f      ← 无thunk  │     │ &thunk→C::g ← 有thunk │
│ &C::h      ← C新增    │     └──────────────────────┘
└──────────────────────┘
```

覆写函数放在被覆写基类对应的虚表。C 新增的 `h()` 只放主 vtable——通过 B* 根本调不到 `h()`（B 的接口里没有），不需要在 B 的虚表里放槽位。

### 两种 thunk 类型

**普通 thunk（多继承）**：偏移在编译期已知，两条指令。

```asm
sub  rdi, 16
jmp  C::g
```

**vcall thunk（虚继承）**：偏移在编译期未知（虚基类位置由最派生类决定），需要运行时从 vtable 中读取。

```asm
mov   rax, [rdi]        ; 读 vptr
mov   rax, [rax + 16]   ; 读 vtable 中的 vcall_offset
add   rdi, rax           ; this += vcall_offset（动态偏移）
jmp   target_func
```

比普通 thunk 多两次内存访问，是虚继承性能代价较大的原因之一。

### thunk 的生成策略：一个 (基类偏移, 目标函数) 对应一个 thunk

**thunk 几乎从不共享。** 每个需要 thunk 的 (基类偏移量, 目标函数地址) 对都生成自己的一份。

例：C::f 同时覆写了三个基类的 f()

```cpp
class A { virtual void f() {} };
class B { virtual void f() {} };
class X { virtual void f() {} };

class C : public A, public B, public X {
    void f() override {}   // 这一个函数，覆写了三个基类的 f
};
```

```
C 对象:
0x1000 ┌──────────┐  ← A 子对象, &C = 0x1000
0x1010 ├──────────┤  ← B 子对象, 偏移 +16
0x1020 ├──────────┤  ← X 子对象, 偏移 +32
       ├──────────┤
       │ C 的东西  │
       └──────────┘
```

三个虚表：

```
A-in-C:     B-in-C:              X-in-C:
┌──────┐    ┌──────────────┐    ┌──────────────┐
│&C::f │    │ thunk_B_f:   │    │ thunk_X_f:   │
│      │    │  sub rdi, 16 │    │  sub rdi, 32 │
│      │    │  jmp C::f    │    │  jmp C::f    │
└──────┘    └──────────────┘    └──────────────┘
```

**两个 thunk，不能合并。** B 和 X 的偏移量不同（16 vs 32），`sub` 的操作数不同，`jmp` 虽然目标相同，但整体指令字节不同。A 不需要 thunk（偏移 = 0）。

换个例子：同一基类，两个不同函数

```cpp
class C : public A, public B {
    void f() override {}   // 覆写 B::f
    void g() override {}   // 覆写 B::g
};
```

```
B-in-C 虚表:
┌──────────────────┐
│ thunk_B_f:       │
│   sub rdi, 16    │
│   jmp C::f       │  ← 目标不同
│ thunk_B_g:       │
│   sub rdi, 16    │
│   jmp C::g       │  ← 目标不同
└──────────────────┘
```

偏移量相同（都是 16），但 `jmp` 目标不同（C::f ≠ C::g），依然是**两个 thunk**。`jmp` 的操作数是绝对地址，编码不同，不能合并。

**真正可能合并的唯一场景**：链接器的 ICF（Identical Code Folding）。如果两个 thunk 的字节完全相同（相同偏移 + 相同目标地址），链接器会在 `--icf=safe` 下合并为一个。这在模板展开产生等价 thunk 时偶尔触发，但常规代码中极少见。

编译器的真实优化策略不是"合并 thunk"，而是**"不需要 thunk 时就不生成"**（偏移为 0 的基类直接放函数地址不带 thunk）。

## Connections

- [[vtable]]: thunk 存储在虚表中，替代真正的函数地址
- [[dynamic-cast]]: 交叉转换时也做类似的 this 指针调整，偏移来源同样是 vtable 中的 offset_to_top
- [[static-cast]]: 多继承下的 static_cast 下行转换也在编译期做类似的指针偏移，但没有运行时检查

## Open Questions

- 协变返回类型（covariant return type）场景下 thunk 如何处理返回值的指针调整？
