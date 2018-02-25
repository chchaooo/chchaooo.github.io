---
layout:     post
title:      "GreenDao_4_Sessions"
subtitle:   "项目中需要使用到GreenDao,于是将官方文档看了一遍，留了一个翻译。由于非常关心该数据库的新能表现，因此写了一个性能测试demo，也放到了github上"
date:       2018-02-27 20:30:00
author:     "chchaooo"
header-img: "img/main_banner.jpg"
header-mask: 0.3
catalog:    true
tags:
    - 数据库
    - GreenDao
---

DaoSession类（生成的）是greenDAO核心接口之一。开始，DaoSession向开发者提供访问基本的实体操作和DAOs(提供更多完整操作)。Session也管理了实体的作用域。
          
### DaoMaster and DaoSession

创建DaoSession对象之前，需要创建一个Daomaster对象

```java
daoMaster = new DaoMaster(db);
daoSession = daoMaster.newSession();
```
数据库连接属于DaoMaster。
所以通过一个DaoMaster获取到的多个sessions依赖相同的数据库连接。新的seesions可以被快速创建。每个session需要分配内存，为每个实体指定一个session cache。
          
### Identity scope and session “cache”（作用域和会话“cache”）

如果两个查询操作返回相同的数据库对象，需要有多少个Java对象存在;一个还是两个？这需要依赖作用域。

greenDAO默认多个查询操作返回相同的Java对象（这个默认行为是配置好的）。例如，根据ID加载USER表中的一个User对象42次，每次查询将会返回相同的Java对象。

这是“caching”实体发挥的作用。如果一个实体始终在内存中（greenDAO这里使用弱引用），实体将不会从数据库中取值进行构造。例如，如果你通过它的ID加载一个实体，并且之前你已经加载了实体，greenDAO就不需要再次查询数据库。代替的是，它将会立即返回来自session cache中的对象，速度会快一到两倍。
          
### 清除作用域
如何清除整个session的作用域，保证不会返回cashing Objects(直接从数据库中查询)

```java
daoSession.clear();
```
如果只需要清除一个Dao的作用域

```java
noteDao = daoSession.getNoteDao();
noteDao.detachAll();
```


          Concepts（概念）
          这页文档将会在将来拓展。现在，请参考Hibernate's session to grasp the concept of sessions and identity scopes(http://docs.jboss.org/hibernate/core/3.6/reference/en-US/html/transactions.html#transactions-basics-uow)。



