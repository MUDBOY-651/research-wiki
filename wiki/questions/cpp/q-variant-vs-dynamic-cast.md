---
title: "C++17 variant + visit 能否替代 dynamic_cast？"
type: question
created: 2026-05-23
updated: 2026-05-23
sources: ["wiki/concepts/dynamic-cast.md", "wiki/concepts/static-cast.md"]
questions: [1]
tags: [c++, c++17, std-variant, polymorphism, dynamic-cast]
---

## 问题

C++17 引入 `std::variant` + `std::visit`，提供了另一种运行时多态的路径——**封闭集合的编译期多态**。而 `dynamic_cast` 是传统的**开放集合的运行时多态**。在项目层面，能否用前者系统性地替代后者？何时该替换，何时不该？

## 两种方案对比

### dynamic_cast 路径（开放集合多态）

```cpp
class Shape {
public:
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

class Circle : public Shape {
    double r;
public:
    Circle(double r_) : r(r_) {}
    double area() const override { return 3.14 * r * r; }
};

class Rect : public Shape {
    double w, h;
public:
    Rect(double w_, double h_) : w(w_), h(h_) {}
    double area() const override { return w * h; }
};

double total_area(const std::vector<Shape*>& shapes) {
    double sum = 0;
    for (auto* s : shapes)
        sum += s->area();
    return sum;
}

// 新增类型不用改任何已有代码 — 继承体系对扩展开放
```

### variant + visit 路径（封闭集合多态）

```cpp
#include <variant>

class Circle {
public:
    double r;
    double area() const { return 3.14 * r * r; }
};

class Rect {
public:
    double w, h;
    double area() const { return w * h; }
};

using Shape = std::variant<Circle, Rect>;

double total_area(const std::vector<Shape>& shapes) {
    double sum = 0;
    for (const auto& s : shapes)
        std::visit([&](const auto& shape) { sum += shape.area(); }, s);
    return sum;
}

// 新增类型需要改 Shape 定义（variant 的类型列表）
```

### 核心差异

| 维度        | dynamic_cast                      | variant + visit                           |
| --------- | --------------------------------- | ----------------------------------------- |
| 类型集合      | 开放（新增子类不改旧代码）                     | 封闭（新增需改 variant 定义 + recompile）           |
| 动态分配      | 通常需要堆分配 + 虚表                      | 值语义，栈上存储，无虚表                              |
| 内存        | 对象分散，cache 不友好                    | 连续内存，cache 友好                             |
| 运行时开销     | vtable 间接调用 + 可能的 dynamic_cast 查表 | `visit` 通常编译为 switch/jump table，无间接调用     |
| 编译时间      | 各类型独立编译，改动局部                      | 变更类型集合需重编译所有调用方                           |
| 二进制大小     | 每类一份 vtable + type_info           | `visit` 为每个调用 lambda × 每个 variant 成员生成实例化 |
| 代码组织      | 易扩展新类型，难加新操作                      | 易加新操作（加 lambda），难加新类型                     |
| 外部扩展      | 库可提供基类，用户自由派生子类                   | 库必须提前列举所有成员类型或使用类型擦除                      |
| this 指针调整 | 多继承需要 thunk                       | 无，值语义无 this 概念                            |

## 取舍分析

### variant 更优的场景

- **类型集合是固定的**：JSON 值类型（object, array, string, number, bool, null）、AST 节点、有限状态机的状态（3-7 个状态不会天天改）
- **热点循环中需要内存局部性**：处理百万级 variant 对象的循环，避免了虚函数调用的 cache miss
- **需要值语义和拷贝安全**：variant 天然是值类型，`=` 就是深拷贝，不存在对象所有权和悬垂指针

### dynamic_cast / 虚函数更优的场景

- **类型集合是开放的**：插件系统、框架/库提供基类供用户派生子类
- **类型层次复杂（多继承/虚继承）**：variant 无法表达继承关系，每个成员是独立类型
- **需要部分共享的实现**：虚函数中复用基类默认实现 + 覆写特定函数很自然，variant 中需要用不同的模式（visitor 内 `if constexpr` 或 type trait）
- **类型数量在编译期不可预测**：variant 是 `union` 的变体，成员数量是编译期常量

### 混合使用

不少项目采用两者混合：
- **core types** 用 variant + visit（最高频、类型集合固定的核心抽象）
- **边界扩展点** 用虚函数/继承（插件、回调、用户扩展点）

例如 LLVM：AST 节点用继承（类型在不断增加），RTTI 以 LLVM 自定义 `dyn_cast` 替代 `dynamic_cast`，某些 IR 操作的值类型用了类似 variant 的机制。

## 补充：Expression Problem 视角

这是典型的 Expression Problem：
- **继承方案（dynamic_cast）**：易加新类型（新子类不改旧代码），难加新操作（需给每个子类加虚函数 → 改接口）
- **variant 方案（visit）**：易加新操作（新 lambda 不改类型定义），难加新类型（需改 variant 定义 + 所有 visit 都要处理新成员）

> 两个方向本质是对偶的——一个在"类型"轴上开放，一个在"操作"轴上开放。选择取决于你的项目在哪个轴上变动更频繁。

## Connections

- [[dynamic-cast]]: dynamic_cast 的运行时查找机制
- [[static-cast]]: 无 RTTI 时的备选方案
- [[conn-cpp-casts]]: C++ 转换的全局视图
