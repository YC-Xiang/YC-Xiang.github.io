# Book notes

**The Adapter Pattern** converts the interface of a class
into another interface the clients expect. Adapter lets
classes work together that couldn’t otherwise because of
incompatible interfaces.

object adpater:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250827224222759.png)

class adpater:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250827224630197.png)

> java 不支持多重继承，所以无法使用 class adpater

# Example

用 java enumeration 实现 iterator 接口，这样在 client code 中可以像使用 iterator 一样使用：

```java
public class EnumerationIterator implements Iterator<Object> {
	Enumeration<?> enumeration;
 
	public EnumerationIterator(Enumeration<?> enumeration) {
		this.enumeration = enumeration;
	}
 
	public boolean hasNext() { // delegate iterator's hasNext() to enumeration hasMoreElements()
		return enumeration.hasMoreElements();
	}
 
	public Object next() { // delegate iterator's next() to enumeration nextElement()
		return enumeration.nextElement();
	}
 
	public void remove() { // enumeration 无法 支持 iterator 的 remove() 操作，所以 throw exception
		throw new UnsupportedOperationException();
	}
}
```

test code:

```java
public class EnumerationIteratorTestDrive {
	public static void main (String args[]) {
		Vector<String> v = new Vector<String>(Arrays.asList(args));
		Iterator<?> iterator = new EnumerationIterator(v.elements());
		while (iterator.hasNext()) {
			System.out.println(iterator.next());
		}
	}
}
```
