---
title: java五种单例模式
date: 2023-12-09 20:41:42
tags:
- 单例模式
- 饿汉式
- 懒汉式
- 双重检锁
categories:
- ["编程语言", "JAVA"]
---
> 引自： https://www.cnblogs.com/KylinBlog/p/13932110.html

### 1、饿汉式(线程安全，调用效率高，但是不能延时加载)：
``` Java
public class ImageLoader{ 
     private static ImageLoader instance = new ImageLoader; 
     private ImageLoader(){} 
     public static ImageLoader getInstance(){  
          return instance;  
      } 
}
```

一上来就把单例对象创建出来了，要用的时候直接返回即可，这种可以说是单例模式中最简单的一种实现方式。但是问题也比较明显。单例在还没有使用到的时候，初始化就已经完成了。也就是说，如果程序从头到位都没用使用这个单例的话，单例的对象还是会创建。这就造成了不必要的资源浪费。所以不推荐这种实现方式。

 

### 2.懒汉式(存在线程风险，调用效率不高，但是能延时加载)：

```Java
public class SingletonDemo2 {
     
    //类初始化时，不初始化这个对象(延时加载，真正用的时候再创建)
    private static SingletonDemo2 instance;
     
    //构造器私有化
    private SingletonDemo2(){}
     
    //方法同步，调用效率低
    public static synchronized SingletonDemo2 getInstance(){
        if(instance==null){
            instance=new SingletonDemo2();
        }
        return instance;
    }
}
```
### 3.Double CheckLock实现单例：DCL也就是双重锁判断机制（由于JVM底层模型原因，偶尔会出问题，不建议使用）：
```Java
    public class SingletonDemo5 {
        private volatile static SingletonDemo5 SingletonDemo5;

        private SingletonDemo5() {
        }

        public static SingletonDemo5 newInstance() {
            if (SingletonDemo5 == null) {
                synchronized (SingletonDemo5.class) {
                    if (SingletonDemo5 == null) {
                        SingletonDemo5 = new SingletonDemo5();
                    }
                }
            }
            return SingletonDemo5;
        }
    }
```
### 4.静态内部类实现模式（线程安全，调用效率高，可以延时加载）

```Java
public class SingletonDemo3 {

    private static class SingletonClassInstance{
        private static final SingletonDemo3 instance=new SingletonDemo3();
    }

    private SingletonDemo3(){}

    public static SingletonDemo3 getInstance(){
        return SingletonClassInstance.instance;
    }

}
```
### 5.枚举类（线程安全，调用效率高，不能延时加载，可以天然的防止反射和反序列化调用）

```Java
public enum SingletonDemo4 {

    //枚举元素本身就是单例
    INSTANCE;

    //添加自己需要的操作
    public void singletonOperation(){
    }
}

```


* 如何选用：

-单例对象 占用资源少，不需要延时加载，枚举 好于 饿汉
-单例对象 占用资源多，需要延时加载，静态内部类 好于 懒汉式
