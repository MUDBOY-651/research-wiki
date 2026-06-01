---
title: "std::string_view"
type: concept
created: 2026-05-26
updated: 2026-05-26
sources: ["raw/C++ cook book/语法基础.md"]
questions: [1]
tags: [c++17, performance, memory, string]
---

## Definition

`std::string_view` 是一个**非拥有型**（non-owning）的连续字符序列视图。它不管理底层内存，只持有两个成员：指向字符串起点的指针 + 长度。C++17 引入，位于 `<string_view>`。

```cpp
// 简化实现（示意）
class string_view {
    const char* data_;
    size_t size_;
};
```

## 解决了什么问题

### 经典 `const std::string&` 的问题

```cpp
void print(const std::string& s) {
    std::cout << s << "\n";
}

print("hello");  // 隐式构造 std::string 临时对象，一次堆分配
```

传 `"hello"`（`const char*`）时，编译器构造临时 `std::string`，触发**堆分配 + 拷贝**。即使只是读操作，仍付出分配代价。

```cpp
void print(std::string_view s) {  // 修复：零分配
    std::cout << s << "\n";
}

print("hello");    // 直接指向字面量，无分配
print(some_str);   // 包装已有 string，无拷贝
print("foo"sv);    // 字面量语法，编译期构造
```

### 替代场景一览

| 旧写法 | 新写法 | 收益 |
|--------|--------|------|
| `const std::string&` 参数（只读） | `std::string_view` | 消除 `const char*` 调用时的临时对象构造 |
| `const std::string&` 参数 + `substr()` | `string_view` + `substr()` | O(n) → O(1) |
| 解析字符串（tokenize/parse） | `string_view` 遍历 | 零拷贝 token |

## 关键操作

### substr — O(1)

```cpp
std::string s = "hello world";
std::string_view sv(s);

auto hello = sv.substr(0, 5);   // O(1)，只调整 data_ 指针和 size_，不拷贝
auto world = sv.substr(6);       // O(1)
```

`std::string::substr()` 是 O(n)，返回新分配字符串。`string_view::substr()` 是 O(1)，返回指向同一底层数据的子视图。

### find / rfind / find_first_of — 与 string 同接口

```cpp
auto pos = sv.find(' ');
auto n = sv.find_first_not_of(" \t");
```

### remove_prefix / remove_suffix — 调整视口

```cpp
void ltrim(std::string_view& sv) {
    auto pos = sv.find_first_not_of(" \t\n");
    if (pos != sv.npos) sv.remove_prefix(pos);
}

void rtrim(std::string_view& sv) {
    auto pos = sv.find_last_not_of(" \t\n");
    if (pos != sv.npos) sv.remove_suffix(sv.size() - pos - 1);
}
```

这两个是 `string_view` 独有的——`std::string` 没有对等操作，因为 string 是 owning 的，不能随意收缩。

## 致命陷阱：悬垂引用

`string_view` 不拥有数据，底层数据失效后它就是悬垂指针：

```cpp
// 错误 1：指向临时对象
std::string_view getConfig() {
    std::string result = readFile("config.json");
    return result;  // result 被销毁，返回的 string_view 悬垂
}

// 错误 2：string 扩容后 string_view 失效
std::string s = "hello";
std::string_view sv = s;
s += " world, this is a very long string...";  // 可能触发 reallocation
std::cout << sv;  // UB！底层指针失效

// 错误 3：表达式临时对象
std::string_view sv = std::string("hello") + " world";  // 临时对象亡于行尾
// sv 已经悬垂
```

**规则**：`string_view` 的生命周期必须短于底层数据。用作函数参数是最安全的使用方式——调用者的实参在函数执行期间一定存活。

## 与 std::string 的对比

| | `std::string` | `std::string_view` |
|---|---|---|
| 拥有数据 | 是 | 否 |
| 分配堆内存 | 构造时可能分配 | 从不 |
| `substr` | O(n) | O(1) |
| `c_str()` | O(1)，保证以 '\0' 结尾 | O(1) `data()`，不保证 '\0' 结尾 |
| null-terminated | 保证 | 不保证 |
| mutable | 是 | 否 |

**`data()` 不保证 null-terminated**：`string_view` 可以指向一个字符串的子区间，该区间末尾不一定是 `\0`。传给需要 `const char*` 的 C API 时，要么确保底层是 null-terminated string，要么手动构造 `std::string`。

```cpp
void legacy_api(const char*);  // C API

std::string s = "hello world";
std::string_view sv = s.substr(0, 5);  // "hello"，不是 null-terminated

legacy_api(sv.data());  // 危险！可能读到 "hello\x00" 也可能读越界
legacy_api(std::string(sv).c_str());  // 安全，但有一次拷贝
```

## 为什么不能替代所有 `const std::string&`

1. **需要 null-terminated**：调用 C API 时，`string_view` 不保证，需要构造 string
2. **需要接管生命周期**：存储字段、返回给调用者时，string_view 无法保证底层数据存活
3. **重载冲突**：同时提供 `f(string)` 和 `f(string_view)` 可能产生二义性

典型安全用法：

