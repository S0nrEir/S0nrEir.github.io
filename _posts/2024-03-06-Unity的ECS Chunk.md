---
title: Unity的ECS Chunk
tags: ["零碎知识"]
---

那你写的ecs有chunk吗

我看et好像也没chunk呢

---

谁写也没chunk啊

---

chunk是啥

---

ecs里面那个概念

dots里边的chunk

---

我就记得那个提高内存命中率相关的

----

不过这个也不是自己写的

这个你要从CPU的组成开始看了

----

chunk是关于component的一个分组

---

提高cpu缓存命中率的

---

chunk 就是一个entity类型

也可以说是component的组合种类

dots性能优化 要避免改变chunk

---

但是一般都是component吧

---

chunk最后的实现就是在内存中开辟的一段区域

---

chunk做好管理就是在创建entity的时候会为这个entity开辟一段连续的内存存放相关的数据

因为内存连续所以缓存命中率就会高

---

为啥component都是struct？因为要计算内存占用

然后每个chunk固定长度，每个entity在chunk位置是不一样的

---

关键就是如何保存操作数据保存在连续内存中

---

改变一个chunk就会导致structrual change

---

你移除entity上面的component就会触发一系列的事件

所以entity一般不做add

好像会重新分配entity在chunk的位置还是啥的->structrual change会重新内存对其。比较耗时 高频操作避免

---

entias就是会预留所有的componet，不涉及到这个。问题是造成大量无效内存，但是简单

反正我看很多ecs都是先申请一大堆内存，

---

对的，就是chunk重排对齐到另外相同的chunk里面取

所以没有chunk操作是不是就不算ecs

---

