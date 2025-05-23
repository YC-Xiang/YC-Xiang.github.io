# A Debugging Mystery

这个问题是因为 Java 中 Integer 对象的缓存机制导致的。

在 Java 中，== 运算符比较的是对象的引用（内存地址），而不是对象的值。当你使用 == 比较两个 Integer 对象时，你实际上是在比较它们是否是同一个对象实例。

Java 为了优化性能，对于范围在 -128 到 127 之间的整数，会预先缓存对应的 Integer 对象。这意味着：

- 当你创建值在 -128 到 127 范围内的 Integer 对象时，Java 会返回缓存中的对象
- 当值超出这个范围时（如 128），每次创建都会生成新的对象实例

因此，当你比较两个超出这个范围的 Integer 对象时，它们是不同的对象实例，== 运算符返回 false。


```java
Integer a = 127;
Integer b = 127;
a == b;  // 返回 true，因为 a 和 b 引用同一个缓存对象

Integer c = 128;
Integer d = 128;
c == d;  // 返回 false，因为 c 和 d 是不同的对象实例
```

所以要比较两个integer大小，需要使用equals方法。

```java
public static boolean isSameNumber(Integer a, Integer b) {
    return a.equals(b);
}
```
