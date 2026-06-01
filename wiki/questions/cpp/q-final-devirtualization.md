---
title: "final 关键字的去虚拟化效果"
type: question
created: 2026-05-23
updated: 2026-05-23
sources: ["wiki/concepts/vtable.md"]
questions: [1, 2]
tags: [c++, compiler-optimization, devirtualization, vtable]
---

## 问题

C++11 `final` 关键字告诉编译器"这个类不可再被继承"或"这个虚函数不可再被覆写"。那么编译器真的会利用 `final` 信息**消除间接调用（devirtualization）**，直接生成直接调用吗？效果有多大？

## 快速答案

**会，但看代码形态。** 只在编译器能**在调用点确定接收者的动态类型就是 final 的那个类**时生效。

## 三种能去虚拟化的代码形态

### 1. 直接值对象调用（最直接）

```cpp
class Derived final : public Base {
    void f() override {}
};

Derived d;
d.f();  // ✓ 100% 去虚拟化：d 不可能是任何其他类型，直接调 Derived::f
```

编译器知道 `d` 的静态类型和动态类型完全一致，不需要查 vtable。

### 2. 通过指针/引用但类型是 final 类（中等确定性）

```cpp
Derived d;
Base* bp = &d;
bp->f();  // ? 不一定能去虚拟化 — bp 可能指向其他非 final 子类

Derived* dp = &d;
dp->f();  // ✓ 去虚拟化 — *dp 一定是 Derived，Derived 是 final
```

通过 `Derived*` 或 `Derived&` 调用时，因为 `Derived` 是 final，不存在更派生的类，编译器可以生成直接调用。

### 3. 构造/析构期间（语言保证）

```cpp
struct Base {
    virtual void f() {}
};

struct Derived final : Base {
    Derived() {
        f();   // ✓ 去虚拟化：构造期间 vptr 指向 Derived，且 Derived 是 final
    }
    void f() override {}
};
```

与 `final` 无关（构造期间本身就绑在当前类的 vtable），但 `final` 强化了这一保证。

## 实际验证

```cpp
struct Base { virtual int f() { return 1; } };

struct Derived final : Base {
    int f() override { return 2; }
};

int call_via_ptr(Base* b) {
    return b->f();  // 编译器不知道 b 是否就是 Derived，生成间接调用
}

int call_via_final_ref(Derived& d) {
    return d.f();   // 编译器把 d.f() 直接降到 Derived::f，无间接调用
}
```

在 GCC/Clang 上用 `-O2` 编译，`call_via_final_ref` 会生成：

```asm
; call_via_final_ref(Derived&)
mov  eax, 2      ; 直接返回 2（连调用都没了，函数被 inline 了）
ret
```

而 `call_via_ptr` 仍然是：

```asm
; call_via_ptr(Base*)
mov  rax, QWORD PTR [rdi]   ; 读 vptr
mov  rax, QWORD PTR [rax]    ; 读 vtable[0]
jmp  rax                      ; 间接跳转
```

## 哪些场景 `final` 帮不上忙

**指针/引用的静态类型包含 open hierarchy**

```cpp
struct A final : Base { void f() override {} };
struct NotFinal : Base { void f() override {} };

void foo(Base* b) {
    b->f();  // b 可能指向 A (final) 也可能指向 NotFinal (non-final)
             // 编译器无法确定，依然用 vtable 间接调用
}
```

即使已知 `A` 是 final，从 `Base*` 进入时编译器不确定指向谁，只能查表。

## PGO 的补充作用

PGO（Profile-Guided Optimization）可以弥补 `final` 的不足。在 LTO + PGO 下，如果 99% 的 `b->f()` 调用实际指向 `Derived`，编译器会：

```asm
; 伪代码：speculative devirtualization
mov  rax, [rdi]             ; 读 vptr
cmp  rax, &Derived_vtable   ; 比较 vptr 是否 == Derived 的 vtable
jne  .Lcold_path            ; 不匹配 → 冷路径（间接调用）
call Derived::f             ; 匹配 → 直接调用（最常见路径，被 inline）
```

这是一种"乐观去虚拟化"——先猜一个最可能的类型，用 `cmp + je` 保护，不对再走 vtable。

## Connections

- [[vtable]]: 虚表是虚函数调用的基础，去虚拟化就是绕过它
- [[dynamic-cast]]: 与去虚拟化无关，只在有 RTTI 需求的层次转换中使用
- [[static-cast]]: 下行转换后调用虚函数：`static_cast<Derived*>(b)->f()` — 如果 `b` 确实指向 Derived，也可以去虚拟化，但有 UB 风险
