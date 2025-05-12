
# Inheritance I: Interface and Implementation Inheritance

## 9.2 Hypernyms, Hyponyms, and the Implements Keyword

interface 类：

```java
public interface List61B<Item> {
    public void addFirst(Item x);
    public void add Last(Item y);
    public Item getFirst();
    public Item getLast();
    public Item removeLast();
    public Item get(int i);
    public void insert(Item x, int position);
    public int size();
}
```

实现类：

```java
public class AList<Item> implements List61B<Item>{...}
```

## 9.3 Overriding, Interface Inheritance

**overriding**

实现类实现 interface 类的时候要加上`@Override`关键字。

```java
@Override
public void addFirst(Item x) {
    insert(x, 0);
}
```

## 9.4 Implementation Inheritance, default

interface 类可以实现默认的 method:

```java
default public void print() {
    for (int i = 0; i < size(); i += 1) {
        System.out.print(get(i) + " ");
    }
    System.out.println();
}
```

对于 SLList 类，这种默认的实现方式效率不高，因为链表每一次 get() 都是 O(n), 可以 override 默认 method:

```java
@Override
public void print() {
    for (Node p = sentinel.next; p != null; p = p.next) {
        System.out.print(p.item + " ");
    }
}
```

## 9.5 Implementation vs. Interface Inheritance
