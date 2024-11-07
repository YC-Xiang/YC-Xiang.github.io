# Building Abstractions with Data

## 2.1 Introduction

### 2.1.1 Native Data Types

Python includes three native numeric types: integers (int), real numbers (float), and complex numbers (complex).

```py
>>> type(2)
<class 'int'>
>>> type(1.5)
<class 'float'>
>>> type(1+1j)
<class 'complex'>
```

## 2.2 Data Abstraction

## 2.3 Sequences

### 2.3.1 Lists

```py
>>> digits = [1, 8, 2, 7]
>>> len(digits) # 返回list长度
4
>>> digits[3]
8
>>> digits[-1] # 最后一个元素
7

# 这样也可以取list中的值
>>> digits = [1, 2, 3]
>>> x, y, z = digits # 注意这边必须和list中元素的数量对应上才能unpack
>>> x
1
>>> y
2
>>> z
3
```

List 的加法和乘法：

```py
>>> [2, 7] + digits * 2
[2, 7, 1, 8, 2, 8, 1, 8, 2, 8]
```

List 还可以嵌套：

```py
>>> pairs = [[10, 20], [30, 40]]
>>> pairs[1]
[30, 40]
>>> pairs[1][0]
30
```

### 2.3.2 Sequence Iteration

python for 循环

```txt
for <name> in <expression>:
    <suite>
```

可以方便的遍历 List 中的元素

```py
>>> def count(s, value):
	"""Count the number of occurrences of value in sequence s."""
	total = 0
	for elem in s:
	    if elem == value:
		total = total + 1
	return total

>>> count(digits, 8)
2
```

**Sequence unpacking:**

for 循环中的\<name\>可以是多个元素

```py
>>> pairs = [[1, 2], [2, 2], [2, 3], [4, 4]]
>>> same_count = 0

>>> for x, y in pairs:
	if x == y:
	    same_count = same_count + 1

>>> same_count
2
```

**Ranges:**

```py
>>> range(1, 10)  # Includes 1, but not 10
range(1, 10)

>>> list(range(5, 8))
[5, 6, 7]

>>> list(range(4)) # 如果只给出一个参数，那么就是从0开始
[0, 1, 2, 3]
```

如果 for 循环中的\<name\>在循环中没有用到，那么可以使用`_`代替

```py
>>> for _ in range(3):
	print('Go Bears!')
```

### 2.3.3 Sequence Processing

**List Comprehensions：**

格式为：

```txt
[<map expression> for <name> in <sequence expression> if <filter expression>]
```

比如对 list 中每个元素加 1，可以用这种简单的方法：

```py
>>> odds = [1, 3, 5, 7, 9]
>>> [x+1 for x in odds]
[2, 4, 6, 8, 10]
```

提取某些元素：

```py
>>> [x for x in odds if 25 % x == 0]
[1, 5]
```

**Aggregation：**

将序列中的所有值聚合为单个值

```py
>>> def divisors(n):
	return [1] + [x for x in range(2, n) if n % x == 0]
>>> divisors(4)
[1, 2]
>>> divisors(12)
[1, 2, 3, 4, 6]
```

**Higher-Order Functions:**

```py
>>> def reduce(reduce_fn, s, initial):
	reduced = initial
	for x in s:
	    reduced = reduce_fn(reduced, x)
	return reduced

>>> reduce(mul, [2, 4, 8], 1)
64
```

### 2.3.4 Sequence Abstraction

**membership:**

提供了 in 和 not in 来判断 value 是否存在于 sequence。

```py
>>> digits
[1, 8, 2, 8]
>>> 2 in digits
True
>>> 1828 not in digits
True
```

**slicing:**

```py
>>> digits[0:2]
[1, 8]
>>> digits[1:]
[8, 2, 8]
```

python 切片还有更多的用法。

### 2.3.5 Strings

字符串用单引号或双引号包裹:

```py
>>> 'I am string!'
'I am string!'
>>> "I've got an apostrophe"
"I've got an apostrophe"
>>> '您好'
'您好'
```

String 支持 len()函数和索引:

```py
>>> city = 'Berkeley'
>>> len(city)
8
>>> city[3]
'k'
```

String 也支持加法和乘法操作：

```py
>>> 'Berkeley' + ', CA'
'Berkeley, CA'
>>> 'Shabu ' * 2
'Shabu Shabu '
```

