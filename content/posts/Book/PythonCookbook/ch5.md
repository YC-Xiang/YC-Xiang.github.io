## 5.1 读写文本数据

读 `rt`

```python
# Read the entire file as a single string
with open('somefile.txt', 'rt') as f:
    data = f.read()

# Iterate over the lines of the file
with open('somefile.txt', 'rt') as f:
    for line in f:
	# process line
	...
```

写 `wt`

```python
# Write chunks of text data
with open('somefile.txt', 'wt') as f:
    f.write(text1)
    f.write(text2)
    ...

# Redirected print statement
with open('somefile.txt', 'wt') as f:
    print(line1, file=f)
    print(line2, file=f)
    ...
```

追加内容 `at`.

可以通过传递 encoding 参数给 open(), 指定文件的编码.

```python
with open('somefile.txt', 'rt', encoding='latin-1') as f:
```

with 控制块结束时，文件会自动关闭. 也可以不使用 with 语句，但是这时候你就必须记得手动关闭文件：

```python
f = open('somefile.txt', 'rt')
data = f.read()
f.close()
```

## 5.2 打印输出至文件中

在 print() 函数中指定 file 关键字参数，像下面这样：

```python
with open('d:/work/test.txt', 'wt') as f:
    print('Hello World!', file=f)
```

## 5.3 使用其他分隔符或行终止符打印

可以使用在 print() 函数中使用 sep 和 end 关键字参数，以你想要的方式输出。比如：

```python
>>> print('ACME', 50, 91.5)
ACME 50 91.5
>>> print('ACME', 50, 91.5, sep=',')
ACME,50,91.5
>>> print('ACME', 50, 91.5, sep=',', end='!!\n')
ACME,50,91.5!!
>>>
```

## 5.4 读写二进制文件

使用模式为 rb 或 wb 的 open() 函数来读取或写入二进制数据。

```python
# Read the entire file as a single byte string
with open('somefile.bin', 'rb') as f:
    data = f.read()

# Write binary data to a file
with open('somefile.bin', 'wb') as f:
    f.write(b'Hello World')
```

// TODO: 更多内容

## 5.5 文件不存在才能写入

默认的`w`模式会覆盖, 使用`x`模式, 文件必须不存在才能写入.
