# Hw1 Functions, Control

`python3 ok -q a_plus_abs_b`

`python3 ok -q two_of_three`

返回三个数中最小的两个数。先取 min(i, j)得到较小的一个数，在从 min(max(i, j), k)中得到第二小的数。

```python
def two_of_three(i, j, k):
    return min(i, j)**2 + min(max(i, j), k)**2
```

`python3 ok -q largest_factor`

返回能被整除的最大数。从 1 到 n 遍历，如果 n%i==0 即能够整除，返回最大值。

```python
def largest_factor(n):
    res = 1;
    for i in range(2, n):
	if n % i == 0:
	    res = i
    return res
```

`python3 ok -q hailstone`

按照题意，偶数除以 2，奇数\*3+1 操作，返回操作的次数。

```python
def hailstone(n):
    count = 1
    while (n != 1):
	print(n)
	if (n % 2 == 0):
	    n = n // 2
	else:
	    n = n * 3 + 1
	count += 1
    print(n)
    return count
```
