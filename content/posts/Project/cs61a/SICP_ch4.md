# Chapter 4 Data Processing

## 4.2 Implicit Sequences

Sequence 可以在使用时才分配内存, 比如下面的 range(), 只有在使用时才分配内存, 而不是在定义时分配内存.

```python
>>> r = range(10000, 1000000000)
>>> r[45006230]
45016230
```

## 4.2.1 Iterators

迭代器.

```python
>>> primes = [2, 3, 5, 7]
>>> type(primes)
<class 'list'>
>>> iterator = iter(primes)
>>> type(iterator)
<class 'list_iterator'>
>>> next(iterator)
2
>>> next(iterator)
3
>>> next(iterator)
5
```

当 next()到序列的最后一个元素之后, 会抛出 StopIteration 异常. 可以通过 try 来 catch 这个异常.

```python
>>> next(iterator)
7
>>> next(iterator)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
>>> try:
	next(iterator)
    except StopIteration:
	print('No more values')
No more values
```

每次调用 next(), 迭代器都会维护一个内部状态, 这个状态会记录当前迭代器的位置.

```python
>>> r = range(3, 13)
>>> s = iter(r)  # 1st iterator over r
>>> next(s)
3
>>> next(s)
4
>>> t = iter(r)  # 2nd iterator over r
>>> next(t)
3
>>> next(t)
4
>>> u = t        # Alternate name for the 2nd iterator
>>> next(u)
5
>>> next(u)
6
```

对迭代器继续调用 iter() 会返回迭代器本身.

```python
>>> v = iter(t)  # Another alterante name for the 2nd iterator
>>> next(v)
8
>>> next(u)
9
>>> next(t)
10
```

## 4.2.2 Iterables

可以生成迭代器的的序列称作 iterable value.

strings, tuples, sets, dictionaries 都是 iterable value.

```python
>>> d = {'one': 1, 'two': 2, 'three': 3}
>>> d
{'one': 1, 'three': 3, 'two': 2}
>>> k = iter(d)
>>> next(k)
'one'
>>> next(k)
'three'
>>> v = iter(d.values())
>>> next(v)
1
>>> next(v)
3
```

在 Sequence 更改内容后, 前面创建的迭代器会失效.

```python
>>> d.pop('two')
2
>>> next(k)

RuntimeError: dictionary changed size during iteration
Traceback (most recent call last):
```

## 4.2.3 Build-in Iterators

map, filter, zip, reversed 这些内置函数都会返回一个迭代器.

`map(function, iterable)`, 返回一个迭代器, 这个迭代器会调用 function 对 iterable 中的每个元素进行处理.

```python
>>> def double_and_print(x):
	print('***', x, '=>', 2*x, '***')
	return 2*x
>>> s = range(3, 7)
>>> doubled = map(double_and_print, s)  # double_and_print not yet called
>>> next(doubled)                       # double_and_print called once
*** 3 => 6 ***
6
>>> next(doubled)                       # double_and_print called again
*** 4 => 8 ***
8
>>> list(doubled)                       # double_and_print called twice more
*** 5 => 10 ***
*** 6 => 12 ***
[10, 12]
```

`filter(function, iterable)`, 返回一个迭代器, 这个迭代器会调用 function 对 iterable 中的每个元素进行处理, 并返回处理结果为 True 的元素.

`zip(iterable1, iterable2, ...)`, 返回一个迭代器, 这个迭代器将 iterable1 和 iterable2 中的元素进行一一配对打包成 tuple.

`reversed(sequence)`, 返回一个迭代器, 这个迭代器会返回 sequence 中的元素, 但是顺序是反的.

### 4.2.4 For Statements

for 语句的实现就是通过迭代器来遍历. Objects 实现 `__iter__` 方法返回迭代器, 再通过实现`__next__`方法来遍历.

```python
>>> counts = [1, 2, 3]
>>> for item in counts:
	print(item)
1
2
3
```

相当于

```python
>>> items = counts.__iter__()
>>> try:
	while True:
	    item = items.__next__()
	    print(item)
    except StopIteration:
	pass
1
2
3
```

### 4.2.5 Generators and Yield Statements

生成器 generator 使我们能够定义更复杂的迭代, generator 是迭代器的一种.

生成器函数 generator function 是定义生成器的一种方式. 生成器函数使用 yield 语句来返回值, 而不是 return.

对于 generator,不需要实现`__iter__`和`__next__`方法, 但可以使用`__next__`方法来遍历.

```python
>>> def letters_generator():
	current = 'a'
	while current <= 'd':
	    yield current
	    current = chr(ord(current)+1)
>>> for letter in letters_generator():
	print(letter)
a
b
c
d

>>> letters = letters_generator()
>>> type(letters)
<class 'generator'>
>>> letters.__next__()
'a'
>>> letters.__next__()
'b'
>>> letters.__next__()
'c'
>>> letters.__next__()
'd'
>>> letters.__next__()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

### 4.2.6 Iterable Interface

实现 object 的 `__iter__` 方法.

```python
>>> class Letters:
	def __init__(self, start='a', end='e'):
	    self.start = start
	    self.end = end
	def __iter__(self):
	    return LetterIter(self.start, self.end)

>>> b_to_k = Letters('b', 'k')
>>> first_iterator = b_to_k.__iter__()
>>> next(first_iterator)
'b'
>>> next(first_iterator)
'c'
>>> second_iterator = iter(b_to_k)
>>> second_iterator.__next__()
'b'
>>> first_iterator.__next__()
'd'
>>> first_iterator.__next__()
'e'
>>> second_iterator.__next__()
'c'
>>> second_iterator.__next__()
'd'
```

### 4.2.7 Creating Iterables with Yield

### 4.2.8 Iterator Interface

实现 object 的 `__next__` 方法。
