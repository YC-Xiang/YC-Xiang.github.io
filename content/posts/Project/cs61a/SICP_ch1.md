---
title: SICP_Chapter1 Building Abstractions with Functions
date: 2024-01-22 22:33:28
tags:
  - CS61A
categories:
  - Project
draft: true
---

## 1.4 Designning Functions

### 1.4.1 Documentation

docstring。在函数名后在``````中描述函数信息。第一行用来描述函数，接着空一行，接着可以描述参数等。

help(pressure)可以查看函数帮助信息。

```py
>>> def pressure(v, t, n):
        """Compute the pressure in pascals of an ideal gas.

        Applies the ideal gas law: http://en.wikipedia.org/wiki/Ideal_gas_law

        v -- volume of gas, in cubic meters
        t -- absolute temperature in degrees kelvin
        n -- particles of gas
        """
        k = 1.38e-23  # Boltzmann's constant
        return n * k * t / v
```

### 1.4.2 Default Argument Values

带有默认值的函数形参在调用函数时不赋值表示使用默认值，可以用其他值代替。

## 1.5 Control

### 1.5.4 Conditional Statements

**Boolean contexts.** 除了 0，None，false 其他都为 True。

### 1.5.5 Iteration

`pred, curr = curr, pred + curr` 先计算=右边的表达式，才会更新=左边的值。

## 1.6 Higher-Order Functions

### 1.6.1 Functions as Arguments

函数做形参

### 1.6.3 Nested Definitions

在函数中定义函数

```py
def sqrt(a):
    def sqrt_update(x):
        return average(x, a/x)
    def sqrt_close(x):
        return approx_eq(x * x, a)
    return improve(sqrt_update, sqrt_close)
```

子函数可以调用父函数的形参。

### 1.6.4 Functions as Returned Values

函数做返回值

```py
    def square(x):
        return x * x

    def successor(x):
        return x + 1

    def compose1(f, g):
        def h(x):
            return f(g(x))
        return h

    def f(x):
        """Never called."""
        return -x

    square_successor = compose1(square, successor)
    result = square_successor(12)
```

### 1.6.6 Currying

把有多个参数的函数转化为一系列只有一个参数的函数。

```py
    def curried_pow(x):
        def h(y):
            return pow(x, y)
        return h

>>>curried_pow(2)(3)
8
```

### 1.6.7 Lambda Expressions

匿名函数，只能有一句 return 语句。

```py
def compose1(f, g):
    return lambda x: f(g(x))
```

```py
     lambda            x            :          f(g(x))
"A function that    takes x    and returns     f(g(x))"
```

### 1.6.9 Function Decorators

函数装饰器。

```py
def trace(fn):
    def wrapped(x):
        print('-> ', fn, '(', x, ')')
            return fn(x)
        return wrapped

@trace
    def triple(x):
        return 3 * x

>>> triple(12)
->  <function triple at 0x102a39848> ( 12 )
36
```

相当于

```py
def triple(x):
    return 3 * x

triple = trace(triple)
```

加了装饰器的 triple 函数后，不仅仅只会返回计算得到的 3\*x，还会执行装饰器中的打印语句。
