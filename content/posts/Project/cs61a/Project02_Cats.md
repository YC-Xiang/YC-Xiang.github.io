---
date: 2024-09-29T19:52:53+08:00
title: "CS61A Project2 Cats"
tags:
  - CS61A
categories:
  - Project
---

# Overview

使用如下命令对每个 question 检查正确性：

```sh
python3 ok -q [question number] -i
```

使用 ok 系统允许的 debug 打印：

```py
print("DEBUG:", x)
```

运行 cats GUI 系统：

```py
python3 cats_gui.py
```

# Problem 1

实现 pick 函数，其中函数接受的参数：

paragraphs: 表示 paragraphs 的字符串 list  
select: 一个对字符串的判断函数，return True or False  
k: 第 k 个满足 select 的 paragraphs

返回 paragraphs 中第 k 个满足 select 的字符串，没有满足的返回空字符串。

```py
def pick(paragraphs, select, k):
    """Return the Kth paragraph from PARAGRAPHS for which the SELECT returns True.
    If there are fewer than K such paragraphs, return an empty string.

    Arguments:
	paragraphs: a list of strings representing paragraphs
	select: a function that returns True for paragraphs that meet its criteria
	k: an integer

    >>> ps = ['hi', 'how are you', 'fine']
    >>> s = lambda p: len(p) <= 4
    >>> pick(ps, s, 0)
    'hi'
    >>> pick(ps, s, 1)
    'fine'
    >>> pick(ps, s, 2)
    ''
    """
    for str in paragraphs:
	if select(str) and k == 0:
	    return str
	elif select(str):
	    k -= 1
    return ""
```

`python3 ok -q 01 --local`

# Problem 2

实现 about 函数，其中，

subject: 字符串 list

返回一个函数，该函数也接收一个字符串，判断该字符串中是否有 subject 中的字符串，返回 True/False。不区分大小写，标点符号不考虑，需要全字匹配。

提供了几个 util 函数，split() 用来分割一个字符串成字符串 list。remove_punctuation() 用来去除标点符号。lower()用来大写转小写。

```py
def about(subject):
    """Return a function that takes in a paragraph and returns whether
    that paragraph contains one of the words in SUBJECT.

    Arguments:
	subject: a list of words related to a subject

    >>> about_dogs = about(['dog', 'dogs', 'pup', 'puppy'])
    >>> pick(['Cute Dog!', 'That is a cat.', 'Nice pup!'], about_dogs, 0)
    'Cute Dog!'
    >>> pick(['Cute Dog!', 'That is a cat.', 'Nice pup.'], about_dogs, 1)
    'Nice pup.'
    """
    assert all([lower(x) == x for x in subject]), "subjects should be lowercase."

    def f(paragraph):
	for sub_str in subject:
	    for para_str in split(remove_punctuation(paragraph)):
		if sub_str == lower(para_str):
		    return True
	return False

    return f
```

# Problem 3

实现 accuracy 函数， 计算准确率。

accuracy 函数接收两个字符串， typed 和 source， typed 为用户输入的字符串，source 为对照的字符串，准确率计算规则为：

- 被空格分开的计算为一个单词，一个单词中包括了标点符号
- 如果 typed 比 source 长，那么多出来的单词都算作错误
- 如果 typed 和 source 都为空，那么 accuracy 为 100；如果 typed 为空，source 不为空，那么 accuracy 为 0；如果 typed 不为空，source 为空，那么 accuracy 也为 0

Note: 大小写敏感

```py
def accuracy(typed, source):
    """Return the accuracy (percentage of words typed correctly) of TYPED
    compared to the corresponding words in SOURCE.

    Arguments:
        typed: a string that may contain typos
        source: a model string without errors

    >>> accuracy('Cute Dog!', 'Cute Dog.')
    50.0
    >>> accuracy('A Cute Dog!', 'Cute Dog.')
    0.0
    >>> accuracy('cute Dog.', 'Cute Dog.')
    50.0
    >>> accuracy('Cute Dog. I say!', 'Cute Dog.')
    50.0
    >>> accuracy('Cute', 'Cute Dog.')
    100.0
    >>> accuracy('', 'Cute Dog.')
    0.0
    >>> accuracy('', '')
    100.0
    """
    typed_words = split(typed)
    source_words = split(source)

    if len(typed_words) == 0 and len(source_words) == 0:
        return 100
    elif len(typed_words) == 0 and len(source_words) != 0:
        return 0
    elif len(typed_words) != 0 and len(source_words) == 0:
        return 0

    right = 0

    # 如果两个字符串长度不同，那么比较到短的字符串结束为止
    compare_len = (
        len(typed_words) if len(typed_words) < len(source_words) else len(source_words)
    )

    for i in range(compare_len):
        if typed_words[i] == source_words[i]: # 一个单词匹配，则right++
            right += 1
    return right / len(typed_words) * 100 # 以输入的字符串为基准计算准确率
```

# Problem 4

实现 wpm 函数，计算 words per minute。其中 5 个 character 为一个 word，而不是具体一个单词的长度。

```py
def wpm(typed, elapsed):
    """Return the words-per-minute (WPM) of the TYPED string.

    Arguments:
        typed: an entered string
        elapsed: an amount of time in seconds

    >>> wpm('hello friend hello buddy hello', 15)
    24.0
    >>> wpm('0123456789',60)
    2.0
    """
    assert elapsed > 0, "Elapsed time must be positive"
    return 60 / elapsed * (len(typed) / 5)
```

# Problem 5

```py
def autocorrect(typed_word, word_list, diff_function, limit):
    """Returns the element of WORD_LIST that has the smallest difference
    from TYPED_WORD based on DIFF_FUNCTION. If multiple words are tied for the smallest difference,
    return the one that appears closest to the front of WORD_LIST. If the
    difference is greater than LIMIT, return TYPED_WORD instead.

    Arguments:
        typed_word: a string representing a word that may contain typos
        word_list: a list of strings representing source words
        diff_function: a function quantifying the difference between two words
        limit: a number

    >>> ten_diff = lambda w1, w2, limit: 10 # Always returns 10
    >>> autocorrect("hwllo", ["butter", "hello", "potato"], ten_diff, 20)
    'butter'
    >>> first_diff = lambda w1, w2, limit: (1 if w1[0] != w2[0] else 0) # Checks for matching first char
    >>> autocorrect("tosting", ["testing", "asking", "fasting"], first_diff, 10)
    'testing'
    """

    for str in word_list:
        if typed_word == str:
            return typed_word

    final_diff = diff_function(typed_word, word_list[0], limit)
    final_str = word_list[0]

    for str in word_list:
        diff = diff_function(typed_word, str, limit)
        if diff < final_diff:
            final_diff = diff
            final_str = str

    if limit < final_diff:
        return typed_word
    else:
        return final_str
```
