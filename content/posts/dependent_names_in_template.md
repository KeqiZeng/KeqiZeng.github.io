---
date: '2024-12-10T12:25:13+01:00'
title: 'Dependent Names in Template'
tags: [C++]
draft: false
---

## Definitions

A name is called a **dependent name** if it depends on a template parameter in any of these ways:

- The type of a type template parameter
- The value of a non-type template parameter
- The template of a template template parameter

Names that do not depend on any template parameters are called **non-dependent names**.

## Lookup and binding rules

**Dependent names** are looked up and bound at the point of template instantiation.

**Non-dependent names** are looked up and bound at the point of template definition. This binding holds even if a better match name becomes available at the point of template instantiation.

```cpp
#include <iostream>

void foo(double) { // [5]
    std::cout << "::foo(double)\n";
}

template <typename T>
struct X {
    void foo(double) {
        std::cout << "X<T>::foo(double)\n";
    }
    void foo(int) {
        std::cout << "X<T>::foo(int)\n";
    }
};

template <typename T>
struct Y : X<T> {
    void f_double() {
        this->foo(1.0); // [1]
    }
    void f_int() {
        this->foo(1); // [2]
    }
    void g_double() {
        foo(1.0); // [3]
    }
    void g_int() {
        foo(1); // [4]
    }
};

void foo(int) { // [6]
    std::cout << "::foo(int)\n";
}

int main() {
    Y<int> y;
    y.f_double();
    y.f_int();
    y.g_double();
    y.g_int();
    return 0;
}
```

At points `[1]` and `[2]`, `this->foo(1.0)` and `this->foo(1)` are dependent names because `this` depends on template `X`. The name lookup occurs during Y's template instantiation point. At this point, the member functions `foo` and their overloads from `X` become visible, and the best match is selected based on overload resolution rules.

At points `[3]` and `[4]`, `foo(1.0)` and `foo(1)` are non-dependent names since they do not depend on any template parameters. The name lookup happens at Y's template definition point. At this point, the member functions `foo` are not visible in either `Y` or `X`. Only the function `foo(double)` at point `[5]` is visible and will be bound. Even though a better matching function `foo(int)` at point `[6]` becomes visible during template instantiation, the binding of `foo(1)` at point `[4]` remains unchanged.

Output:

```
X<T>::foo(double)
X<T>::foo(int)
::foo(double)
::foo(double)
```

## Dependent type names

A dependent name can refer to not only functions or member functions but also types. A dependent type name can be ambiguous since the compiler cannot determine whether it refers to a type without additional context. In such cases, the `typename` keyword is used to explicitly indicate that the name refers to a type.

```cpp
template <typename T>
struct X {
    using type = T;
};

template <typename T>
struct Y : X <T> {
    void f() {
        type x; // [1]
        X<T>::type y; // [2]
        typename X<T>::type z; // [3]
    }
};

int main() {
    Y<int> y;
    return 0;
}
```

The code above will fail to compile for the following reasons:

- At point `[1]`, `type` is a non-dependent name. The name lookup is performed at Y's template definition point, where `type` is not yet known as a type name.
- At point `[2]`, `X<T>::type` is a dependent type name and becomes visible at Y's template instantiation point. However, without `typename`, the compiler cannot determine if it refers to a type or a static member.

The correct approach is to use the `typename` keyword as shown at point `[3]`.

In C++20, the `typename` keyword can be implicitly deduced by the compiler in the following contexts:

1. Using declarations
2. Data member declarations
3. Function parameter declarations or definitions
4. Trailing return types
5. Default arguments of template type parameters
6. Type-id in `static_cast`, `const_cast`, `reinterpret_cast`, and `dynamic_cast` expressions

```cpp
#include <iostream>

template <typename T>
struct X {
    using type = T;
};

template <typename T>
struct Y : X <T> {
    using type_y = X<T>::type; // 1
    X<T>::type v{}; // 2

    template <typename U = X<T>::type> // 5
    auto foo(U u, X<T>::type) -> X<T>::type { // 3 4
        return static_cast<X<T>::type>(u); // 6
    }
};

int main() {
    Y<int> y;
    auto res = y.foo(1.2, 1);
    std::cout << "res = " << res << std::endl;
    return 0;
}
```

Output:

```
res = 1
```

The improvements in C++20 simplify the writing of dependent type names. However, it’s important to note that this may cause compatibility issues with older compilers, especially in projects that need to support standards prior to C++20.

## Dependent template names

A dependent name can also refer to a template, such as a function template or a class template. By default, the compiler interprets dependent names as non-template entities. To explicitly indicate that a dependent name refers to a template, the `template` keyword must be used. This keyword can only appear immediately after a scope resolution operator `::` or member access operators `->` and `.`.

```cpp
template <typename T>
struct X {
    template <typename U>
    void foo() {}
};

template <typename T>
void bar() {
    X<T> x;
    x.foo<T>(); // Error: '<' is parsed as less-than operator,
                // because the compiler does not yet know that foo is a template.

    x.template foo<T>(); // OK: explicitly indicates foo is a template
};

int main() {
    bar<int>();
    return 0;
}
```

Dependent template name can also refer to a class template.

```cpp
template <typename T>
struct X {
    template <typename U>
    struct Y {};
};

template <typename T>
void bar() {
    using type0 = X<T>::Y<T>; // Without the template keyword, the compiler cannot
                              // disambiguate Y as a template or a non-template member.

    using type1 = X<T>::template Y<T>; // OK: Y is explicitly marked as a template
};

int main() {
    bar<int>();
    return 0;
}
```

## References

- [Dependent names](https://en.cppreference.com/w/cpp/language/dependent_name)
- [Template metaprogramming with C++](https://www.amazon.com/Template-Metaprogramming-everything-templates-metaprogramming/dp/1803243457)
- [现代C++模板教程](https://mq-b.github.io/Modern-Cpp-templates-tutorial/md/%E7%AC%AC%E4%B8%80%E9%83%A8%E5%88%86-%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/09%E5%BE%85%E5%86%B3%E5%90%8D)
