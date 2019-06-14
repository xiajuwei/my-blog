---
title: 将对象用作Map中的key
date: 2019-05-31 10:03:36
tags:
---
如果将对象作为Map中的key，需要是实现该对象的equals方法和hashCode方法；现在一般通过lombok可以简单得实现，并且可以选择具体需要哪些字段参与equals和hashCode方法的计算。

Java类型系统中分为基础类型和引用类型，引用类型中所有的对象都有一个父类——java.lang.Object。基类Object提供了一些可扩展的方法：equals、hashCode、toString、clone和finalize。开发者在覆盖这些方法的时候，要遵循一定的约定，如果使用不当就会造成bug。

equals 方法
 
如果类有自己的“逻辑相等”概念，而且父类的equals方法又无法满足期望的时候，就应该覆盖equals方法。在开发中我们有时候会将一个自定义的对象作为map中的key，或者将一个自定义的对象加入到集合中，这时候就需要覆盖equals方法。

hashCode方法

覆盖equals方法的时候，要同时覆盖hashCode方法。这里一起看一个案例。假设我定义一个用户信息类，代码如下所示：

![hashCode方法](将对象用作Map中的key\hashCode方法.jpg)

这里使用@EqualsAndHashCode注解生成equals和hashCode方法，并排除了除userId以外的其他字段，表示该用户信息对象的唯一性只跟userId这个字段有关。如果该类是继承了某个自定义的类，需要考虑父类的字段，那么还可以使用@EqualsAndHashCode中的callSuper字段，设置为true就会连父类的字段一起考虑，默认是只考虑当前类中的字段。关注lombok的用法，这里不展开讲了。

假设有一个场景，需要过滤确保某个列表里的用户对象是没有重复的，那么我们就需要确定用户对象的唯一id是什么？在这里是userId，可以使用集合来进行重复对象的过滤，代码如下所示：

![hashCode方法](将对象用作Map中的key\重复集合.jpg)

所有自定义的类都要覆盖toString方法，我会使用lombok的@ToString注解来帮我生成toString方法。使用toString方法可以将对象的字段都以可读的形式展示出来。这样在打印日志的时候，要打印某个对象，就不会打印出一个对象的地址，类似于UserInfo@1768b4，这种展示出来对排查问题一点帮助没有。

clone方法

我在开发中没有用过这个方法。要完成对象的拷贝，只需要区分自己是要深拷贝还是浅拷贝。一般我会使用拷贝构造器或静态工厂方法作为替代方案。

finalize

根据Java文档，finalize方法被设计出来是用于释放非Java资源，但是由于jvm的运行机制导致有很大可能不会调用到对象的finalize方法，或者调用的时机和顺序是不确定的，所以这个方法并没有达到其设计目标。Java9中这个方法已经被废弃了，不过现在很多面试还是会问到这个方法背后的原理，需要理解几个概念：
自定义类的对象，就是我们自定义的类，该类覆盖了finalize方法；
Finalizer对象，在新建一个覆盖了finalize方法的类的对象的时候，就会伴生一个Finalizer对象，并将该对象加入到一个双向列表中；
ReferenceQueuequeue，Finalizer对象创建出来后，就会被加入到这个双向列表；
FinalizerThread对象，Finalizer线程是3号线程，它的作用就是不断从上面哪个队列中取Finalizer对象，然后调用它的runFinalzier方法。

在Java应用中，如果对finalize方法使用不合理，有时候会引发一类问题——存放Finalizer的双向列表过长，导致一些对象的finalze方法调用延迟，如果程序在这个方法中进行了某些对时间敏感的资源的释放，那么就会有问题。

















