---
slug: spring-transaction-propagation-characteristics
title: Spring 事务传播特性
authors: [solidSpoon]
tags: []
---

# 背景

## 问题
在项目中写出了如下模式的代码

```java
@Override  
@Transactional  
public void parent() {
    // 期望：parent() 不回滚
    balabalaService.child();  
}  
  
@Override    
public void child() {  
    try {  
        balabalaService.grandChild();  
    } catch (Exception ignore) {  
        // 忽略异常  
    }  
}  
  
@Override  
@Transactional  
public void grandChild() {  
    // 通过抛出异常回滚当前事务  
    throw new RuntimeException("grandChild");  
}
```

上面的代码在 `parent()` 方法中通过 `child()` 调用了 `grandChild()` ，期望 `grandChild()` 回滚时 `parent()` 不会回滚。

这段代码实际上是不会按照预期工作的，`parent()` 方法也会跟着回滚。

## 解释

当 `grandChild()` 抛出异常时，会将当前事务标记为回滚，虽然 `child()` 中捕获了异常，看似 `parent()` 不会因为异常而回滚，但由于事务的传播特性，现在 `grandChild()` 与 `parent()` 处于一个事务中，因此实际上是 `parent()` 的事务被 `grandChild()` 标记为了回滚，导致 `parent()` 发生回滚。

# 解决

## NESTED

既然 `parent()` 和 `grandChild()` 两个方法处在一个事务中，我就想能不能在 `child()` 方法上新建一个嵌套事务，这样 `grandChild()` 便与 `child()` 处于同一个事物，因此 `grandChild()` 回滚时就不会导致 `parent()` 回滚。

```java
@Override  
@Transactional  
public void parent() {
    // 期望：parent() 不回滚
    balabalaService.child();  
}  
  
@Override
// 开启嵌套事务
@Transactional(propagation = Propagation.NESTED)
public void child() {  
    try {  
        balabalaService.grandChild();  
    } catch (Exception ignore) {  
        // 忽略异常  
    }  
}  
  
@Override  
@Transactional  
public void grandChild() {  
    // 通过抛出异常回滚当前事务  
    throw new RuntimeException("grandChild");  
} 
```

这段代码看起来没啥问题，`grandChild()` 与  `child()` 处于同一个嵌套事务中，嵌套事务的回滚不会影响外层事务的回滚，同时又在 `child()` 捕获了所有的异常，因此外部事物也不会因为接收到异常而回滚，然而事实也不是这样的。

`grandChild()` 方法的事务传播特性为默认值 `REQUIRED` ，他的特性之一是「支持当前事务」，那么当前事务是什么呢？

通过[[Spring Boot 打印事务日志]]等方式发现当前事务是 `parent()` 方法的事务，也就是说 `Propagation.NESTED` 方式创建的事务不是真正的事务，实际上他只是 MySQL 中的一个「savepoint」，导致 `grandChild()` 仍然与  `child()` 处在同一个事物中。

> The SAVEPOINT in MySQL is used for dividing (or) breaking a transaction into multiple units so that the user has a chance of roll backing the transaction up to a specified point. That means using Save Point we can roll back a part of a transaction instead of the entire transaction.

可见：**NESTED 中调用带事务的方法可能导致外层事务回滚**

## REQUIRES_NEW

解决办法就是使用 `Propagation.REQUIRES_NEW` 创建一个真正的事务。

```java
@Override  
@Transactional  
public void parent() {
    // 期望：parent() 不回滚
    balabalaService.child();  
}  
  
@Override
// 开启嵌套事务
@Transactional(propagation = Propagation.Propagation.REQUIRES_NEW)
public void child() {  
    try {  
        balabalaService.grandChild();  
    } catch (Exception ignore) {  
        // 忽略异常  
    }  
}  
  
@Override  
@Transactional  
public void grandChild() {  
    // 通过抛出异常回滚当前事务  
    throw new RuntimeException("grandChild");  
} 
```

这种情况下，Spring 会暂停当前的事务链接，使用一个新的链接启动一个新的事务，也就是说 `REQUIRES_NEW` 的事务跟普通的事务是完全一样的。

## Catch Exception

此时再运行代码，会发现 `child()` 方法抛出了一个异常，描述为「Transaction rolled back because it has been marked as rollback-only」，很明显我们已经在 `child()` 中捕获了所有的异常，那这个异常就不是我们抛出的。由此得知，当事务被标记为 rollback-only 的时候，Spring 会在事务的方法上抛出一个异常。

```java
@Override  
@Transactional  
public void parent() {
    // 期望：parent() 不回滚
    try {  
        balabalaService.child();  
    } catch (Exception e) {  
        e.printStackTrace();  
    }
}  
  
@Override
// 开启嵌套事务
@Transactional(propagation = Propagation.Propagation.REQUIRES_NEW)
public void child() {  
    try {  
        balabalaService.grandChild();  
    } catch (Exception ignore) {  
        // 忽略异常  
    }  
}  
  
@Override  
@Transactional  
public void grandChild() {  
    // 通过抛出异常回滚当前事务  
    throw new RuntimeException("grandChild");  
} 
```

在 `parent()` 方法中捕获异常后，这段代码终于工作了。
