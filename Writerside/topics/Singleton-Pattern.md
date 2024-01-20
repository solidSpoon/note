# Singleton Pattern

Implementing a singleton pattern requires consideration of the following issues

- 构造函数需要是 private 访问权限的，这样才能避免外部通过 new 创建实例；
- 考虑对象创建时的线程安全问题
- 考虑是否支持延迟加载
- 考虑 `getInstance()` 效率是否高（是否加锁）

## Hungry man

Lazy loading is not supported, which is good or bad depending on the scenario.

```java
import java.util.concurrent.atomic.AtomicLong;

public class IdGenerator {
    private AtomicLong id = new AtomicLong(0);
    private static final IdGenerator instance = new IdGenerator();
    private IdGenerator(){}
    public static IdGenerator getInstance() {
        return instance;
    }
    public long getId() {
        return id.incrementAndGet();
    }
}
```

## Lazy initialization

`getInstance()`性能低

```java
import java.util.concurrent.atomic.AtomicLong;

public class IdGenerator {
    private AtomicLong id = new AtomicLong(0);
    private static IdGenerator instance;
    private IdGenerator() {}
    // 方法锁，并发度是 1
    public static synchronized IdGenerator getInstance() {
        if (instance == null) {
            instance = new IdGenerator();
        }
        return instance;
    }
    public long getId() {
        return id.incrementAndGet();
    }
}
```

## Double detection

既支持延迟加载、又支持高并发

```java
import java.util.concurrent.atomic.AtomicLong;

public class IdGenerator {
    private AtomicLong id = new AtomicLong(0);
    private static IdGenerator instance;
    private IdGenerator() {}
    public IdGenerator getInstance() {
        if (instance == null) {
            synchronized (IdGenerator.class) {
                if (instance == null) {
                    instance = new IdGenerator();
                }
            }
        }
        return instance;
    }
    public long getId() {
        return id.incrementAndGet();
    }
}
```

> **传言：**因为指令重排序，可能会导致 IdGenerator 对象被 new 出来，并且赋值给 instance 之后，还没来得及初始化（执行构造函数中的代码逻辑），就被另一个线程使用了。要解决这个问题，我们需要给 instance 成员变量加上 volatile 关键字，禁止指令重排序才行。

> 实际上高版本 Java 已经把对象 new 操作和初始化操作设计为原子操作了

## Static internal class

```java
import java.util.concurrent.atomic.AtomicLong;

public class IdGenerator {
    private AtomicLong id = new AtomicLong(0);
    private IdGenerator() {}
    
    private static class SingletonHolder {
        private static final IdGenerator instance = new IdGenerator();
    }
    
    public static IdGenerator getInstance() {
        return SingletonHolder.instance;
    }
    
    public Long getID() {
        return id.incrementAndGet();
    }
}
```

只有当调用 `getInstance()` 方法时，`SingletonHolder` 才会被加载，这个时候才会创建 instance。

## Enumeration static internal classes

```java
import java.util.concurrent.atomic.AtomicLong;

public enum IdGenerator {
    INSTANCE;
    private  AtomicLong id = new AtomicLong(0);
    
    public Long getId() {
        return id.incrementAndGet();
    }
}
```

## 使用序列化与反序列化破坏单例模式

[掘金](https://juejin.cn/post/6844903745772339214)

除了上文中提到的方法，也可以使用枚举方式来防止破坏
