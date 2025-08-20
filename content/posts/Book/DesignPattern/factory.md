# 简单工厂

## 问题描述

把不同类型的 pizza 的创建过程从 Pizza 类中抽出来封装成 SimplePizzaFactory 类，其他统一的操作放在 Pizza 类中

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250819223820465.png)

# 工厂方法

**The Factory Method Pattern** defines an interface for creating an object, but lets subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses.

工厂方法的目的是允许一个类将实例化延迟到它的子类。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250820222805618.png)

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250820231414699.png)

# 抽象工厂

**The Abstract Factory Pattern** provides an interface for creating families of related or dependent objects without specifying their concrete classes.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250820231314612.png)

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250820231347686.png)

# 工厂方法和抽象工厂对比
