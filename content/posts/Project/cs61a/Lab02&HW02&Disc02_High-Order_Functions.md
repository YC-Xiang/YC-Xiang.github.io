# Lab

## Q1 Short Circuiting

```py
>>> True and 13
13
>>> False or 0
0
>>> not 10
False
>>> print(3) or "" # 首先会打印3，再打印“”
3
""
```

## Q2 High-order Function

```py
def cake():
    print('beets')
    def pie():
        print('sweets')
        return 'cake'
    return pie

>>> chocolate = cake() # chocolate等于cake()的返回值即pie
beets # 调用cake() print打印出来的
>>> chocolate
<function cake.<locals>.pie at 0x10a0faca0>
>>> chocolate() # pie()
sweets
'cake'
>>> more_chocolate, more_cake = chocolate(), cake
sweets
>>> more_choclate # more_chocolate即chocolate()的返回值，即pie()的返回值
'cake'
```

## Q3 Lambda

```py
>>> (lambda: 3)() # 返回3，()后面紧跟在 lambda 表达式后面，表示立即执行这个匿名函数

>>> higher_order_lambda = lambda f: lambda x: f(x)
>>> g = lambda x: x * x
>>> higher_order_lambda(g)(2) # 注意传入顺序，先传g，再传2
4
```

## Q4 Composite Identity Function

返回一个 function，该 function 根据传入的值，判断 f(g(x))和 g(f(x))的值是否一致，一致返回 True, 不一致返回 False。

```py
def composite_identity(f, g):
    def identify(x):
        if f(g(x)) == g(f(x)):
            return True
        else:
            return False
    return identify
```

或者

```py
def composite_identity(f, g):
    return lambda x: f(g(x)) == g(f(x))
```

# HW
