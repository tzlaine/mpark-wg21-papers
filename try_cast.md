---
title: "Pattern Matching: Customization Point for Open Sum Types"
document: D3521R0
date: today
audience: Library Evolution
author:
  - name: Michael Park
    email: <mcypark@gmail.com>
toc: true
toc-depth: 4
---

# Introduction

This is a separate paper for a library component that accompanies [@P2688R3].
At the Wrocław meeting in November 2024, the following poll was taken in EWG:

> Poll: [@P2688R3] — Pattern Matching: `match` Expression, we encourage more work
> on the language-only paper towards C++26 in the next meeting (note: voting
> against this poll does not exclude getting pattern matching in C++29)
>
> Result: SF: 17, F: 16, N: 6, A: 1, SA: 9

This paper proposes a library component that interact with the language feature.
Specifically, it proposes to add a customization point for a user-defined type
to opt in as an open sum type.

# Motivation and Scope

[@P2688R3] proposes an alternative pattern which has syntax like the following:

```cpp
int f(const std::variant<int, double>& v) {
  return v match {
    int: let i => i;
    double: let d => int(d);
  };
}
```

This uses the "variant-like" protocol which uses the existing set of variant
helpers: `variant_size`, `variant_alternative`, `get`, and `index`.

For open sum types however, there is no such existing facilities. `std::any` and 
`std::exception_ptr` are examples of such types.

::: cmptable

### C++23

```cpp
int f(const std::any& a) {
  if (auto* i = std::any_cast<int>(&a)) {
    return *i;
  } else if (auto* d = std::any_cast<double>(&a)) {
    return *d;
  } else {
    return -1;
  }
}
```

### P2688 with This Paper

```cpp
int f(const std::any& a) {
  return a match -> int {
    int: let i => i;
    double: let d => d;
    _ => -1;
  };
}
```

:::

[@P2927R2] mentions a desired pattern matching use case for `std::exception_ptr`.
The example is written with [@P1371R3], but the following is what it would look
like with [@P2688R3]:

::: cmptable

### C++23

```cpp
void f(const std::exception_ptr& eptr) {
  if (auto* e = std::exception_ptr_cast<logic_error>(eptr)) {
    // use `*e`
  } else if (auto* e = std::exception_ptr_cast<exception>(eptr)) {
    // use `*e`
  } else if (ep == nullptr) {
    std::print("no exception");
  } else {
    std::print("some other exception");
  }
}
```

### P2688 with This Paper

```cpp
void f(const std::exception_ptr& eptr) {
  eptr match {
    logic_error: let e => // use `e`
    exception: let e => // use `e`
    nullptr => std::print("no exception");
    _ => std::print("some other exception");
  };
}
```

:::

This paper is to introduce a `try_cast` customization point that pattern matching
will invoke to test and extract the state of open sum types.

# Design Overview

The proposed approach is to add overloads of `try_cast` like so:

```cpp
namespace std {
  template <typename T>
  const T* try_cast(const any* a) noexcept { return any_cast<T>(a); }

  template <typename T>
  T* try_cast(any* a) noexcept { return any_cast<T>(a); }

  template <typename T>
  const T* try_cast(const exception_ptr* p) noexcept {
    return exception_ptr_cast<T>(*p); // P2927R2
  }
}
```

The parameters are taken by pointer instead of references due to `any`
being able to be implicitly converted from virtually anything.

This approach also allows for user-defined types to add its own overload
without having to open the `std` namespace:

```cpp
namespace ns {
  struct Widget { /* ... */ };

  template <typename T>
  const T* try_cast(const Widget* w) noexcept {
    return // ...
  };
}
```

# Proposed Wording

## [any.synop]{.sref} Header `<any>` synopsis {.unnumbered .unlisted}

```diff
  template<class T>
    const T* any_cast(const any* operand) noexcept;
  template<class T>
    T* any_cast(any* operand) noexcept;

+ template<class T>
+   const T* try_cast(const any* operand) noexcept;
+ template<class T>
+   T* try_cast(any* operand) noexcept;
```

## [any.nonmembers]{.sref} Non-member functions {.unnumbered .unlisted}

```diff
+ template<class T>
+   const T* try_cast(const any* operand) noexcept;
+ template<class T>
+   T* try_cast(any* operand) noexcept;
```

::: add
*Mandates*: `is_void_v<T>` is `false`.

*Returns*: `any_cast<T>(operand)`
:::

### [exception.syn]{.sref} Header `<exception>` synopsis {.unnumbered .unlisted}

```diff
  exception_ptr current_exception() noexcept;
  [[noreturn]] void rethrow_exception(exception_ptr p);
  template <class E>
    const E* exception_ptr_cast(const exception_ptr& p) noexcept;
  template <class T> [[noreturn]] void throw_with_nested(T&& t);

+ template<class E>
+   const E* try_cast(const exception_ptr* p) noexcept;
```

### [propagation]{.sref} Exception propagation {.unnumbered .unlisted}

```diff
+ template<class E>
+   const E* try_cast(const exception_ptr* p) noexcept;
```

::: add
*Mandates*: `E` is a *cv*-unqualified complete object type. `E` is not an array type.
            `E` is not a pointer or pointer-to-member type.

*Returns*: `exception_ptr_cast<E>(*p)`
:::

# Design Alternatives

## Overload `any_cast`

Adding new overloads of `any_cast` was considered as a way to opt-in as
an any-like type. Effectively designating variant-like as the closed sum type
opt-in, and any-like as the open sum type opt-in.

The reason this approach is not taken is purely technical. The current overloads
of `any_cast` include versions that takes `any` by reference. Since `any` is
implicitly constructible from anything, testing the validity of `any_cast` is
virtually always true.

## `std::hash` Like Approach

This approach is to make `try_cast` be a class template that users specialize.

```cpp
namespace std {
  template <typename>
  struct try_cast; // undefined

  template <>
  struct try_cast<std::any> {
    template <typename T>
    static const T* to(const std::any& a) noexcept {
      return std::any_cast<T>(&a);
    }

    template <typename T>
    static T* to(std::any& a) noexcept {
      return std::any_cast<T>(&a);
    }
  };

  template <>
  struct try_cast<std::exception_ptr> {
    template <typename T>
    static const T* to(const std::exception_ptr& p) noexcept {
      return std::exception_ptr_cast<T>(p); // P2927R2
    }
  };
}
```

This allows the parameters to become references again. There are two downsides
of this approach:

  1. The user-defined type needs to re-open the `std` namespace in order to
     provide the specialization.
  2. The API becomes `std::try_cast<From>::to<To>(from)`, compared to
     the proposed API which would be `try_cast<To>(&from)`.