---
title:  C++ Compile Time Computation
author: Ricky Wang
layout: post
tags: [C++]
comments: true
---
This article demonstrates different ways to do compile-time computation.
You may have seen one or more of them in your projects depending on the ages of the code base.

Let's, take Fibonacci sequence as an example. 
A Fibonacci sequence can be defined by the below recursive equation.

```
fib(0) = 0
fib(1) = 1
fib(2) = 1 + 0 = 1
fib(3) = 1 + 1 = 2
...
fib(n) = fib(n-1) + fib(n-2)
```

It is easy to come out with a recursive function to compute the Nth Fibonacci number at run-time.

```cpp
int Fib(int n ) {
  if (n == 0)
    return 0;
  else if (n == 1)
    return 1;
  else
    return Fib(n-1) + Fib(n-2);
}
```

Let's take a look at the above code. It's implemented as a function.
The function does its job well to calculcate the numbers. This is simple and intuitive.
And we expect to do the similar thing for compile time computation.
Actually latest C++ standard supports it as well. But in early days, developers were not that lucky.
They had to play nasty tricks with the class template for it.

To deal with compile time computation, we need a way to declare constants first.
In early days, the only way to create constant named members in class declarations is to use enumeration values.

`FibCls1` is the first attempt to achieve compile time computation.
It works with compilers older than C++98 standard.

```cpp
template<int V>
struct FibCls1 {
  enum { value = FibCls1<V-1>::value + FibCls1<V-2>::value };
};

template<>
struct FibCls1<0> {
  enum { value = 0 };
};

template<>
struct FibCls1<1> {
  enum {value = 1 };
};
```

The C++98 standard introduced the in-class static constant initialization. And we could re-write the class template as follows.

`FibCls2` works with C++98 standard.

```cpp
template<int V>
struct FibCls2 {
  static const int value = FibCls2<V-1>::value + FibCls2<V-2>::value;
};

template<>
struct FibCls2<0> {
  static const int value = 0;
};

template<>
struct FibCls2<1> {
  static const int value = 1;
};
```

The C++11 standard introduced the `constexpr` specifier, which is more accurate specify compile time constant expressions.

`FibCls3` works with a C++11 compliant compiler

```cpp
template<int V>
struct FibCls3 {
  static constexpr int value = FibCls3<V-1>::value + FibCls3<V-2>::value;
};

template<>
struct FibCls3<0> {
  static constexpr int value = 0;
};

template<>
struct FibCls3<1> {
  static constexpr int value = 1;
};
```

With the `constexpr` specifier introduced in C++11, we were able to define `constexpr` functions.

`FibFn1` worked with a C++11 compliant compiler.

**Note** This time, we can use a function template instead of a class template.
It is a big improvement, as it doesn't look right to instantiate a class to just compute numbers.

```cpp
template<int V>
constexpr int FibFn1() {
    return FibFn1<V-1>() + FibFn1<V-2>();
}

template<>
constexpr int FibFn1<0>() {
  return 0;
}

template<>
constexpr int FibFn1<1>() {
  return 1;
}
```

C++17 introduced the `constxpr if` statement, which could be used to select code paths at compile time.

`FibFn2` worked in a C++17 compliant compiler.

```cpp
template<int V>
constexpr int FibFn2() {
  if constexpr (V==0)
    return 0;
  else if constexpr (V==1)
    return 1;
  else
    return FibFn2<V-1>() + FibFn2<V-2>();
}
```

`FibFn3` works with a C++14 compliant compiler.
  * It supports both compile time and run time computation.
  * You may want to try it in a C++11 compliant compiler, but it should not work.
    As only one return statement is supported in a constexpr function in C++11.

**Note**: This time it is not a template any more.
We could finally use function arguments instead of template arguments.
It is now almost the same as the run-time version we wrote at the beginning of this article.
The only difference is the `constexpr` specifier to indicate this function supports compile time computation
when it's parameter is a `constexpr`.
Template solution leads to recursive template instantiation.
To compute `fib<n>`, `fib<n-1>` and `fib<n-2>` must be instantiated.
For this Fibonacci problem, all transitive sub-template would be instantiated until `fib<1>` and `fib<0>`.
In the template solution, more work need to be done by the compiler and requires more CPU and memory.
Also, compilers usually have a maximum recursive instantiation level. The standards recommends 1024 levels.

```cpp
constexpr int FibFn3(int n ) {
  if (n == 0)
    return 0;
  else if (n == 1)
    return 1;
  else
    return FibFn3(n-1) + FibFn3(n-2);
}
```

C++20 introduces another specifier `consteval`, which declares a function can only be evaluated at compile time.
By replacing `constexpr` with `consteval`, we declares a compile time only function.
This avoids any surprises when a function designed for compile time but actually evaluated at run time due to misuse.

`FibFn4` works with a C++20 compliant compiler, such as g++ v10+ and clang++ v10+.

```cpp
consteval int FibFn4(int n ) {
  if (n == 0)
    return 0;
  else if (n == 1)
    return 1;
  else
    return FibFn4(n-1) + FibFn4(n-2);
}
```

We can use below main function to test all the above 7 versions.

```cpp
using namespace std;
int main(int argc, char* argv[]) {
  cout << "FibCls1<6>::value " << FibCls1<6>::value << endl;
  cout << "FibCls2<6>::value " << FibCls2<6>::value << endl;
  cout << "FibCls3<6>::value " << FibCls1<6>::value << endl;

  cout << "FibFn1(6) " << FibFn1<6>() << endl;
  cout << "FibFn2(6) " << FibFn2(6) << endl;
  cout << "FibFn3(6) " << FibFn3(6) << endl;
  cout << "FibFn4(6) " << FibFn4(6) << endl; // requires C++20

  int x;
  cout << "Please input a number: x=";
  cin >> x;
  cout << "FibFn3(x) " << FibFn3(x) << endl;

  // The below line will lead to compile error, because `consteval` disables run time invoke.
  // cout << "FibFn4(x) " << FibFn4(x) << endl
}
```

To build and run the code, we can run any of following commands with clang++ v10+ or g++ v10+.

```shell
$ clang++ --std=c++20 -c CompileTimeComputationFibonacci.cpp -o CompileTimeComputationFibonacci
$ g++ --std=c++20 -c CompileTimeComputationFibonacci.cpp -o CompileTimeComputationFibonacci
```

```shell
$ ./CompileTimeComputationFibonacci                                                                                               ✘ 130 
FibCls1<6>::value 8
FibCls2<6>::value 8
FibCls3<6>::value 8
FibFn1<6>() 8
FibFn2<6>() 8
FibFn3(6) 8
FibFn4(6) 8
Please input a number: x=6
FibFn3(x) 8
```

***Conclusion***

If you work with a compiler compliant with more recent C++ standards, you would have more choices.
I would recommend below principles choosing the solution.
* function solution over class solution
* non-template solution over template solution
* `consteval` over `constexpr`