**membership：**

匹配子字符串返回 True, 否则返回 False

```py
>>> 'here' in "Where's Waldo?"
True
```

**Multiline Literals:**

多行字符串使用三重引号

字符串中的\n 换行符属于 1 个元素。

**String Coercion:**

str()强制转换为字符串。

```py
>>> str(2) + ' is an element of ' + str(digits)
'2 is an element of [1, 8, 2, 8]'
```

### 2.3.6 Trees

### 2.3.7 Linked Lists

## 2.4 Mutable Data

### 2.4.2 Sequence Objects

List 支持一系列操作,包括 pop,remove,append,extend,insert 等。

```py
>>> chinese = ['coin', 'string', 'myriad']  # A list literal
>>> suits = chinese                         # Two names refer to the same list

>>> suits.pop()             # Remove and return the final element
'myriad'
>>> suits.remove('string')  # Remove the first element that equals the argument
>>> suits.append('cup')              # Add an element to the end
>>> suits.extend(['sword', 'club'])  # Add all elements of a sequence to the end
>>> suits[2] = 'spade'  # Replace an element
>>> suits
['coin', 'cup', 'spade', 'club']
```

**Sharing and Identity**

is 判断, 类似于指针判断，指向同一个 object 返回 true, 和==判断内容不同。

The former checks for identity, while the latter checks for the equality of contents.

```py
suits = ['heart', 'diamond', 'spade', 'club']
nest[0] = suits

>>> suits is nest[0]
True
>>> suits is ['heart', 'diamond', 'spade', 'club']
False
>>> suits == ['heart', 'diamond', 'spade', 'club']
True
```

**List comprehensions**

List 是 mutable data。

A list comprehension always creates a new list.

```py
>>> from unicodedata import lookup
>>> [lookup('WHITE ' + s.upper() + ' SUIT') for s in suits]
['♡', '♢', '♤', '♧']
```

This resulting list does not share any of its contents with suits

**Tuples**

元组。immutable data。只支持 count, index 操作，改变内容的操作不支持。

```py
>>> code = ("up", "up", "down", "down") + ("left", "right") * 2
>>> len(code)
8
>>> code[3]
'down'
>>> code.count("down")
2
>>> code.index("left")
4
```

### 2.4.3 Dictionaries

key-value 对。key 一般是字符串也可以是任何其他的，可用来索引 value。

一个 key 只能对应一个 value。

```py
>>> numerals = {'I': 1.0, 'V': 5, 'X': 10}
>>> numerals['X']
10
>>> numerals['I'] = 1 # 修改key对应的value
>>> numerals['L'] = 50
>>> numerals
{'I': 1, 'X': 10, 'L': 50, 'V': 5}
```

</br>

字典支持.keys(), values(), items()操作：

```py
>>> d = {2: 4, 'two': ['four'], (1, 1): 4}

>>> for k in d.keys():
...     print(k)
...
2
two
(1, 1)
>>> for v in d.values():
...     print(v)
...
4
['four']
4
>>> for k, v in d.items():
...     print(k, v)
...
2 4
two ['four']
(1, 1) 4
```

</br>

字典还支持 get 操作，传入 key 和 default value, 如果没找到对应的 value, 则返回传入的 default value。

```py
>>> numerals.get('A', 0)
0
>>> numerals.get('V', 0)
5
```

</br>

in 用来判断 key 是否在字典内：

```py
>>> 'two' in d
True
```

</br>

字典也支持 comprehension 操作来创建, 使用大括号。

```py
>>> {x: x*x for x in range(3,6)}
{3: 9, 4: 16, 5: 25}
```

### 2.4.4 Local State

**nonlocal** 关键字，表示该变量是上一层嵌套函数中的变量。

```py
>>> def make_withdraw(balance):
	"""Return a withdraw function that draws down balance with each call."""
	def withdraw(amount):
	    nonlocal balance                 # Declare the name "balance" nonlocal
	    if amount > balance:
		return 'Insufficient funds'
	    balance = balance - amount       # Re-bind the existing balance name
	    return balance
	return withdraw
```

可以看到 balance 是上一层的变量，如果要对其修改，则必须加上 nonlocal 关键字，否则会报错。如果只是打印之类的操作，不对变量修改的话可以直接使用。
