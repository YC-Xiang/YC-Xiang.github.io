# Chapter 3: Interpreting Computer Programs

## 3.2 Functional Programming

### 3.2.1 Expressions

scheme 语言的一些语法：

```shell
(quotient 10 2)
5

(+ (* 3 5) (- 10 6))
19

(+ (* 3
      (+ (* 2 4)
         (+ 3 5)))
   (+ (- 10 7)
      6))
57

# if 表达式
(if <predicate> <consequent> <alternative>)

(>= 2 1)
true

# cond 表达式

(cond
    (<p1> <e1>)
    (<p2> <e2>)
    ...
    (<pn> <en>)
    (else <else-expression>))

#t （或 true ）
#f （或 false ）

# 与 或 非，短路性质
(and <e1> ... <en>)
(or <e1> ... <en>)
(not <e>)
```

### 3.2.2 Definitions

绑定 symbol：

```shell
(define pi 3.14)
(* pi 2)
6.28
```

函数定义:

```shell
(define (<name> <formal parameters>) <body>)
(define (square x) (* x x))

(square 21)
441
(square (+ 2 5))
49
(square (square 3))
81
```

```shell
(define (average x y)
  (/ (+ x y) 2))

(average 1 3)
2

(define (abs x)
    (if (< x 0)
        (- x)
        x))

(abs -3)
3
```

匿名函数：

```shell
(lambda (<formal-parameters>) <body>)

(define (plus4 x) (+ x 4))
 # 等价于
(define plus4 (lambda (x) (+ x 4)))
```

### 3.2.3 Compound values

pairs 通过 cons 创建, car 和 cdr 访问：

```shell
(define x (cons 1 2))
x
(1 . 2)
(car x)
1
(cdr x)
2
```

Recursive lists. 特殊值 nil 或 () 表示空列表。

```shell
# 底层实现
(cons 1
      (cons 2
            (cons 3
                  (cons 4 nil))))
(1 2 3 4)
# 定义list
(list 1 2 3 4)
(1 2 3 4)
(define one-through-four (list 1 2 3 4))
(car one-through-four)
1
(cdr one-through-four)
(2 3 4)
(car (cdr one-through-four))
2
(cons 10 one-through-four)
(10 1 2 3 4)
(cons 5 one-through-four)
(5 1 2 3 4)
```

### 3.2.4 Symbolic Data

在符号前面加上单引号，使用符号本身。

```txt
(define a 1)
(define b 2)
(list a b)
(1 2)
(list 'a 'b)
(a b)  （a b）
(list 'a b)
(a 2)  （a 2）
```
