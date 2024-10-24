# Lab

## Q1 WWPD: Lists & Ranges

`python3 ok -q lists-wwpd -u`

```py
>>> list(range(3, 6))
[3,4,5,6,7,8]

>>> range(3, 6)
range(3,6)

>>> range(4)[-1]
3
```

## Q4 WWPD: List Comprehensions

```py
>>> [[1] + s for s in [[4], [5, 6]]]
[[1, 4], [1, 5, 6]]
```

## Q8 Making Onions

# HW

## Q1 Num Eights

计算 n 中 8 出现的次数。

```py
def num_eights(n):
    """Returns the number of times 8 appears as a digit of n.

    >>> num_eights(3)
    0
    >>> num_eights(8)
    1
    >>> num_eights(88888888)
    8
    >>> num_eights(2638)
    1
    >>> num_eights(86380)
    2
    >>> num_eights(12345)
    0
    >>> num_eights(8782089)
    3
    """
    if n == 0:
	return 0
    return num_eights(n // 10) + (1 if n % 10 == 8 else 0)
```

## Q2 Digit Distance

递归计算 n 各位差的总和。

```py
def digit_distance(n):
    """Determines the digit distance of n.

    >>> digit_distance(3)
    0
    >>> digit_distance(777) # 0 + 0
    0
    >>> digit_distance(314) # 2 + 3
    5
    >>> digit_distance(31415926535) # 2 + 3 + 3 + 4 + ... + 2
    32
    >>> digit_distance(3464660003)  # 1 + 2 + 2 + 2 + ... + 3
    16
    """
    if n < 10:
	return 0
    return digit_distance(n // 10) + abs(n % 10 - n // 10 % 10)
```

## Q3 Interleaved Sum

对奇数计算 odd_func(n), 对偶数计算 even_func(n)，利用递归计算从 0 到 n 的和。

不能使用%来判断 n 是奇数还是偶数。

```py
def interleaved_sum(n, odd_func, even_func):
    """Compute the sum odd_func(1) + even_func(2) + odd_func(3) + ..., up
    to n.

    >>> identity = lambda x: x
    >>> square = lambda x: x * x
    >>> triple = lambda x: x * 3
    >>> interleaved_sum(5, identity, square) # 1   + 2*2 + 3   + 4*4 + 5
    29
    >>> interleaved_sum(5, square, identity) # 1*1 + 2   + 3*3 + 4   + 5*5
    41
    >>> interleaved_sum(4, triple, square)   # 1*3 + 2*2 + 3*3 + 4*4
    32
    >>> interleaved_sum(4, square, triple)   # 1*1 + 2*3 + 3*3 + 4*3
    28
    """

    def sum_odd(k):
	if k > n:
	    return 0
	return sum_odd(k + 2) + odd_func(k)

    def sum_even(k):
	if k > n:
	    return 0
	return sum_even(k + 2) + even_func(k)

    return sum_odd(1) + sum_even(2)
```

因为不能对 n 进行判断奇数还是偶数，所以我们从 1 开始计算，递归到 n。创建两个 helper function，传入 k，计算 k 到 n 的和。

## Q4 Count Dollars

一共有 total 的钱，计算可以有多少种找零方式，面值有 1,5,10,20,50,100。

```py
def count_dollars(total):
    """Return the number of ways to make change.

    >>> count_dollars(15)  # 15 $1 bills, 10 $1 & 1 $5 bills, ... 1 $5 & 1 $10 bills
    6
    >>> count_dollars(10)  # 10 $1 bills, 5 $1 & 1 $5 bills, 2 $5 bills, 10 $1 bills
    4
    >>> count_dollars(20)  # 20 $1 bills, 15 $1 & $5 bills, ... 1 $20 bill
    10
    >>> count_dollars(45)  # How many ways to make change for 45 dollars?
    44
    >>> count_dollars(100) # How many ways to make change for 100 dollars?
    344
    >>> count_dollars(200) # How many ways to make change for 200 dollars?
    3274
    """

```

## Q5 Count Dollars Upward

## Q6 Towers of Hanoi

汉诺塔问题。解法的基本思想是递归。

假设有 A、B、C 三个塔，A 塔有 N 块盘，目标是把这些盘全部移到 C 塔。那么先把 A 塔顶部的 N−1 块盘移动到 B 塔，再把 A 塔剩下的大盘移到 C，最后把 B 塔的 N−1 块盘移到 C。

```py
def move_stack(n, start, end):
    """Print the moves required to move n disks on the start pole to the end
    pole without violating the rules of Towers of Hanoi.

    n -- number of disks
    start -- a pole position, either 1, 2, or 3
    end -- a pole position, either 1, 2, or 3

    There are exactly three poles, and start and end must be different. Assume
    that the start pole has at least n disks of increasing size, and the end
    pole is either empty or has a top disk larger than the top n start disks.

    >>> move_stack(1, 1, 3)
    Move the top disk from rod 1 to rod 3
    >>> move_stack(2, 1, 3)
    Move the top disk from rod 1 to rod 2
    Move the top disk from rod 1 to rod 3
    Move the top disk from rod 2 to rod 3
    >>> move_stack(3, 1, 3)
    Move the top disk from rod 1 to rod 3
    Move the top disk from rod 1 to rod 2
    Move the top disk from rod 3 to rod 2
    Move the top disk from rod 1 to rod 3
    Move the top disk from rod 2 to rod 1
    Move the top disk from rod 2 to rod 3
    Move the top disk from rod 1 to rod 3
    """
    assert 1 <= start <= 3 and 1 <= end <= 3 and start != end, "Bad start/end"
    if n == 1:
	print_move(start, end)
	return
    else:
	mid = 6 - start - end
	move_stack(n - 1, start, mid)
	print_move(start, end)
	move_stack(n - 1, mid, end)
```
