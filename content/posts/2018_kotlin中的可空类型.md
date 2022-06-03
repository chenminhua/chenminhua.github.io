---
title: "我只是想要一个函数啊"
date: 2018-03-23T20:00:08+08:00
---

## kotlin 中的可空类型

最近在公司使用 Kotlin 写后端服务（spring + mybatis），遇到 optional 的小问题，如下

```kt
fun getProject(projectCode: String): ProjectDO = projectMapper.getProjectByCode(projectCode)
```

上面这个方法根据项目号去数据库拿项目，从类型签名上看，getProject 方法返回的是一个 ProjectDO 类型的对象，而且由于不是可空类型，所以不会是 null。但是，如果数据库里面确实没有这个项目号的项目呢？试验一下

```kt
print(getProject("no such project"))   // null
```

返回结果是 null!!

kotlin 欺骗了我们，明明说好是非空，怎么就返回了一个 null 给我!!

其实这锅 kotlin 真不背。了解 mybatis 的同学应该知道，mybatis 采用的是 jdk 代理的模式来代理 Mapper 接口。尽管我们在定义 mapper 接口时写明了返回非空类型的对象，

```kt
projectMapper.getProjectByCode(projectCode): ProjectDO
```

但是在 jvm 上运行的时候，运行时类型可没有什么非空不非空的。查不到数据，mybatis 的 mapper 的代理对象就返回了一个 null 给调用者，调用的地方拿这个对象赋值也好，返回也好，也不存在可空类型，拿到 null 就是 null 了。

所以 kotlin 的可空类型，只能在编译时帮你搞定一些程序员搞出来的空指针异常。而在一些使用了代理技术的地方，运行时还是会给你跑出一个 null 来，仍然需要你手动处理。

by the way，如果你阅读 kotlin 代码编译后的字节码，会发现其实 kotlin 在所有不是可空的变量上加上了@NotNull 注解，其 RetentionPolicy 为 class

```java
/**
  * Annotations are to be recorded in the class file by the compiler
  * but need not be retained by the VM at run time.  This is the default
  * behavior.
  */
CLASS,
```

说白了，它可不会在运行时起作用。
