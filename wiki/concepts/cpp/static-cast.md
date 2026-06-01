---
title: "static_cast"
type: concept
created: 2026-05-23
updated: 2026-05-23
sources: ["raw/C++ cook book/语法基础.md"]
questions: [1, 2]
tags: [c++, type-system, compile-time]
---

## Definition

`static_cast` 是 C++ 中最常用的显式类型转换操作符，所有检查在**编译期**完成，零运行时开销。它替代了 C 风格的强制转换 `(type)expr`，提供更明确、更安全的类型转换语义。

```cpp
double d = 3.14;
int i = static_cast<int>(d);  // 显式标量转换
```

## Key Ideas

### 使用场景

**1. 基本类型间的隐式转换（让意图显式化）**

```cpp
int a = 10;
float b = static_cast<float>(a);  // int → float
char c = static_cast<char>(255);  // int → char，可能截断
enum Color { RED, GREEN };
int val = static_cast<int>(RED);  // enum → int
```

**2. 上行转换（Up-cast）：安全，编译期完成**

```cpp
class Base {};
class Derived : public Base {};
Derived d;
Base* bp = static_cast<Base*>(&d);  // 安全，偏移在编译期已知
```

**3. 下行转换（Down-cast）：危险，不做运行时检查**

```cpp
Base* bp = new Derived();
Derived* dp = static_cast<Derived*>(bp);  // 编译通过，但假设 bp 真的指向 Derived
                                           // 如果假设错了 → UB（未定义行为）
Base* bp2 = new Base();
Derived* dp2 = static_cast<Derived*>(bp2); // 编译通过，运行时 UB
```

**4. `void*` 转换**

```cpp
void* vp = some_ptr;
int* ip = static_cast<int*>(vp);  // 从 void* 恢复为具体类型
                                   // 必须保证原始类型是 int*，否则 UB
```

**5. 无法完成的事（编译期错误）**

```cpp
// static_cast 不能移除 const — 必须用 const_cast
const int* cp = ...;
// int* p = static_cast<int*>(cp);  // 编译错误

// static_cast 不能在没有继承关系的类型间转换 — 必须用 reinterpret_cast
class A {};
class B {};
A* a = new A();
// B* b = static_cast<B*>(a);       // 编译错误
```

### static_cast 与 dynamic_cast 对比

| 维度 | static_cast | dynamic_cast |
|------|-------------|--------------|
| 作用阶段 | 编译期 | 运行期 |
| 下行转换安全 | 不安全，类型不匹配则 UB | 安全，失败返回 nullptr / throw |
| 性能 | 零开销 | 有运行时开销（查 RTTI） |
| 依赖 | 无 | 依赖 vtable + type_info（仅多态类型） |
| 上行转换 | 高效 | 效果相同 |
| 交叉转换 | 不支持 | 支持 |

### 为什么 static_cast 比 C 风格转换好

C 风格 `(type)expr` 的问题：语义模糊，它实际执行的是混合操作——可能是 static_cast、const_cast、reinterpret_cast 的组合，取决于上下文：

```cpp
const int* p = ...;
int* q = (int*)p;  // 这是 C 风格转换，做了什么？实际上是 const_cast
                   // 编译器不报错，读代码的人不知道意图
```

static_cast 将转换意图明确化，编译器会拒绝非法的转换。搜索 `static_cast` 可以在大型代码库中精确找到所有转换点，而搜索 `(int)` 会匹配大量无关内容。

### static_cast 与隐式转换的边界

核心原则：**当隐式转换可能造成信息丢失、行为意外、或代码意图模糊时，用 `static_cast` 显式标注。**

**应当显式标注的情况：**

1. 缩窄转换（narrowing conversion，会丢失信息或触发 `-Wconversion`）

```cpp
double d = 3.14;
int i = static_cast<int>(d);     // 明确：截断是故意的
int x = some_int64;              // -Wconversion 警告，应用 static_cast
```

2. 有符号/无符号间转换

```cpp
size_t n = 42;
int i = static_cast<int>(n);     // 明确：我知道 n 在 int 范围内
```

3. enum ↔ 整数

```cpp
enum Color { RED = 0, GREEN = 1 };
int val = static_cast<int>(GREEN);
Color c = static_cast<Color>(1);  // 明确：我知道 1 是有效枚举值
```

4. 下行转换

```cpp
Base* bp = get_widget();
Derived* dp = static_cast<Derived*>(bp);  // 明确：我断言 bp 指向 Derived
```

5. 在重载决议中绑定特定版本

```cpp
void f(int);
void f(double);
f(static_cast<double>(42));  // 明确调用 double 版本
```

**不需要显式标注的情况（隐式就够了）：**

1. 扩宽转换，无信息丢失 — `int → double`、`short → int`
2. 上行转换（upcast）— `Derived* → Base*`，语言语义明确且安全
3. 指针/整数 → bool 在条件中 — `if (ptr)` 所有人理解
4. 基本类型提升 — `short + short → int`，语言自然行为，标 `static_cast` 只会分散注意力

**实用判断心法：**

```
隐式转换:
  编译器不警告  +  无信息丢失风险  +  语义业界共识 → 隐式够了

显式 static_cast:
  编译器警告    +  可能截断/溢出   +  想被 grep 到 → 显式标注
```

项目开启 `-Wconversion` / `-Wsign-conversion` 时，所有缩窄和符号转换都会触发警告，此时 `static_cast` 就是标准做法："我看到了警告，我的选择是接受它。"

## Connections

- [[dynamic-cast]]: 运行时转换，安全但昂贵
- [[rtti]]: dynamic_cast 的依赖，static_cast 不需要
- [[conn-cpp-casts]]: C++ 四种转换操作符的全局对比

## Open Questions

- 在 `-fno-rtti` 的代码库中，如何系统性地保证 static_cast 下行转换的安全性？（编码规范、静态分析、架构约束）
