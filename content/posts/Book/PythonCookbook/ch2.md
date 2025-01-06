## 2.1 使用多个分隔符分割字符串

`re.split()`

```python
>>> line = 'asdf fjdk; afed, fjek,asdf, foo'
>>> import re
>>> re.split(r'[;,\s]\s*', line)
['asdf', 'fjdk', 'afed', 'fjek', 'asdf', 'foo']
```

`r'[;,\s]\s*'` 是一个正则表达式, 表示分号/逗号/空格, 后面还可以跟任意数量空格, 都表示分隔符.

## 2.2 字符串开头或结尾匹配

`str.startswith()`和`str.endswith()`

```python
>>> filename = 'spam.txt'
>>> filename.endswith('.txt')
True
>>> filename.startswith('file:')
False
>>> url = 'http://www.python.org'
>>> url.startswith('http:')
True
>>>
```

注意这两个方法匹配多种类型只接受元组, 比如`name.endswith(('.c', '.h'))`.

## 2.3 用 shell 通配符匹配字符串

fnmatch 模块的`fnmatch()`和`fnmatchcase()`

```python
>>> from fnmatch import fnmatch, fnmatchcase
>>> fnmatch('foo.txt', '*.txt')
True
>>> fnmatch('foo.txt', '?oo.txt')
True
>>> fnmatch('Dat45.csv', 'Dat[0-9]*')
True
>>> names = ['Dat1.csv', 'Dat2.csv', 'config.ini', 'foo.py']
>>> [name for name in names if fnmatch(name, 'Dat*.csv')]
['Dat1.csv', 'Dat2.csv']
>>>
```

fnmatch()函数匹配能力介于简单的字符串方法和强大的正则表达式之间.
如果在数据处理操作中只需要简单的通配符就能完成的时候, 这通常是一个比较合理的方案.

## 2.4 字符串匹配和搜索

re 模块和正则表达式.

```python
>>> text1 = '11/27/2012'
>>> import re
>>> # Simple matching: \d+ means match one or more digits
>>> if re.match(r'\d+/\d+/\d+', text1):
... print('yes')
... else:
... print('no')
...
yes
```

如果想使用同一个模式做多次匹配, 应该先将模式字符串预编译为模式对象.

```python
>>> datepat = re.compile(r'\d+/\d+/\d+')
>>> if datepat.match(text1):
... print('yes')
... else:
... print('no')
...
yes
```

// TODO: 还有更多关于 re 模块正则匹配的内容

## 2.5 字符串搜索和替换

简单的使用`str.replace()`, 复杂的正则替换用`re.sub()`.

## 2.6 字符串忽略大小写的搜索替换

在使用 re 模块的时候, 加上`flags=re.IGNORECASE`

## 2.7 最短匹配模式

通过在 `*` 或者 `+` 这样的操作符后面添加一个 `?` 可以强制匹配算法改成寻找最短的可能匹配。

## 2.8 多行匹配模式
