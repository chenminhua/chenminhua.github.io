---
title: "feature vs function"
date: 2018-05-10T20:00:08+08:00
---

最近看到一段关于微服务的视频：[拆分单体应用](https://www.youtube.com/watch?v=k9QZ4oIOHnk&list=WL&index=3)，其中有一段话引起了我的注意：**split code by feature not by functionality**。

大多数项目在一开始往往都只有一个后端服务。但是随着团队的扩展，业务需求的增加，单服务的可扩展性问题就会开始困扰你。如何拆分服务就成为了一个重要的问题。

关于上面那段话：split code by feature not by functionality。我查了一些资料，自己也做了一些思考，但是还是感觉很模糊。[stackexchange 上的这个问题](https://softwareengineering.stackexchange.com/questions/94164/feature-vs-function)甚至被直接关闭了，理由是这个问题的答案很主观。

在日常的开发中，我们也常常会混用这两个词，加个 feature 和加个功能听起来是一回事，如果不分语境地强行去区分这两者，感觉没什么意义。但是在服务拆分的语境下面，区分这两个词或许是有意义的，甚至是有必要的。

function 更接近于描述我们想要的功能，比如记账的功能，汇率转换的功能，聊天的功能。而 feature 更接近于我们实现功能的方式，比如自动记账，手动记账，扫码记账。function 更为抽象，而 feature 更为具体。function 更稳定，而 feature 更易变。