```cpp
// 参数：用 string_view（只读，短期）
void process(std::string_view input);

// 存储：用 string
struct Config {
    std::string path;  // 需要 owner
    // 不要：std::string_view path;  // 你不知道调用方何时释放
};

// 返回：用 string
std::string format();  // 返回 owner
// 不要：std::string_view format();  // 除非明确指向静态数据
```

## 一个实用的 tokenizer

```cpp
std::vector<std::string_view> split(std::string_view sv, char delim) {
    std::vector<std::string_view> tokens;
    while (!sv.empty()) {
        auto pos = sv.find(delim);
        if (pos == sv.npos) {
            tokens.push_back(sv);
            break;
        }
        tokens.push_back(sv.substr(0, pos));
        sv.remove_prefix(pos + 1);
    }
    return tokens;
}

// tokens 中所有 string_view 都指向原始 sv 的底层数据
// 只要原始数据存活，tokens 就有效
```

零拷贝 tokenize，非常适合配合 `getline`、配置文件解析等场景。

## 面试常见题

### Q1：string_view 内部存了什么？sizeof 多大？

两个成员：`const char*` + `size_t`。不含所有权，不含 capacity。64 位系统上 sizeof 是 16 字节。

追问 **为什么不是三个成员（再加 capacity）？** 因为不拥有数据，没有扩容能力，只描述一块已经存在的内存。

### Q2：这段代码有什么问题？（最常考）

**变体 A — 指向临时对象**
```cpp
std::string_view getExtension(std::string filename) {
    auto pos = filename.rfind('.');
    return std::string_view(filename).substr(pos);  // filename 亡于行尾
}
// 返回的 view 悬垂
```

**变体 B — string 扩容后失效**
```cpp
std::string s = "hello";
std::string_view sv = s;
s += " very long string that causes reallocation";
std::cout << sv;  // UB — view 仍指向旧内存
// 注意：短字符串优化 (SSO) 下可能不 reallocate，行为未定义但可能"碰巧工作"
```

**变体 C — 表达式临时对象**
```cpp
std::string_view sv = std::string("foo") + std::string("bar");
// 临时 std::string 亡于该行末尾，sv 立即悬垂
```

**变体 D — split 返回值的生命周期**
```cpp
auto tokens = split(getUserInput(), ' ');
for (auto t : tokens) std::cout << t;  // UB
// getUserInput 返回的临时 string 已销毁，tokens 全部悬垂
```

核心规则：**`string_view` 的生命周期必须短于底层数据。函数参数是最安全的位置——调用者保证实参在调用期间存活。返回 `string_view` 前必须确认底层数据比返回的 view 活得更长。**

### Q3：为什么有了 `const std::string&` 还需要 `string_view`？

两个原因：

1. **消除隐式临时对象**：`f("hello")` 传 `const std::string&` 会触发堆分配构造临时 string，`string_view` 不会
2. **O(1) substr**：`std::string::substr` 每次都分配新 string（O(n)），`string_view::substr` 只调整指针（O(1)）

### Q4：什么时候不能用 `string_view`？

三个硬约束：

| 场景 | 原因 |
|------|------|
| 需要接管所有权（存储/返回） | 不控制底层生命周期 |
| 需要 null-terminated C 字符串 | `data()` 不保证 `\0` 结尾 |
| 需要修改字符串内容 | 只读视图 |

```cpp
// 错误——存储
struct Config {
    std::string_view path;  // 调用方的 string 释放后 Config 持有悬垂指针
};

// 必须用
struct Config {
    std::string path;  // owner
};

// 错误——给 C API
void legacy(const char*);
std::string s = "hello world";
legacy(std::string_view(s).substr(0, 5).data());  // 可能越界读

// 正确：先构造 string 再取 c_str()
legacy(std::string(string_view(s).substr(0, 5)).c_str());  // 安全，但有一次拷贝
```

### Q5：`string_view` 是零开销抽象吗？有例外吗？

常规路径零开销——只传一个指针+长度。唯一例外是从 `string_view` 转 `const char*` 给 C API：如果 view 只覆盖子区间，不保证 null-terminated，必须构造 `std::string` 来取 `c_str()`，触发一次堆分配。

### Q6：split 返回的 `vector<string_view>` 能用吗？

能用——前提是传入的底层数据比 tokens 活得长。

```cpp
// 安全用法
std::string line;
while (getline(file, line)) {
    auto tokens = split(line, ',');  // tokens 指向 line 的内部 buffer
    process(tokens);                  // 在本轮迭代内使用，安全
}

// 危险用法
auto tokens = split(returns_temp_string(), ',');  // 临时 string 已销毁
```

追问：**如果是 `unordered_map<string, int>` 的 key 做 string_view，rehash 会失效吗？** 不会。unordered_map 用链地址法，节点是堆分配的不移动，rehash 只重排桶指针。但 `vector<string>` 扩容一定失效。

## Connections

- [[dynamic-cast]]: 同属 C++ type system 的编译期/运行期工具，string_view 是 owning vs non-owning 的设计模式
- [[static-cast]]: 与 string_view 的零开销设计哲学一致——编译期解决能解决的问题
