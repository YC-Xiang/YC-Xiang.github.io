# Lab

检查所有得分:

`python3 ok --score --local`

## Q1: WWPD: List-Mutation

`python3 ok -q list-mutation -u --local`

```python
>>> s = [9, 7, 8]
>>> a, b = s, s[:] # s[:] 是 s 的浅拷贝
>>> s = [3]
>>> s.extend([4, 5])
>>> a
[9, 7, 8]
```

注意在 s 重新赋值后, a 和 s 不指向同一位置了.

初始状态：
s ----> [9,7,8]
a ----> [9,7,8] (和 s 指向同一个对象)
b ----> [9,7,8] (独立的副本)

执行 s = [3] 后：
s ----> [3]
a ----> [9,7,8] (保持不变)
b ----> [9,7,8] (保持不变)

执行 s.extend([4,5]) 后：
s ----> [3,4,5]
a ----> [9,7,8] (保持不变)
b ----> [9,7,8] (保持不变)

```python
>>> s = [3,4,5]
>>> s.extend([s.append(9), s.append(10)])
>>> s
[3, 4, 5, 9, 10, None, None]
```

首先会执行 s.append(9), 然后执行 s.append(10), 但是 s.append()返回的是 None, 所以 extend()会接受两个 None, 然后 append 到 s 中.

## Q2: Insert Items

`python3 ok -q insert_items --local`

给一个 list s, 和一个元素 before 和 after, 在 s 中每个 before 后面插入 after.

```python
def insert_items(s, before, after):
    """Insert after into s after each occurrence of before and then return s.

    >>> test_s = [1, 5, 8, 5, 2, 3]
    >>> new_s = insert_items(test_s, 5, 7)
    >>> new_s
    [1, 5, 7, 8, 5, 7, 2, 3]
    >>> test_s
    [1, 5, 7, 8, 5, 7, 2, 3]
    >>> new_s is test_s
    True
    >>> double_s = [1, 2, 1, 2, 3, 3]
    >>> double_s = insert_items(double_s, 3, 4)
    >>> double_s
    [1, 2, 1, 2, 3, 4, 3, 4]
    >>> large_s = [1, 4, 8]
    >>> large_s2 = insert_items(large_s, 4, 4)
    >>> large_s2
    [1, 4, 4, 8]
    >>> large_s3 = insert_items(large_s2, 4, 6)
    >>> large_s3
    [1, 4, 6, 4, 6, 8]
    >>> large_s3 is large_s
    True
    """
    for i in range(len(s) - 1, -1, -1):
	if s[i] == before:
	    s.insert(i + 1, after)
    return s
```

注意这边是从后往前遍历, 因为如果从前往后遍历, 插入元素后, 原来的元素会向后移动,
导致插入的元素会插入到原来的元素后面, 而不是在 before 后面.

## Q3 Group By

给一个 list s, 和一个函数 fn, 返回一个字典, 字典的 key 是 fn(s[i]) 返回的值, value 就是 s[i] 组成的 list.

```python
def group_by(s, fn):
    """Return a dictionary of lists that together contain the elements of s.
    The key for each list is the value that fn returns when called on any of the
    values of that list.

    >>> group_by([12, 23, 14, 45], lambda p: p // 10)
    {1: [12, 14], 2: [23], 4: [45]}
    >>> group_by(range(-3, 4), lambda x: x * x)
    {9: [-3, 3], 4: [-2, 2], 1: [-1, 1], 0: [0]}
    """
    grouped = {}
    for item in s:
	key = fn(item)
	if key in grouped:
	    grouped[key].append(item)
	else:
	    grouped[key] = [item]
    return grouped
```

## Q4 WWPD: Iterators

```python
>>> r = range(6)
>>> r_iter = iter(r)
>>> next(r_iter)
0
>>> [x + 1 for x in r_iter]
[2, 3, 4, 5, 6]
```

迭代器的一个重要特性：迭代器是有状态的，一旦某个值被消耗，就不能重新获取，除非重新创建迭代器。

如果我们没有先调用 next(r_iter)，结果会是 [1, 2, 3, 4, 5, 6]，因为会从 0 开始遍历整个序列。

## Q5 Count Occurrences

给一个迭代器 t, 一个整数 n, 和一个元素 x, 返回 t 中前 n 个元素中 x 出现的次数.

```python
def count_occurrences(t, n, x):
    """Return the number of times that x is equal to one of the
    first n elements of iterator t.

    >>> s = iter([10, 9, 10, 9, 9, 10, 8, 8, 8, 7])
    >>> count_occurrences(s, 10, 9)
    3
    >>> t = iter([10, 9, 10, 9, 9, 10, 8, 8, 8, 7])
    >>> count_occurrences(t, 3, 10)
    2
    >>> u = iter([3, 2, 2, 2, 1, 2, 1, 4, 4, 5, 5, 5])
    >>> count_occurrences(u, 1, 3)  # Only iterate over 3
    1
    >>> count_occurrences(u, 3, 2)  # Only iterate over 2, 2, 2
    3
    >>> list(u)                     # Ensure that the iterator has advanced the right amount
    [1, 2, 1, 4, 4, 5, 5, 5]
    >>> v = iter([4, 1, 6, 6, 7, 7, 6, 6, 2, 2, 2, 5])
    >>> count_occurrences(v, 6, 6)
    2
    """
    count = 0
    for _ in range(n):
	if next(t) == x:
	    count += 1
    return count
```

