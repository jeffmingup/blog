---
title: "为什么mysql存储要用b+树"
date: 2021-05-27T20:00:56+08:00
draft: false
tags: ["mysql","数据结构"]
---

首先mysql是一个数据库，那数据库是干啥用呢？大家都知道用来存储和查询的。关键字来了“存储”，“查询”，存储包含新增，更新，删除。更新和删除需要先查询相应的数据才能操作，所以查询很重要。那么怎么查询效率才高呢？举个例子斗地主，我们抓到一副牌，要先干嘛？需要先整理一下，整理其实就是排序，因为排序好了我们才更容易找到我们的牌，如果我们没有先整牌，我们每次出牌都要把所有的牌看一下才能找到相应的牌（全表扫描），很麻烦，对全表扫面很麻烦。放在mysql里面也一样，数据存储的时候需要保持顺序。

数据是有序的之后，我们就可以愉快的使用二分法查找数据了，二分法没毛病，但是放到磁盘存储里面有点问题，磁盘访问i o代价是很大的，二分法会导致数据层数比较高（随机磁盘io多），也不能很好的利用磁盘页的特性（以后补充啥是磁盘页）,这个时候就可以使用b树来解决了。b树和二叉树的区别就是不止两个叶子节点。这样可以减少树的高度，可以减少随机磁盘io。

b树可以很好的定位到数据，但是我们通常需要根据条件范围查询很多条数据，b树中当我们需要查询范围数据超过单节点的数据范围的时候，会需要扫描父节点，会出现父节点和子节点反复横跳的情况。这时候我们可以考虑b+树，只有叶子节点才存储数据，非叶子节点只用来索引，叶子节点相互指向，形成链表。这样就可以减少随机磁盘io了。这就是现在数据存储的数据结构大多使用b+树的原因了。



写的可真垃圾，我自己都看不下去了，今天看到一句话“只有写作才能表现出你的思路有多混乱”，差不多是这个意思，我以后要多写了，加油！

