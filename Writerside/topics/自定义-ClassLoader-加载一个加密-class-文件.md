---
title: 自定义 ClassLoader 加载一个加密 class 文件
date: 2021-03-05 12:49:28
updated: 
tags: 
 - Java
 - JVM
 - 类加载器
categories: 
mathjax: false
---



跟着我体验一下传说中非常厉害的类加载器吧！
## 制作加密 class
### 目标类
我们要加载的类很简单，它只有一个 `hello()` 方法。编译这个类生成 class 文件，待会要用
```java
package ClassLoader;

public class Hello {
    public void hello(){ 
        System.out.println("Hello, classLoader!"); 
    }
    public static void main(String[] args) {
        System.out.println();
    }
}
```
### 加密
下面这段代码读取了刚才生成的 Hello.class ，加密之后保存为 Hello.xlass


`encode()` 实现了一个简单的加密，加载类的时候使用同样的方法就可以解密
```java
package ClassLoader;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;

/**
 * @author : solidSpoon
 * @date : 2021/3/5 1:57
 */
public class EncodeFile {

    public static void main(String[] args) {
        String name = "Hello";
        EncodeFile ef = new EncodeFile();
        byte[] fileByteArray = ef.loadFile(name);
        fileByteArray = ef.encode(fileByteArray);;
        ef.storeFile(fileByteArray, name);
    }
    public byte[] loadFile(String name){
        File f = new File(this.getClass().getResource(name + ".class").getPath());
        byte[] fileByteArray = new byte[(int)f.length()];
        try {
            new FileInputStream(f).read(fileByteArray);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return fileByteArray;
    }
    void storeFile(byte[] fileByteArray, String name) {
        String p = this.getClass().getResource("").getPath();
        File file = new File(p + "/" + name + ".xlass");
        try (FileOutputStream fop = new FileOutputStream(file)) {
            if (!file.exists()) {
                file.createNewFile();
            }
            fop.write(fileByteArray);
            fop.flush();
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
    public byte[] encode (byte[] fileToEncode){
        for(int i=0; i< fileToEncode.length; i++){
            fileToEncode[i] = (byte) (255 - fileToEncode[i]);
        }
        return fileToEncode;
    }
}
```
## 加载
接下来我们定义自己的加载器，把刚才的 xlass 文件解密之后加载到 JVM 中，并反射运行它的 `hello()` 方法。


具体方法是继承 `ClassLoader` 类，覆盖它的 `findClass()` 方法，在该方法中使用 `defineClass()` 将字节流转成 `Class`
```java
package ClassLoader;

import java.io.File;
import java.io.FileInputStream;
import java.lang.reflect.Method;

/**
 * @author : solidSpoon
 * @date : 2021/3/5 1:30
 */

public class MyClassLoader extends ClassLoader{
    public static void main(String[] args) {
        try {
            Class<?> clazz = new  MyClassLoader().findClass("Hello");
            Object obj = clazz.getConstructor().newInstance();
            Method method = clazz.getMethod("hello");
            method.invoke(obj);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        File f = new File(this.getClass().getResource(name + ".xlass").getPath());
        byte[] fileByteArray = new byte[(int)f.length()];
        try {
            new FileInputStream(f).read(fileByteArray);
        } catch (Exception e) {
            e.printStackTrace();
        }
        fileByteArray = decode(fileByteArray);
        String pack = this.getClass().getPackage().getName();
        return defineClass(pack + "." + name, fileByteArray, 0, fileByteArray.length);
    }
    /**
     * 将编码过的字节数组解码
     * @param fileToDecode 要解码的字节数组
     * @return 解码的字节数组
     */
    public byte[] decode (byte[] fileToDecode){
        for(int i=0; i< fileToDecode.length; i++){
            fileToDecode[i] = (byte) (255 - fileToDecode[i]);
        }
        return fileToDecode;
    }
}
```
## 运行结果
我们的类加载器解密了 xlass 并将它加载到了 JVM 中
```java
Hello, classLoader!
```
## 原理
类加载的原则是双亲委派模型：如果一个类加载器收到了类加载的请求，它首先会把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去完成加载。


我们定义的这个 `findClass()` 方法会在下面这个地方调用，如代码所示，如果该类还没有被加载并且父加载器无法加载个类（当然肯定不能加载），就会调用我们定义的 `findClass()` 去加载这个类
```java
// java.lang ClassLoader

	protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```