## Q6 Repeated

给一个迭代器 t, 一个整数 k, 返回 t 中第一个连续出现 k 次的元素.

```python
def repeated(t, k):
    """Return the first value in iterator t that appears k times in a row,
    calling next on t as few times as possible.

    >>> s = iter([10, 9, 10, 9, 9, 10, 8, 8, 8, 7])
    >>> repeated(s, 2)
    9
    >>> t = iter([10, 9, 10, 9, 9, 10, 8, 8, 8, 7])
    >>> repeated(t, 3)
    8
    >>> u = iter([3, 2, 2, 2, 1, 2, 1, 4, 4, 5, 5, 5])
    >>> repeated(u, 3)
    2
    >>> repeated(u, 3)
    5
    >>> v = iter([4, 1, 6, 6, 7, 7, 8, 8, 2, 2, 2, 5])
    >>> repeated(v, 3)
    2
    """
    assert k > 1
    current = next(t)
    count = 1
    while count < k:
	next_item = next(t)
	if next_item == current:
	    count += 1
	else:
	    current = next_item
	    count = 1
    return current
```

# HW

## Q1: Infinite Hailstone

`python3 ok -q hailstone --local`

创建一个生成器, 生成 hailstone 序列.

```python
def hailstone(n):
    """
    Yields the elements of the hailstone sequence starting at n.
    At the end of the sequence, yield 1 infinitely.

    >>> hail_gen = hailstone(10)
    >>> [next(hail_gen) for _ in range(10)]
    [10, 5, 16, 8, 4, 2, 1, 1, 1, 1]
    >>> next(hail_gen)
    1
    """
    if n == 1:
        yield 1
        yield from hailstone(n)
    else:
        yield n
        if n % 2 == 0:
            yield from hailstone(n // 2)
        else:
            yield from hailstone(n * 3 + 1)
```

## Q2 Merge

`python3 ok -q merge --local`

给两个生成器 a 和 b, 返回一个生成器, 生成 a 和 b 的合并序列, 序列中没有重复的元素.

```python
def merge(a, b):
    """
    Return a generator that has all of the elements of generators a and b,
    in increasing order, without duplicates.

    >>> def sequence(start, step):
    ...     while True:
    ...         yield start
    ...         start += step
    >>> a = sequence(2, 3) # 2, 5, 8, 11, 14, ...
    >>> b = sequence(3, 2) # 3, 5, 7, 9, 11, 13, 15, ...
    >>> result = merge(a, b) # 2, 3, 5, 7, 8, 9, 11, 13, 14, 15
    >>> [next(result) for _ in range(10)]
    [2, 3, 5, 7, 8, 9, 11, 13, 14, 15]
    """
    a_val, b_val = next(a), next(b)
    while True:
        if a_val == b_val:
            yield a_val
            a_val, b_val = next(a), next(b)
        elif a_val < b_val:
            yield a_val
            a_val = next(a)
        else:
            yield b_val
            b_val = next(b)
```

## Q3 Stair Ways

`python3 ok -q stair_ways --local`

给一个整数 n, 返回一个生成器, 生成所有可能的爬楼梯的方式, 每次可以爬 1 或 2 级.

```python
def stair_ways(n):
    """
    Yield all the ways to climb a set of n stairs taking
    1 or 2 steps at a time.

    >>> list(stair_ways(0))
    [[]]
    >>> s_w = stair_ways(4)
    >>> sorted([next(s_w) for _ in range(5)])
    [[1, 1, 1, 1], [1, 1, 2], [1, 2, 1], [2, 1, 1], [2, 2]]
    >>> list(s_w) # Ensure you're not yielding extra
    []
    """
    if n == 0:
        yield []
    elif n == 1:
        yield [1]
    else:
        for way in stair_ways(n - 1):
            yield way + [1]
        for way in stair_ways(n - 2):
            yield way + [2]
```

## Q4 Yield Paths

`python3 ok -q yield_paths --local`

给一个树 t, 一个整数 value, 返回一个生成器, 生成所有从根节点到值为 value 的节点的路径.

```python
def yield_paths(t, value):
    """
    Yields all possible paths from the root of t to a node with the label
    value as a list.

    >>> t1 = tree(1, [tree(2, [tree(3), tree(4, [tree(6)]), tree(5)]), tree(5)])
    >>> print_tree(t1)
    1
      2
        3
        4
          6
        5
      5
    >>> next(yield_paths(t1, 6))
    [1, 2, 4, 6]
    >>> path_to_5 = yield_paths(t1, 5)
    >>> sorted(list(path_to_5))
    [[1, 2, 5], [1, 5]]

    >>> t2 = tree(0, [tree(2, [t1])])
    >>> print_tree(t2)
    0
      2
        1
          2
            3
            4
              6
            5
          5
    >>> path_to_2 = yield_paths(t2, 2)
    >>> sorted(list(path_to_2))
    [[0, 2], [0, 2, 1, 2]]
    """
    if label(t) == value:
        yield [label(t)]
    for b in branches(t):
        for way in yield_paths(b, value):
            yield [label(t)] + way
```
