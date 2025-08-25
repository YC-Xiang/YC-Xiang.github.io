# Book notes

**The Singleton Pattern** ensures a class has only one
instance, and provides a global point of access to it.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250825215111352.png)

- Java 的单例模式实现使用 private constructor，static method combine with static variable。
- 为多线程应用程序仔细选择合适的单例实现
- 可以使用 Java 的 enum 来简化单例实现。

# example

```java
public class Singleton {
	private static Singleton uniqueInstance;

	private Singleton() {} // 私有构造函数，防止外部实例化
 
	public static synchronized Singleton getInstance() { // synchronized 关键字确保线程安全
		if (uniqueInstance == null) {
			uniqueInstance = new Singleton();
		}
		return uniqueInstance;
	}
 
	public String getDescription() {
		return "I'm a thread safe Singleton!";
	}
}
```

```java
public class SingletonClient {
	public static void main(String[] args) {
		Singleton singleton = Singleton.getInstance(); // 通过静态方法获取单例实例
		System.out.println(singleton.getDescription());
	}
}
```
