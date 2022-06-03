---
title: "Haystack(facebook是怎么存照片的)"
date: 2019-12-25T20:00:08+08:00
---

本文写于 21 世纪 10 年代最后一个圣诞节的晚上，内容为 facebook 的论文 《Finding a needle in Haystack: Facebook’s photo storage》的阅读笔记。该论文旨在解决社交网络中海量小文件且请求具有长尾特性的服务能力问题。

简单来说，传统文件系统会给每个文件建立 metadata，比如 inode 这种数据结构，在访问文件的时候，需要先从磁盘找到 inode，然后从 inode 里面查到文件真正的位置，再去拿文件，于是 metadata lookup 就成了瓶颈。而 haystack 所做的就是尽可能减少磁盘访问。

当然，那文件的那次磁盘 IO 是跑不掉的，那有没有可能省掉 metadata lookup 这一此磁盘 IO 呢？有点难，因为要处理海量的小文件，他们的 metadata 也是非常非常大的，内存肯定放不下。所以思路就到了怎么压缩 metadata，以使其能够放入内存中去。

具体是怎么做的呢？卖个关子，去我的知乎专栏看吧~ https://zhuanlan.zhihu.com/p/99388774
