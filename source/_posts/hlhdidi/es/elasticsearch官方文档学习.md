---
title: elasticsearch官方文档学习
date: 2017-08-08
tags: elasticsearch
---

<!-- MDTOC maxdepth:6 firsth1:1 numbering:0 flatten:0 bullets:1 updateOnSave:1 -->

- [基本概念,入门和安装](#基本概念,入门和安装)
   - [基本概念](#基本概念)
   - [安装](#安装)
   - [基本操作](#基本操作)
<!-- /MDTOC -->

## 基本概念,入门和安装

### 基本概念

#### Cluster
  Cluster是一个结点(Node)的集合,保存了所有数据,并且提供了跨结点之间的索引和搜索.Cluster被一个唯一性的名字所标识,而这个名字的默认值是"elasticsearch",这个名字非常重要,因为当结点加入Cluster的时候是通过名字去搜寻Cluster的
#### Node
  结点(Node)是Cluster的一个独立的服务器,储存数据,并且参与了Cluster的索引过程.结点的默认的名称是UUID生成的.一个结点会被设置去加入到一个Cluster中,在默认情况下,所有结点都会被加入叫做elasticsearch的集群中.
#### Index
  索引(Index)是一些具有相同特征的文档(documents)的集合.例如,可以有个商品目录的索引,也可以有个客户数据的索引.一个索引通过名称标识(注意,这个名称不能包含大写),当对文档进行索引和增删的时候,可以通过索引名称去获得文档对应的索引
#### Type
  Type是索引内部的逻辑分类.这个有点类似于数据库的Table.通常拥有多个公共字段的documents会被定义在一个Type里.Type是索引的一级概念.
#### Document
  文档(Document)是索引的基本单元.例如可以有一个Customer的文档,也可以有一个Products的文档.但是需要注意的是文档必须指定一个Type才可以在索引中使用
#### Shards&Replicas
  索引有时候在存储数据的时候,可能会面临数据量较大,遇到阈值.解决这个问题的方法就是切分索引为一个个单独独立的部分叫做Shards.每一个Shards都是一个独立的索引,并且可以在Cluster的任意节点存储.此外,es允许我们进行对于Shards的备份,称为(Replicas).这样子防止了宕机的危险.当索引创建的时候,可以去指定Shards和Replicas的数量,但是在索引创建后,就只能指定Replicas的数量了.

### 安装
  windows安装:
  下载地址:https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.1.msi
  正常安装,选择do not install as service.配置选择默认的即可,不要安装任何plugins.
  随后进入目录下:.\elasticsearch.exe
  可以指定cluster的名称和Node的名称: .\elasticsearch.exe -Ecluster.name=hlhcluster -Enode.name=hlhnode
  报错的处理:如果报错如下:
```java
java.lang.IllegalStateException: Received message from unsupported version: [2.0.0] minimal compatible version is: [5.0.0]
    at org.elasticsearch.transport.TcpTransport.messageReceived(TcpTransport.java:1236) ~[elasticsearch-5.1.1.jar:5.1.1]
    at org.elasticsearch.transport.netty4.Netty4MessageChannelHandler.channelRead(Netty4MessageChannelHandler.java:74) ~[transport-netty4-5.1.1.jar:5.1.1]
```
  解决方法是检查jdk的版本,需要使用较新的版本才可以使用.
  启动完成后,访问127.0.0.1:9200 发现如下字符串,就安装成功:
```js
{
	"name": "hlhnode",
	"cluster_name": "hlhcluster",
	"cluster_uuid": "O_Je1UUeSb2mmI5C1IKOfQ",
	"version": {
	"number": "5.5.1",
	"build_hash": "19c13d0",
	"build_date": "2017-07-18T20:44:24.823Z",
	"build_snapshot": false,
	"lucene_version": "6.6.0"
	},
	"tagline": "You Know, for Search"
}
```

### 基本操作

* 检测Cluster的生命周期

  可以直接在服务器启动后,在地址栏访问接口_cat/health?v,通过返回值可以确定当前集群的状态,当前节点数量,当前Shards数量.
  可以访问_cat/nodes?v接口,观察当前结点的信息,包括ip地址,版本号,结点名称等

* 查询当前所有的index
  可以访问接口_cat/indices?v,可以看到当前还没有任何Index

* 新增一个Index
  安装kibana插件.kibana插件是用于协同和es工作的插件,可以通过https://www.elastic.co/guide/en/kibana/5.5/windows.html 进行下载.
  解压后,进入bin目录,运行.\kibana.bat 访问http://127.0.0.1:5601 并且打开devtools.
  调用下面的命令新建一个索引,需要注意的是POST创建索引
  ```js
    PUT /customer?pretty
    GET /_cat/indices?v
  ```
