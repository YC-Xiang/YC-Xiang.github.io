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

Python 中的一些对象，如列表和字典，是 Mutable 可变的，这意味着它们的内容或状态可以改变。其他对象，如数字类型、元组和字符串，是 Immutable 不可变的，这意味着它们一旦被创建就不能被更改。

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

## 2.5 Object-Oriented Programming

### 2.5.1 Objects and Classes

Attributes are the data associated with an object.

Methods are the functions associated with an object.

### 2.5.2 Defining Classes

`__init__` 是构造函数, 用于初始化对象的属性。

```python
class Account:
    def __init__(self, account_holder):
        self.balance = 0
        self.holder = account_holder
	def deposit(self, amount):
	    self.balance = self.balance + amount
	    return self.balance
	def withdraw(self, amount):
	    if amount > self.balance:
		return 'Insufficient funds'
	    self.balance = self.balance - amount
	    return self.balance
```

### 2.5.3 Message Passing and Dot Expressions

两种调用方法. 在前一种情况下，必须显式地为 self 形参提供实参。在后一种情况下，self 参数被自动绑定。

```python
>>> spock_account = Account('Spock')
>>> Account.deposit(spock_account, 1001)  # The deposit function takes 2 arguments
1011
>>> spock_account.deposit(1000)           # The deposit method takes 1 argument
2011
```

以下划线开头和结尾的属性是私有的, 不能被外部访问。

### 2.5.4 Class Attributes

Class attributes are attributes that are shared by all instances of a class.

```python
>>> class Account:
	interest = 0.02            # A class attribute
	def __init__(self, account_holder):
	    self.balance = 0
	    self.holder = account_holder
	# Additional methods would be defined here
```

改变 class 的 class attribute 会影响所有实例的属性。

```python
>>> spock_account = Account('Spock')
>>> kirk_account = Account('Kirk')
>>> spock_account.interest
0.02
>>> kirk_account.interest
0.02
>>> Account.interest = 0.04
>>> spock_account.interest
0.04
>>> kirk_account.interest
0.04
```

如果有同名的 instance attribute, 则 instance attribute 比 class attribute 优先级更高。

</br>

改变某个 instance 的 class attribute, 不会影响其他 instance 的 class attribute。

```python
>>> kirk_account.interest = 0.08
>>> kirk_account.interest
0.08
>>> spock_account.interest
0.04
```

```python
>>> Account.interest = 0.05  # changing the class attribute
>>> spock_account.interest     # changes instances without like-named instance attributes
0.05
>>> kirk_account.interest     # but the existing instance attribute is unaffected
0.08
```

因为 kirk_account 在前面赋值了 instance attribute, 所以改变 class 的 class attribute 不会影响 kirk_account 的 instance attribute。

### 2.5.5 Inheritance

子类继承其基类的 attributes, 但可以覆盖某些 attributes, 包括某些 methods。

在子类中未指定的任何内容都会自动假定为与基类一样的行为。

### 2.5.6 Using Inheritance

定义子类时，在类名后加括号，括号内写基类名。

```python
>>> class CheckingAccount(Account):
	"""A bank account that charges for withdrawals."""
	withdraw_charge = 1
	interest = 0.01
	def withdraw(self, amount):
	    return Account.withdraw(self, amount + self.withdraw_charge)
```

覆盖了基类的 withdraw 方法。

### 2.5.7 Multiple Inheritance

Python 支持子类从多个基类继承属性的概念, 称为多重继承.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250110222814.png)

如上图的菱形继承, 从左到右, 从下到上检索 attributes.

### 2.5.8 The Role of Objects

## 2.6 Implementing Classes and Objects

### 2.6.1 Instances

### 2.6.2 Classes

### 2.6.3 Using Implemented Objects

## 2.7 Object Abstraction

### 2.7.1 String Conversion

### 2.7.2 Special Methods

介绍一些 object 的特殊 Methods.

`__init__` : 在 object 创建时自动调用.

`__str__`: 定义 `str(object)` 的行为, 返回 human readable 的字符串.

`__repr__`: 定义`repr(object)` 的行为, 返回 python interpreter readable 的字符串.

`__bool__`: 定义 `bool(object)` 的行为.

`__len__`: 定义 `len(object)` 的行为.

`__getitem__`: 定义 `object[i]`的行为.

`__call__`: 构造 object 可以返回一个函数.

```python
>>> class Adder(object):
	def __init__(self, n):
	    self.n = n
	def __call__(self, k):
	    return self.n + k

>>> add_three_obj = Adder(3)
>>> add_three_obj(4)
7
```

### 2.7.3 Multiple Representations

### 2.7.4 Generic Functions
