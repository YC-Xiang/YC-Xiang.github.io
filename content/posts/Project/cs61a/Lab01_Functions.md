# Lab1 Functions, Control

## Quiz

```py
>>> def how_big(x):
...     if x > 10:
...         print('huge')
...     elif x > 5:
...         return 'big'
...     elif x > 0:
...         print('small')
...     else:
...         print("nothing")
>>> how_big(7)
? big
-- Not quite. Try again! --

? 'big' #
-- OK! --

>>> how_big(12)
? 'huge'
-- Not quite. Try again! --

? huge
-- OK! --

>>> def how_big(x):
...     if x > 10:
...         print('huge')
...     elif x > 5:
...         return 'big'
...     if x > 0:
...         print('positive')
...     else:
...         print(0)

>>> print(how_big(12))
(line 1)? huge
(line 2)? positive
(line 3)?
-- Not quite. Try again! --

(line 1)? huge
(line 2)? positive
(line 3)? None
-- OK! --

>>> print(how_big(1), how_big(0))
(line 1)? positive
(line 2)? 0
(line 3)? None, None
-- Not quite. Try again! --

(line 1)? positive
(line 2)? 0
(line 3)? None None
-- OK! --
```

```py
>>> True and 13
? True
-- Not quite. Try again! --

? 1
-- Not quite. Try again! --

? 13
-- OK! --

>>> False or 0
? False
-- Not quite. Try again! --

? 0
-- OK! --
```

Print with 'DEBUG:' at the front of the outputted line 用来 Debug，ok autograder 也不会计入。
比如：`print('DEBUG:', res)`
