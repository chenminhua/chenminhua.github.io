---
title: "spring aop中遇到的一个小问题"
date: 2018-05-05T20:00:08+08:00
---

最近做的一个项目使用了 spring+mybatis 的技术栈。实现很简单，在数据访问层写一系列 mapper 接口，定义一系列数据查询的方法。在服务启动时，让 spring 去扫这些接口，并为这些接口生成代理对象，也就是 DAO，这些 DAO 会实现 mapper 接口。同时这些 dao 都被 spring IOC 注入到相应的 service 中。

然后有一天，我们遇到了一个需求，某几个 mapper 中的某几个方法需要做一些特殊判断逻辑，并且需要改掉返回的数据。

一种方案是直接去改使用 mapper 的地方，但是以现有的架构来看，mapper 和 service 之间没有新的分层了，这会导致这个奇葩的需求需要写到 service 中的各个角落。如果有一天你想用删掉这些代码，或者修改这部分逻辑，后果不堪设想。

另一种方案是加切面，这样这部分逻辑就被集中到了代码主流程的外部，灵活度更高。

spring aop 有两种常用的给方法加切面的方式。一种是给方法加 annotation，另一种是用 execute()表达式。

起初我的想法是用 annotation 的方式。原因是这样更显式，在以后改代码的时候能够给自己一个提醒，其他伙伴改代码的时候也能知道这个方法会被拦截。于是我就写了个 annotation 加在 mapper 接口的某个方法签名上，然后去拦截这个 annotation 注解的方法。但是，拦截并没有生效。做了一番调查之后，我发现原来 annotation 不能从 interface 继承过来。参考 [stackoverflow](https://stackoverflow.com/questions/4745798/why-java-classes-do-not-inherit-annotations-from-implemented-interfaces)。简单来说就是@Inherited 只能在 superclass 上的注解才能起作用，而在接口上不起作用（不会往上追溯到接口）。这种设计的考虑可能是因为 java 单继承多实现的设计，如果去拿接口的 annotaion 可能会导致冲突。另外要注意的一点是，annotation 的继承并不能在运行时通过反射的方式直接在类上看到，而是通过类继承树向上查找的方式实现的。

如果换成使用 execute()表达式的方法，对既有代码的侵入性更低（有好处也有坏处），也确实可以解决问题。因为在拦截对象方法的时候，不用去查 annotation，而是直接找到实现该接口的指定方法去拦截。
