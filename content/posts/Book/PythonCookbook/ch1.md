# Chapter1 Data Structures and Algorithms

## 1.1 Unpacking a Sequence into Separate Variables

任何序列都可以通过赋值操作来分解为单独的变量, 但数量要吻合, 否则会报错.

如果有不需要的成员, 可以使用一些特殊变量名, 比如下面的`_`

```python
>>> data = [ 'ACME', 50, 91.1, (2012, 12, 21) ]
>>> _, shares, price, _ = data
>>> shares
50
>>> price
91.1
```

## 1.2 Unpacking Elements from Iterables of Arbitrary Length

对于不确定个数和任意个数的序列, 可以使用星号来获取其中一段数据.

```python
>>> items = [1, 10, 7, 4, 5, 9]
>>> head, *tail = items
>>> head
1
>>> tail
[10, 7, 4, 5, 9]
>>>
```

## 1.3 Keeping the Last N Items

collections.deque 双端队列适合用来有限的历史记录.

```python
from collections import deque

def search(lines, pattern, history=5):
    previous_lines = deque(maxlen=history)
    for line in lines:
	if pattern in line:
	    yield line, previous_lines # 如果匹配返回(line, previous_lines)
	previous_lines.append(line) # 记录每一行

# Example use on a file
if __name__ == '__main__':
    with open(r'../../cookbook/somefile.txt') as f:
	for line, prevlines in search(f, 'python', 5):
	    for pline in prevlines:
		print(pline, end='')
	    print(line, end='')
	    print('-' * 20)
```

文件中搜索包含特定模式的行, 并打印这些行及其前面的几行历史记录.

yield 定义生成器函数, 在每次迭代时生成一个值并暂停执行, 可以用来生成一个元组 `(line, previous_lines)`

## 1.4 Finding the Largest or Smallest N Items

寻找最大或最小的几个数.

heapq 的 nlargest()和 nsmallest()函数.

还可以指定一个 key 参数, 用于更复杂的数据结构.
