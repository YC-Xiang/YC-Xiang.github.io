# Book notes

**The Template Method Pattern** defines the skeleton of an algorithm in a method, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm’s structure.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250902215744270.png)

Key points:

- A template method defines the steps of an algorithm, deferring to subclasses for the implementation of those steps.
- The Template Method Pattern gives us an important technique for code reuse
- The template method’s abstract class may define concrete methods, abstract methods, and hooks.
- Abstract methods are implemented by subclasses.
- Hooks are methods that do nothing or default behavior in the abstract class, but may be overridden in the subclass.
- To prevent subclasses from changing the algorithm in the template method, declare the template method as final.
- The Hollywood Principle guides us to put decision making in high level modules that can decide how and when to call low-level modules.
- Factory Method is a specialization of Template Method.

# Example

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250902221925548.png)
