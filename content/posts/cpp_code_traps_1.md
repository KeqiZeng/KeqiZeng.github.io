---
date: '2024-12-18T12:04:30+01:00'
title: 'Common C++ Code Traps and How to Avoid Them - Part 1'
tags: [C++]
draft: false
---

```cpp
class A {
public:
    A() = default;
    A(int value) : value_(value) {}

    int value() { return value_; } // [1]

private:
    int value_;
};

int main() {
    A a(42);
    int x = a.value();

    const A b(66);
    int y = b.value(); // [2] Compilation error
    return 0;
}
```

The code above will fail to compile because the member function `value()` at point `[1]` is **non-const**, which means it cannot guarantee that it won't modify the object's state. At point `[2]`, we attempt to call this non-const member function on a **const** object `b`, which is not allowed since const objects can only call const member functions.

To fix this, we need to declare the member function `value()` as const to indicate that it won't modify the object's state:

```cpp
int value() const { return value_; }
```

---

```cpp
class A {
public:
    static const float value = 42.0f; // [1] declaration
};

int main() {
    auto x = A::value;
    auto* p = &A::value;
    return 0;
}
```

The code cannot compile because at point `[1]`, we're attempting to declare and initialize a static const member of type `float` inside the class definition. According to the C++ standard, in-class initialization of constant static members is only allowed for integral types (such as `char`, `bool`, `short`, `int`, `long`, `long long`, and their unsigned variants) or enumeration types.

To fix this, we need to separate the declaration and definition. Declare the constant static member without an initializer inside the class, then define and initialize it in the source file:

```cpp
class A {
public:
    static const float value; // declaration
};

// in source file
const float A::value = 42.0f; // definition
```

---

An important detail about static const integral members: even when declared with an initializer inside the class, you cannot take their address without providing a separate definition. The following code will fail to compile:

```cpp
class A {
public:
    static const int value = 42; // declaration with in-class initializer
};

int main() {
    auto x = A::value;    // OK: can use the value
    auto* p = &A::value;  // Error: no definition available for address-taking
    return 0;
}
```

To enable taking the address of the static member, you need to provide a definition:

```cpp
class A {
public:
    static const int value = 42; // declaration
};

const int A::value; // definition, initializer is not repeated
```

---

Since C++17, we can **define** inline static member and constexpr static member with initializer inside the class, and their addresses can be taken. Here's an example:

```cpp
#include <string>

class A {
public:
    inline static std::string str = "Hello, World!";     // definition with initializer
    constexpr static double value = 42.0;                // definition with initializer, constexpr static member must have a constant initializer
};

int main() {
    auto x = A::str;
    auto* p = &A::str;     // OK: address can be taken
    auto y = A::value;
    auto* q = &A::value;   // OK: address can be taken
    return 0;
}
```
