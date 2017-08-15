---
title: mysql必知必会
date: 2017-07-25
tags: mysql
---

<!-- MDTOC maxdepth:6 firsth1:1 numbering:0 flatten:0 bullets:1 updateOnSave:1 -->

- [mysql必知必会](#mysql必知必会)
   - [数据过滤与搜索](#数据过滤与搜索)
   - [计算和数据处理](#计算和数据处理)
   - [关联查询和联结表](#关联查询和联结表)
   - [操纵数据和操作表](#操纵数据和操作表)
<!-- /MDTOC -->

# mysql必知必会

## 数据过滤与搜索

  * MYSQL支持NOT对于IN,BETWEEN,EXISTS子句取反,这和多数其他DBMS允许NOT对各种条件取反有较大的差别
  * %通配符可以匹配0个,1个或者多个字符,但是需要注意,%是没有办法匹配NULL的.因此WHERE product_name like '%'是没办法
匹配为该字段为NULL的记录
  * 搜索通配符的位置:不要将搜索通配符放在搜索模式的最开始处,放在最开始处,搜索起来是最慢的.
  * MYSQL的正则表达式.
  如下,下面是使用MYSQL的正则表达式的示例:
  ```sql
  select * from products where prod_name REGEXP '1000'
  ```
  可以看出上面的SQL和LIKE语句很像,下面是它的返回值,可以看出是有些不同的.

| prod_id | vend_id | product_name | prod_price |    prod_desc    |
| ------- | ------- | ------------ | ---------- | --------------- |
| JP1000  | 1005    | JetPack 1000 | 35         | JetPack 1000,ir |

  如果只把REGEXP改为LIKE,这样子的返回值会有很大不同.会直接返回空,这是LIKE和REGEXP跟大的区别.LIKE匹配整个列,就算文本
在列值内出现,它也不会直接返回相应的行(除非使用通配符)而REGEXP会在整个列值内进行匹配,如果被匹配的文本在列值内出现对应的
行就会返回,这是很大的区别..想要使其匹配整个列值,可以使用如下方式编写SQL:
```sql
-- 需要注意,^如果出现在[]用于表示否定,其他情况下则用于表示行的开始
select * from products where prod_name REGEXP '^1000$'
```
  正则表达式的OR操作:为了搜索两个串之一,可以使用|,如下所示:
```sql
select * from products where prod_name REGEXP '1000|2000'
```
  该SQL匹配的就是字段中有1000,2000的记录
  匹配几个字符之一:可以使用[]来进行匹配.

```sql
-- product_name : 1000XXXXX 2000XXXX ...
select * from products where prod_name REGEXP '[123]000'
-- product_name not 1000XXXX 2000XXXX
select * from products where prod_name REGEXP '[^123]000'
-- product_name 介于1000XXXX 5000XXXX
select * from products where prod_name REGEXP '[1-5]000'
```
  如果想匹配特殊字符,可以使用\\去匹配特殊字符
```sql
select * from vendors where vend_name REGEXP '\\.'
```
  测试正则表达式:
  由于REGEXP总是返回0或者1,因此可以通过测试返回的数值判断是否匹配:
```sql
select 'hello' REGEXP '[0-9]+'
```

## 计算和数据处理

* Concat函数用于拼接各个参数的值.它可以将各个列的值拼接在一起并且计算.例如下面的:
```sql
  select CONCAT(vend_id,':',vend_name,'(',vend_country,')') from vendors
```

* Mysql的测试计算:
  虽然SELECT通常用来从表中检索数据,但是也可以省略FROM语句去进行简单的访问和处理表达式.例如下面的将返回'abc'可以便捷
  测试.
```sql
  select trim('abc ')
```

* mysql中的日期和时间函数
首先是MYSQL中的日期的数据格式,不管是插入或者更新表值或者是用WHERE子句进行过滤,日期的格式都必须是yyyy-mm-dd格式.
首先看看下面的语句是否正确:
```sql
select * from orders where order_date = '2015-09-01'
```
需要注意的是order_date是datetime的数据类型,上面的SQL目的是过滤指定日期的记录,但是显然,直接这么处理的话过滤的是指定
日期的00:00:00的记录,这就不对了
这时候需要使用DATE()函数.它会对date/datetime的数据类型的数据弄出其日期进行判断.
此外还用一种函数可以直接使用MONTH和YEAR函数去计算一个指定日期的年份和月份.例如:
```sql
select YEAR(tstamp) from log_opt_cart;
```

* 聚集函数

AVG:AVG会忽略NULL值.AVG的计算顺序在WHERE之后,在HAVING之前.
COUNT:如果使用*则不忽略NULL值.如果不使用*,指定列值,则忽略NULL值
MAX/MIN:都会忽略NULL值
SUM:会忽略NULL值

可以在一次查询中使用多个上述函数.(也就是说上述函数是可以互相之间相互组合使用的)
```sql
select count(DISTINCT(prod_price)),max(prod_id) from products
```

* 数据分组

分组需要注意的几点:
如果在SELECT指定表达式,则必须在GROUP BY指定相同的表达式,GROUP BY字句中的每个列都必须是字段或者检索表达式(但是
不能是分组函数).
如果分组具有NULL值,则作为一组返回,如果分组有多个列为NULL值,则他们被视为一组
除了聚集计算的列以外,SELECT使用的字段都需要在GOUP BY中给出
WHERE是在分组前过滤,HAVING是在分组后过滤

## 关联查询和联结表

* 组合查询
在两种情况下需要使用到组合查询:
1.在单个情况下需要从多个表返回结构相似的数据
2.从一个表中返回多次数据

通常情况下,组合查询和单表查询中的多个where条件是类似的.
例如下面的sql把子渠道为1的和子渠道为2的都查出来了:
```sql
select * from stats_cart where sub_stats_channel = 1
union
select * from stats_cart where sub_stats_channel = 2
```
UNION的规则:
1.UNION必须有两条或两条以上的select语句联结而成,并且语句之间用关键字UNION分割
2.UNION的每个查询必须有相同的列,表达式或者聚集函数
3.列数据类型必须相同,不一定要完全相同,但是至少要可以隐式转换的.
4.UNION将会自动去除重复的行,如果不希望去除重复的行,使用UNION ALL.
5.排序,order by 只允许对最终union生成的结果进行排序,不允许对于单个子查询语句进行排序,也就是说一条UNION ALL语句是没有办法有多个order by的

* mysql的全文本搜索

可以在创建表的时候指定需要全文本搜索的列,例如如下所示(注意FULLLTEXT关键词以及ENGINE):
```sql
create table productnotes
(
	note_id int(11) not null auto_increment,
	prod_id int(11),
	note_date datetime not null,
	note_text text NULL,
	PRIMARY key(note_id),
	FULLTEXT(note_text)
)Engine = MyISAM
```

上面的SQL指定了note_text为支持全文搜索的索引.
进行全文本搜索:
使用Match和Against进行全文本搜索.
Match指定了要全文搜索的列,而Against则指定要匹配的表达式.
```sql
select note_text from productnotes where MATCH(note_text) AGAINST('rabbit')
```
需要注意的是上面的sql语句,使用like其实也可以完成:
```js
select note_text FROM productnotes where note_text like '%rabbit%'
```
这个和全文本搜索有什么区别呢,区别在于全文本搜索采用了mysql自己判断的权重对于查询出来的结果进行了排序,权重是mysql根据词的数目,词最先出现的数字,包含词的行数计算出来,因此最先展示的通常是我们所需要的,此外全文搜索往往更加快捷

使用查询扩展:
查询扩展的意思就是说在通常情况下,我们指定一个关键字通常只会筛选出包含关键字的行,而在这时候我们需要筛选出来的是包含关键字且包含在包含关键字的行数里对我们有用的词的记录,这时候就需要使用到查询扩展.如下所示:
```sql
select note_text from productnotes where MATCH(note_text) AGAINST('anvils' with QUERY expansion)
```

mysql筛选结果如下:
![mysql筛选结果](http://zdoc.oss-cn-beijing.aliyuncs.com/ff47b1bf5b3fd4f147f0164a308839fb.png)

可以看出除了第一行,下面的行都是不包含关键字的,但是他们或多或少的包括关键字所在的行的其他单词,因此也被筛选出来了.
布尔文本搜索,mysql还支持另外一种文本搜索的形式,称之为布尔文本搜索,以布尔的形式可以提供如下的细节:
要匹配的词,要排斥的词,排列提示等.下面是使用布尔文本搜索的一些关键字:

| 布尔操作符 |       含义        |
| ---------- | ----------------- |
| +          | 该词必须存在      |
| -          | 排除,该词不能存在 |
| >          | 包含,且增加等级值 |
| <          | 包含,且减少等级值 |
| *          | 词尾的通配符      |
| ""         | 定义一个短语      |

例如:
包含heavy,不包含rope开头的词的
```sql
select note_text from productnotes where MATCH(note_text) AGAINST('heavy -rope*' IN BOOLEAN MODE)
```
包含rabit,habit至少一个的
```sql
select note_text from productnotes where MATCH(note_text) AGAINST('rabbit habbit' IN BOOLEAN MODE)
```
匹配rabit和habit,增加前者的等级,降低后者的等级
```sql
select note_text from productnotes where MATCH(note_text) AGAINST('>rabbit <habbit' IN BOOLEAN MODE)
```

全文本搜索有几个注意事项:
1.MYSQL有个内用词表,这些词在索引的时候总是被忽略
2.如果一个词出现在50%以上,则这个词会忽略,这一规则不使用与布尔模式
3.如果表的行数较少(少于3行),则全文本搜索不会返回结果.
4.忽略词中的单引号

## 操纵数据和操作表

* 插入数据

省略列:如果表的定义允许可以在插入数据的时候,省略某些列的值,但是省略的列必须满足以下某个条件:
1.该列定义为允许空值(无值或空值)
2.表定义中给出默认值,这表示如果不给出值,将会使用默认值.
如果不满足上面两个条件,则在插入的时候会报错.

降低优先级:由于大多数情况下数据检索比数据插入要更加重要,因此当大规模操作的时候,而数据插入通常比较耗时,因此有时候,可以指定较低的优先级,保证不影响用户的数据检索,可以使用INSERT LOW_PRIORITY INTO xxx
