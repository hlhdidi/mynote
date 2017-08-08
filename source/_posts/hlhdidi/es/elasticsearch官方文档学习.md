---
title: elasticsearch官方文档学习
date: 2017-08-08
tags: elasticsearch
---

<!-- MDTOC maxdepth:6 firsth1:1 numbering:0 flatten:0 bullets:1 updateOnSave:1 -->

- [基本概念,入门和安装](#基本概念,入门和安装)
   - [基本概念](#基本概念)
<!-- /MDTOC -->

## 基本概念,入门和安装

### 基本概念

**Cluster**<br/>
  Cluster是一个结点(Node)的集合,保存了所有数据,并且提供了跨结点之间的索引和搜索.Cluster被一个唯一性的名字所标识,而这个名字的默认值是"elasticsearch",这个名字非常重要,因为当结点加入Cluster的时候是通过名字去搜寻Cluster的
**Node**<br/>
  结点(Node)是Cluster的一个独立的服务器,储存数据,并且参与了Cluster的索引过程.结点的默认的名称是UUID生成的.一个结点会被设置去加入到一个Cluster中,在默认情况下,所有结点都会被加入叫做elasticsearch的集群中.
**Index**<br/>
  索引(Index)是一些具有相同特征的文档(documents)的集合.例如,可以有个商品目录的索引,也可以有个客户数据的索引.一个索引通过名称标识(注意,这个名称不能包含大写),当对文档进行索引和增删的时候,可以通过索引名称去获得文档对应的索引
**Type**<br/>
  Type是索引内部的逻辑分类.这个有点类似于数据库的Table.通常拥有多个公共字段的documents会被定义在一个Type里.Type是索引的一级概念.
**Document**<br/>
  文档(Document)是索引的基本单元.例如可以有一个Customer的文档,也可以有一个Products的文档.但是需要注意的是文档必须指定一个Type才可以在索引中使用
