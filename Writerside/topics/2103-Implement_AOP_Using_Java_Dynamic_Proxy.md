---
title: 使用 Java 的动态代理实现 AOP
date: 2021-03-08 15:09:30
updated: 
tags: 
 - Java
 - AOP
categories: 
mathjax: false
---

Spring 的 AOP 是基于动态代理实现的，本文基于 Java 里的动态代理，实现一个简单的 AOP。
## 要代理的接口
```java
public interface ExampleService {
    /**
     * 打印信息
     */
    void info();
}
```
## 接口的实现
本文就会通过动态代理增加该实例的功能
```java
public class ExampleServiceImpl implements ExampleService {
    @Override
    public void info() {
        System.out.println("Print example info");
    }
}
```
## 实现代理的类
该实例负责实现接口方法的调用
```java
public class ProxyHandler implements InvocationHandler {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        method.invoke(object, args);
        after();
        return null;
    }

    private Object object;

    public ProxyHandler(Object object) {
        this.object = object;
    }

    private void before() {
        System.out.println("proxy before method");
    }

    private void after() {
        System.out.println("proxy after method");
    }
}
```
## 测试代码
主要就是通过 `Proxy.newProxyInstance()` 创建 `interface` 实例，该方法需要 3 个参数：

- 接口类的 ClassLoader
- 一个存放实现的接口的数组
- 处理接口方法调用的 `InvocationHandler` 实例



注意 `Proxy.newProxyInstance()` 返回的是 `Object` ，需要强转成接口
```java
public class DynamicProxyTest {
    public static void main(String[] args) {
        ExampleServiceImpl exampleService = new ExampleServiceImpl();
        ClassLoader classLoader = exampleService.getClass().getClassLoader();
        Class[] interfaces = exampleService.getClass().getInterfaces();
        InvocationHandler handler = new ProxyHandler(exampleService);
        ExampleService proxy = (ExampleService) Proxy.newProxyInstance(classLoader, interfaces, handler);
        proxy.info();
    }
}
```
## 运行结果
运行结果如下，我们的实现类被成功增强了！
```java
proxy before methid
Print example info
proxy after method
```