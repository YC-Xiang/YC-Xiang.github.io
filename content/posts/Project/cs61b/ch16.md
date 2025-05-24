# ADTs and BSTs

## 16.1 Abstract Data Types

常见的 ADTs 有：

- Stacks
  - push(int x)
  - pop()
- Lists
  - add(int i)
  - get(int i)
- Sets
  - add(int i)
  - contains(int i)
- Maps
  - put(K key, V value)
  - V get(K key)

## 16.2 Binary Search Trees

二叉搜索树

node 的左边子节点都比 node 小，右边子节点都比 node 大。

```java
private class BST<Key> {
    private Key key;
    private BST left;
    private BST right;

    public BST(Key key, BST left, BST Right) {
        this.key = key;
        this.left = left;
        this.right = right;
    }

    public BST(Key key) {
        this.key = key;
    }
    
    static BST find(BST T, Key sk) {
        if (T == null)
            return null;
        if (sk.equals(T.key))
            return T;
        else if (sk < T.key)
            return find(T.left, sk);
        else
            return find(T.right, sk);
    }

    static BST insert(BST T, Key ik) {
        if (T == null)
            return new BST(ik);
        if (ik < T.key)
            T.left = insert(T.left, ik);
        else if (ik > T.key)
            T.right = insert(T.right, ik);
        return T;
    }
}
```

查找和插入操作比较简单，看代码即可理解。

删除操作需要分三种情况：

1. 要删除的节点没有子节点
2. 要删除的节点有一个子节点
3. 要删除的节点有两个子节点

第一种情况直接删除父节点对该节点的指针，该节点就会被自动回收。

第二种情况将父节点的指针指向要删除节点的子节点。

第三种情况需要使用删除节点左子树中的最大值或者右子树中的最小值替换当前节点。
