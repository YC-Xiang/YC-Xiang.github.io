# VSCode 配置 python

# Lab00

`python3 ok --help`: 查看 ok 提示。

`python3 ok -q <file> -u`: 运行单个 tests 目录下的测试。

`python3 ok -q <function>`: 运行单个函数的测试。

`python3 ok`: 运行全部测试。

`python3 ok --score`:查看分数。

`python3 ok --local`:本地运行测试。

`python3 -m doctest lab00.py`: 运行 doctest。

</br>

`python3 -i xxx.py`: 进入 python 交互模式

`python3 -m doctest xxx.py`: 运行函数的 doctest，如下，会运行>>>后的函数，并与 2024 对比。如果 pass 的话不会有任何输出。

```py
def twenty_twenty_four():
    """Come up with the most creative expression that evaluates to 2024
    using only numbers and the +, *, and - operators.

    >>> twenty_twenty_four()
    2024
    """
    return 2024
```

# Lab01

`print("DEBUG:", x)`: 带 DEBUG 的 print 语句不会影响到测试
