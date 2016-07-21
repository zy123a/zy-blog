---
layout: post
title: mysql执行计划
categories: db
tags: explain select mysql
---
* [select sql执行顺序](#select)
  * [细解](#sql)
* [explain](#explain)
  * [explain输出字段说明](#explain_result)
    * [select_type说明](#select_type)
    * [table列说明](#table)
    * [type列说明](#type)
* [索引的数据结构](#ds)
* [case](#case)

## select sql执行顺序 {#select}

        (8)SELECT (9)DISTINCT
        <select_list>
        (1)FROM <left_table>
        (3)　<join_type> JOIN <right_table>
        (2)　 ON <join_condition>  
        (4)WHERE <where_condition>
        (5)GROUP BY <group_by_list>
        (6)WITH {CUBE | ROLLUP}
        (7)HAVING <having_condition>
        (10)ORDER BY <order_by_list>
        (11)<LIMIT_specification>

执行顺序:

`笛卡尔积---on---join类型---where---group by---with{cube | rollup}---having---select---distinct---order by---limit`

ps:

limit works on MySQL and PostgreSQL, top works on SQL Server, rownum works on Oracle.

### 细解 {#sql}

1.  on vs where
    两者效果可能一样，但on是连接两表做笛卡尔积时的连接条件，where连接之后的筛选条件。

2.  where vs having

    1.  where 在对查询结果进行分组前(`group by 之前`)，将不符合where条件的行去掉，即在分组之前过滤数据， ***where条件中不能包含聚集函数***，使用where条件过滤出特定的行。

    2.  having 筛选满足条件的组(`group by 之后`,having是专门搭配group by干活的)，即在分组之后过滤数据，条件中经常包含聚集函数，使用having条件过滤出特定的组，也可以使用多个分组标准进行分组。

3.  join、left join、right join　　
    eg table_a,table_b

    1.  left join 以左为准，右可以为空, 记录数>=table_a总数
    2.  right join 以右为准，左可以为空，记录数>=table_b总数

4.  group by

    1.  先对某一列或多列进行分组，然后在分组内进行相应的操作，一般和count，max等聚集函数一起使用。

    2.  聚集函数与列不能同时出现在select子句中，除非这个列是group by子句的分组列,参见下错误例子

    3.  当查询语句中有group by子句时，聚集函数作用的对象是一个分组，而不是整个查询的结果,没有group by的话是整个查询结果

        select DepartmentId,Name,count(Salary) from Employee group by Name;

        Error Code: 1055. Expression #1 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'test.Employee.DepartmentId' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by

5.  order by

    对某一列进行排序,发生在select，distinct之后

6.  limit

    取出结果的几个 eg：limit 1,2从1开始的2个。  查询结果是从0 开始的,db会扫出这n＋m条记录，然后筛选。所以分页慎用limit m,n

7.  union和union all

    跨库查询，union会去重，union all 不会去重

## 执行计划 {#explain}

explain来解释和分析sql查询语句

    explain extended format=traditional select * from mtp_book_id in (select max(book_id) from book_goods_info order by id)

![explain](/images/database/explain.png)

ps:
1.  format有traditional 和json两种格式
2.  认为增加explain时mysql不会执行查询，这是错误的。实际上如果查询在from自居中包含子查询，mysql实际上是会执行子查询的。
3.  explain只是个近似的结果

### explain输出字段说明 {#explain_result}

|字段|对应json格式的key|含义|备注|
|-|-|-|-|-|
|id|select_id|查询语句的id|mysql按照id从小到大的顺序进行解释，实际上执行是按照id从大到小的顺序|
|select_type|none|select的类型|select查询分为简单和复杂类型，复杂类型可分为三大类：简单子查询、派生表（在from子句中的子查询）、union查询，详见[select_type说明](#select_type)|
|table|table_name|显示对应行正砸访问哪张表|可以是一张表，一个子查询、一个union结果。详见[table说明](#table)|
|partitions|partitions|匹配到的分区||
|type|access_type|**mysql手册上说这一列是关联类型，但我们认为更准确的说法是访问类型**，参见*高性能mysql 3th p695*，我也觉得这样说比较合适|详见[type说明](#type)|
|possible_keys|possiable_keys|可能会选择的索引|这是基于查询访问的列和使用的比较操作符来判断的，这个列表是在优化过程中早起创建的，因此罗列出来的索引可能对于后续优化过程是没用的|
|key|key|真实选择的索引|显示的是mysql实际使用的索引|
|key_len|key_length|选择的索引的长度|mysql决定使用的索引的长度|
|ref|ref|与索引进行比较的列数||
|rows|rows|被检查的记录数的估量|rows是核心指标，绝大部分rows小的语句执行一定很快（有例外),所以优化语句基本上都是在优化rows。|
|filtered|filtered|被查询条件过滤的记录数的占比|5.7.3之前extended才会显示,总是100%！？参见[总是100%](http://blog.chinaunix.net/uid-20726500-id-5573764.html)|
|Extra|none|额外信息|using index 使用覆盖索引；using where 使用where过滤|

#### select_type说明 {#select_type}

|类型|对应json格式的key|含义|
|-|-|-|
|simple|none|`简单的查询（没有使用union或者子查询）`|
|primary|none|`最外层的select（有子查询或者union时）`|
|union|none|多个union时，第二个和更后的select|
|union result|union_result|`从union的匿名临时表检索结果的select被标记为union result`|
|subquery|none|`包含在select列表中的子查询中的select（不再from子句中）`|
|derived|none|`包含在from子句中的select，mysql会递归执行并将结果放到一个临时表中，服务器内部称其为‘派生表’，因为该临时表是从子查询中派生来的`|
|materialized|materialized_from_subquery||
|dependent subquery|dependent(true)|类subquery，依赖于外层查询|
|dependent union|dependent(true)|类union，依赖于外层的查询|
|uncacheable subquery|cacheable (false)||
|uncacheable union|cacheable (false)||

ps:
1.  denpendent: select依赖于外层查询中的数据（你就看你的sql单拿出来，是否可以执行）。
2.  uncacheable: select中的某些特性阻止结果被缓存在一个Item_cache中（Item_cache未被文档记载，它与查询缓存不是一回事，尽管它可以被一些相同类型的构件否定，例如RAND()函数）
3.  针对select，不出意外的话，应该是你的sql中有多少个select，explain你的sql，就会有多少行，每个select都对应有自己的select_type。

#### table列说明 {#table}

M，N都是explain结果中的id字段值

|类型|含义|
|-|-|
|`<unionM,N>`|当select type为union result，table列为参与union的id列表，总是`向后引用`|
|`<derivedN>` |第n行的派生表的select|
|`<subqueryN>`| The row refers to the result of a materialized subquery for the row with an id value of N|

#### type {#type}

|类型|含义|备注|case|
|-|-|-|-|
|system|表只有一行（系统表），这是const的特殊情形|todo??|
|const|表至多匹配一行，用于和主键或者unique index比较时。||`SELECT * FROM tbl_name WHERE primary_key=1;`|
|eq_ref|索引访问（索引查找）|表至多匹配一条记录，用于primary key或者unique not null索引|`SELECT * FROM ref_table,other_table WHERE ref_table.key_column=other_table.column;`|
|ref|索引访问|匹配的不是一个记录，而是一些记录，访问只有当使用非唯一性索引或者唯一索引的非唯一性前缀时才会发生|`SELECT * FROM ref_table WHERE key_column=expr;`|
|ref_or_null|类似ref，多了个null|| `SELECT * FROM ref_table WHERE key_column=expr OR key_column IS NULL;`|
|range|范围扫描|有限制的索引扫描，使用操作符`=, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, or IN()`、与常量进行比较 ，使用索引列表去查找一系列值，例如in（）和or列表们也会显示范围查询，但访问性能有重要的差异，开销跟索引类型相当|`SELECT * FROM tbl_name WHERE key_column = 10;`  `SELECT * FROM tbl_name WHERE key_column BETWEEN 10 and 20;`|
|index|全索引扫描|类全表扫描，只是msyql扫描表时按照索引次序进行，而不是行。优点是避免了排序，缺点是要承担按索引读取整个表的开销。当查询结果可以被索引包含时，只扫描索引的数据，而不是按索引次序的每一行，这时Extra列会有`using index`标示（`覆盖索引`）||
|all|全表扫描|通常意味着mysql必须扫描整张表,也有例外，例如在查询中使用了limit，或者在Extra列显示`using distinct/not exists`||
|fulltext|处理fulltext索引时||
|index_merge|访问使用了索引合并优化|todo|
|unique_subquery|类似eq_ref，只不过不是＝，而是in||`value IN (SELECT primary_key FROM single_table WHERE some_expr)`|
|index_subquery|类似unique_subquery,只不过作用于非unique索引||`value IN (SELECT key_column FROM single_table WHERE some_expr)`|

注：

1. `覆盖索引`(Covering Index)：只用到某个索引，且该索引包含查询需要的数据列，也就没有回表操作。
2. 覆盖索引必须要存储索引列的值，而哈希索引、空间索引、和全文索引都不能存储列的值，所以MySQL只能使用B-Tree索引做覆盖索引

## 索引的数据结构 {#ds}

myisam

![myisam_pri](/images/database/myisam_pri.png)

1.  使用[b＋树](https://en.wikipedia.org/wiki/B%2B_tree)
2.  **索引文件和数据文件是分离的。** 索引文件仅保存数据记录的地址，索引的叶子节点的data域存放的是数据记录的地址
(上图`标红1处`key为15的叶子节点，只保存数据域的指针0x07,标红2处,是存放数据记录的位置)
3.  **主索引和二级索引结构上没有任何差别，只是主索引要求key必须是唯一的**

innodb

![innodb_pri](/images/database/innodb_pri.png)
![innodb_second](/images/database/innodb_second.png)

1.  同样是b+树
2.  数据文件同样是索引文件，数据文件本身就是按B+Tree组织的一个索引结构，叶子节点data域保存完整的数据记录(primary key图中`标红1处`,key为15的叶子节点，数据域就是数据记录)，这个索引的key是数据表的主键，因此innodb的表数据文件本身就是主索引。
3.  innodb主索引，叶节点包含了完整的数据记录，这种索引叫做`聚集索引`
4.  innodb的辅助索引data域存储相应记录主键的值而不是数据记录的地址（`标红2处`,以col3建了个索引，key为col3，value是其对应的主键）。
5.  联合索引和最左前缀匹配
6.  主键不一定是一个字段哟！

ps: 因为innodb的数据文件本身要按主键聚集，所以innodb要求表必须有主键（myisam可以没有），如果没有显式指定，则mysql系统会自动选择一个可以唯一标识数据记录的列作为主键，如果不存在这种列，则mysql自动为innodb表生成一个隐含字段作为主键，这个字段长度为6个字节，类型为长整形。

## case
case1:

    mysql> explain select * from mtp_book_info order by book_id desc limit 10000,10;
    +----+-------------+---------------+-------+---------------+---------+---------+------+-------+-------+
    | id | select_type | table         | type  | possible_keys | key     | key_len | ref  | rows  | Extra |
    +----+-------------+---------------+-------+---------------+---------+---------+------+-------+-------+
    |  1 | SIMPLE      | mtp_book_info | index | NULL          | PRIMARY | 4       | NULL | 10010 |       |
    +----+-------------+---------------+-------+---------------+---------+---------+------+-------+-------+
    1 row in set (0.05 sec)

1.  没有子查询和union，所以select_type为simple
2.  type为index,表明为索引扫描，扫描索引数，然后回表拿数据
3.  book_id 为primary key
4.  book_id类型为int，字节长度为4
5.  从10000到10010，但mysql需要读取前10010，然后截取10000-10010? 存疑！

case2:

    mysql> explain select  * from mtp_book_info where book_id >= (select book_id from mtp_book_info
    order by book_id  desc limit 10000, 1) order by book_id desc  limit 10;  
    +----+-------------+---------------+-------+---------------+---------+---------+------+-------+-------------+
    | id | select_type | table         | type  | possible_keys | key     | key_len | ref  | rows  | Extra       |
    +----+-------------+---------------+-------+---------------+---------+---------+------+-------+-------------+
    |  1 | PRIMARY     | mtp_book_info | range | PRIMARY       | PRIMARY | 4       | NULL | 19764 | Using where |
    |  2 | SUBQUERY    | mtp_book_info | index | NULL          | PRIMARY | 4       | NULL | 10001 | Using index |
    +----+-------------+---------------+-------+---------------+---------+---------+------+-------+-------------+
    2 rows in set (0.04 sec)

1.  解释是按照id从小到大开始解释的，执行是从大到小。
2.  有子查询，且不在from，join子句中，所以2的select_type是subquery，而不是derived
3.  id为2: type为index，且Extra为Using index表明使用了覆盖索引优化,索引树中包含所有select的数据，所以就不需要回表拿数据了
4.  id为1: 使用了book_id索引，且使用了>=,所以为range，
5.  id为1: 使用了where条件，所以Extra中有Using where
6.  id为1: rows 19764 ？ todo

case3:

    mysql> explain select * from mtp_book_info as book1 join (select book_id from mtp_book_info
    order by book_id desc limit 10000, 10) as book2 where  book1.book_id = book2.book_id ;
    +----+-------------+---------------+--------+---------------+---------+---------+---------------+-------+-------------+
    | id | select_type | table         | type   | possible_keys | key     | key_len | ref           | rows  | Extra       |
    +----+-------------+---------------+--------+---------------+---------+---------+---------------+-------+-------------+
    |  1 | PRIMARY     | <derived2>    | ALL    | NULL          | NULL    | NULL    | NULL          |    10 |             |
    |  1 | PRIMARY     | book1         | eq_ref | PRIMARY       | PRIMARY | 4       | book2.book_id |     1 |             |
    |  2 | DERIVED     | mtp_book_info | index  | NULL          | PRIMARY | 4       | NULL          | 10010 | Using index |
    +----+-------------+---------------+--------+---------------+---------+---------+---------------+-------+-------------+
    3 rows in set (0.03 sec)

1. id为2: join子句，所以select_type为derived
2. id为2: 覆盖索引，case2
3. id为1: 都是最外层的select，所以select_type为primary
4. id为1的第一个:因为对于派生表来说，book_id只是一个普通字段，所以其type为all，需要进行全表扫描
5. id为1的第二个:book_id为mtp_book_info的主键，又使用的是＝，对于每个主键，至多又一条记录，所以type为all

case4:

    mysql> explain select order_display_id from mtp_book_info  union all select order_display_id from book_goods_info;
    +----+--------------+-----------------+-------+---------------+----------------------+---------+------+-------+-------------+
    | id | select_type  | table           | type  | possible_keys | key                  | key_len | ref  | rows  | Extra       |
    +----+--------------+-----------------+-------+---------------+----------------------+---------+------+-------+-------------+
    |  1 | PRIMARY      | mtp_book_info   | index | NULL          | idx_order_display_id | 8       | NULL | 64174 | Using index |
    |  2 | UNION        | book_goods_info | index | NULL          | idx_order_display_id | 8       | NULL | 64147 | Using index |
    | NULL | UNION RESULT | <union1,2>      | ALL   | NULL          | NULL                 | NULL    | NULL |  NULL |             |
    +----+--------------+-----------------+-------+---------------+----------------------+---------+------+-------+-------------+
    3 rows in set (0.04 sec)

1.  id为2: 多个union时，第二个和更后的select,select_type为union
2.  id为null: union result 参与union的两张表为id为1和2的select结果

## 参考 {#ref}

[mysql explain详解]<http://www.cnitblog.com/aliyiyi08/archive/2016/04/21/48878.html>

[官方文档]<https://dev.mysql.com/doc/>

[MySQL索引背后的数据结构及算法原理]<http://blog.codinglabs.org/articles/theory-of-mysql-index.html>

[《高性能MySQL》读书笔记－－索引]<http://blog.csdn.net/xifeijian/article/details/20312557>

[MySQL索引原理及慢查询优化]<http://tech.meituan.com/mysql-index.html>

[Innodb页面存储结构-1]<http://tencentdba.com/blog/innodb%E9%A1%B5%E9%9D%A2%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84-1/>

[Innodb页面存储结构-2]<http://tencentdba.com/blog/innodb%E9%A1%B5%E9%9D%A2%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84-2/>