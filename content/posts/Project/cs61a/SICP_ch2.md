## 2.3 Sequences

## 2.3.1 Lists

```py
>>> digits = [1, 8, 2, 7]
>>> len(digits) # 返回list长度
4
>>> digits[3]
8
>>> digits[-1] # 最后一个元素
7
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

## 2.3.2 Sequence Iteration

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

## 2.3.3 Sequence Processing

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

## 2.3.4 Sequence Abstraction

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

## 2.3.5 Strings

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
