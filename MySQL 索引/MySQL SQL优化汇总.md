# MySQL SQL优化汇总

##### NL & HASH JOIN

MySQL8.0有两种连接方式，选择NL还是hash join，要看两表关联后返回少量数据还是大量数据，一般情况下，少量数据 NL 优于 hash join，大量数据，hash join优于 NL。

如果只能选择NL连接（低于MySQL8.0的版本），那么在NL 情况下，是小表驱动大表快还是大表驱动小表快，看大表关联使用的索引是否形成索引覆盖，及关联后返回的数据量。

大表关联使用二级索引，关联后返回大量数据，又需要回表，这种情况下，一般选择大表驱动小表效率高些；关联后返回少量数据，一般选择小表驱动大表效率高些。

大表关联使用索引覆盖，要看大表过滤后的数据量占全表的百分比，不同的数据量可能就需要选择不同的方式。

不要试图去记住这些结论，深入了解表的连接方式与扫描方式，理解SQL的执行过程，一切都会变得顺理成章，我们的人脑会对SQL选择哪种执行计划执行效率高有一个清晰的判断，如果优化器做出错误的决策，可以尝试使用各种优化方式干涉优化器的决策。

(https://mp.weixin.qq.com/s/k9OxACEbAF91TxwU-_Zkfg)例子中小表驱动大表与大表驱动小表的执行耗时是差不多的，哪种方式效率高主要看大表过滤后的数据量占全表的百分比，不同的数据量可能就需要选择不同的方式。

##### HINT &

1. `STRAIGHT_JOIN`: 这个提示告诉查询优化器按照在查询中指定的顺序进行表的联接操作，而不考虑优化器自身的联接顺序决策。可以使用该提示来强制使用指定的表联接顺序。

   示例：

   ```sql
   sqlCopy code
   SELECT STRAIGHT_JOIN t1.column1, t2.column2 FROM table1 AS t1, table2 AS t2 WHERE t1.id = t2.id;
   ```

2. `USE INDEX` / `IGNORE INDEX`: 这些提示用于指定在查询中使用或忽略的索引。可以使用这些提示来强制查询使用或忽略特定的索引。

   示例：

   ```sql
   sqlCopy codeSELECT * FROM table1 USE INDEX (index_name) WHERE column1 = 'value';
   SELECT * FROM table1 IGNORE INDEX (index_name) WHERE column1 = 'value';
   ```

3. `FORCE INDEX`: 这个提示用于强制查询使用特定的索引。可以使用该提示来确保查询使用指定的索引。

   示例：

   ```sql
   sqlCopy code
   SELECT * FROM table1 FORCE INDEX (index_name) WHERE column1 = 'value';
   ```

4. `IGNORE INDEX`是一种查询提示（query hint），用于指示查询优化器忽略特定的索引。当使用`IGNORE INDEX`提示时，查询优化器将不会考虑指定的索引来执行查询，而是选择其他可用的索引或执行表扫描。

   `IGNORE INDEX`提示可以在查询语句的`FROM`子句中使用。以下是一个示例：

   ```sql
   sqlCopy code
   SELECT * FROM table_name IGNORE INDEX (index_name) WHERE condition;
   ```

请注意，使用提示需要小心，因为它们可能会限制优化器的选择和灵活性。在使用提示之前，建议先进行性能测试和分析，确保提示对查询性能产生了实际的改进。同时，应该定期评估查询和索引的性能，以便根据实际情况进行调整和优化。

另外，提示的语法和支持程度可能因MySQL版本和配置而有所不同。建议查阅MySQL官方文档或特定版本的文档，以获取更准确和详细的提示使用说明。

##### SQL优化过程：

总结如下：

1.本条SQL的最终执行计划是大表驱动小表，这也算是给上篇文章《NL连接一定是小表驱动大表效率高吗》提供了一个案例。

2.优化措施可能有很多不同的选择，要根据实际情况选择最优的，不要草率做出决定。

3.精益求精是优化的极致，但是有时候也是需要做出折中选择的，达到业务运行的要求是目的，这点以后遇到案例再说。

其中t_1表25条记录，t_2表100条记录，t_3表500万条数据。实际业务表数据量分别是（30，150，2700万）。（createtable末尾）

```sql
insert into t_4
SELECT
 c.w_name,
 a.s_i_id,
 a.s_quantity,
 a.s_ytd,
 a.s_order_cnt,
 a.s_remote_cnt,
 a.s_data,
 a.t_2_id,
 b.i_name,
 b.i_price
FROM
 t_3 a,
 t_2 b,
 t_1 c
WHERE
 a.t_2_id = b.i_id
and a.t_1_id = c.w_id
and a.s_ytd = 0;
```

就是三张表进行关联，但是关联字段上都没有索引，都进行了全表扫描。那么解决措施就是加索引，<font color='orange'>但是索引怎么加就需要做出选择了</font>。

给大表加上索引就不用全表扫描了，首先大表加索引，会锁表很长时间。

test：

```sql
mysql> alter table t_3 add key(t_1_id,t_2_id);
Query OK, 0 rows affected (28.35 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

索引加好之后，执行计划如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/zYias49R9JlTTia5yOfuG9IEHemIibZyzxNglUXcDic8CibrbHAnzIqlzicOLMsNMXH6m6587TiabKw21aS456wd2aUEw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

看出优化器并没有选择走索引，依然是使用BNL优化策略，进行全表扫描，为什么不走索引？应该是优化器认为索引扫描的成本高于全表扫描的成本，因为这条语句最终结果要返回大表的90%以上的数据，走索引后回表代价是很高的。

这点我们是不认同优化器的，怎么着2500次全表扫描也比每次通过索引范围扫描的代价要高，使用force index来干涉优化器决策，让它使用索引。

![图片](https://mmbiz.qpic.cn/mmbiz_png/zYias49R9JlTTia5yOfuG9IEHemIibZyzxNqQyAXJmW9xIs9zZKzLcvG3WNEnRpHy3oCKbOBzYKj91iadtozZ2CxtQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

```sql
mysql> insert into t_4
    -> SELECT
    ->   c.w_name,
    ->   a.s_i_id,
    ->   a.s_quantity,
    ->   a.s_ytd,
    ->   a.s_order_cnt,
    ->   a.s_remote_cnt,
    ->   a.s_data,
    ->   a.t_2_id,
    ->   b.i_name,
    ->   b.i_price
    -> FROM
    ->  t_3 a force index(t_1_id),
    ->  t_2 b,
    ->  t_1 c
    -> WHERE
    ->  a.t_2_id = b.i_id
    -> and a.t_1_id = c.w_id
    -> and a.s_ytd = 0;
Query OK, 4800000 rows affected (4 min 43.57 sec)
Records: 4800000  Duplicates: 0  Warnings: 0
```

500万数据需要`4 min 43.57 sec`

但不是效率最高的办法呢，因为最终结果集会返回大表的90%以上的数据，所以需要对大量的索引数据回表，因为回表是会产生随机IO的，这个回表代价确实比较高，优化器默认也没有选择这种执行计划。给小表的关联字段上加索引:

test:

```sql
mysql> alter table t_2 add key(i_id);
Query OK, 0 rows affected (0.05 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> alter table t_1 add key(w_id);
Query OK, 0 rows affected (0.03 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

去掉大表的force index，不干涉优化器，让优化器自己做决策。执行计划如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/zYias49R9JlTTia5yOfuG9IEHemIibZyzxNOD9mDibcqnHKzqicR8rfctDDh8N2MdMvzAZianHr1JW5H6xKMovW47H8w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

执行计划显示，优化器选择了对大表全表扫描，大表做驱动表，驱动两个小表;

实际效果:

```sql
mysql> insert into t_4
    -> SELECT
    ->   c.w_name,
    ->   a.s_i_id,
    ->   a.s_quantity,
    ->   a.s_ytd,
    ->   a.s_order_cnt,
    ->   a.s_remote_cnt,
    ->   a.s_data,
    ->   a.t_2_id,
    ->   b.i_name,
    ->   b.i_price
    -> FROM
    ->  t_3 a,
    ->  t_2 b,
    ->  t_1 c
    -> WHERE
    ->  a.t_2_id = b.i_id
    -> and a.t_1_id = c.w_id
    -> and a.s_ytd = 0;
Query OK, 4800000 rows affected (1 min 59.06 sec)
Records: 4800000  Duplicates: 0  Warnings: 0
```

耗时`1min 59.06sec` ，采用大表驱动小表这种方式效率提高了，优化器的选择是对的。

选择这种方式的好处：

**1.SQL的执行效率高一倍**

**2.节省空间，因为大表的索引会占用很大的磁盘空间。**

**3.响应及时，避免了必须等到变更窗口才能加索引的麻烦。**

**4.不用修改SQL语句**

优化结束，但是如果想要精益求精，追求极致的话，小表上的索引可以建成覆盖索引，防止小表回表取数据。

```sql
mysql> alter table t_1 drop key w_id;
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> alter table t_2 drop key i_id;
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> alter table t_2 add key(i_id,i_name,i_price);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> alter table t_1 add key(w_id,w_name);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

执行效果如下：

```sql
mysql> insert into t_4
    -> SELECT
    ->   c.w_name,
    ->   a.s_i_id,
    ->   a.s_quantity,
    ->   a.s_ytd,
    ->   a.s_order_cnt,
    ->   a.s_remote_cnt,
    ->   a.s_data,
    ->   a.t_2_id,
    ->   b.i_name,
    ->   b.i_price
    -> FROM
    ->  t_3 a,
    ->  t_2 b,
    ->  t_1 c
    -> WHERE
    ->  a.t_2_id = b.i_id
    -> and a.t_1_id = c.w_id
    -> and a.s_ytd = 0;
Query OK, 4800000 rows affected (1 min 38.99 sec)
Records: 4800000  Duplicates: 0  Warnings: 0
```

可以看出，小表上的索引建成覆盖索引，耗时**又缩短了20秒**，执行效率更高了。

```sql
Create Table: CREATE TABLE `t_1` (
 `w_id` int(11) DEFAULT NULL,
 `w_name` varchar(10) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

```sql
Create Table: CREATE TABLE `t_2` (
 `i_id` int(11) NOT NULL,
 `i_name` varchar(24) DEFAULT NULL,
 `i_price` decimal(5,2) DEFAULT NULL,
 `i_data` varchar(50) DEFAULT NULL,
 `i_im_id` int(11) NOT NULL,
 PRIMARY KEY (`i_im_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

```sql
Create Table: CREATE TABLE `t_3` (
 `s_w_id` int(11) NOT NULL,
 `s_i_id` int(11) NOT NULL,
 `s_quantity` int(11) DEFAULT NULL,
 `s_ytd` int(11) DEFAULT NULL,
 `s_order_cnt` int(11) DEFAULT NULL,
 `s_remote_cnt` int(11) DEFAULT NULL,
 `s_data` varchar(50) DEFAULT NULL,
 `s_dist_01` char(24) DEFAULT NULL,
 `s_dist_02` char(24) DEFAULT NULL,
 `s_dist_03` char(24) DEFAULT NULL,
 `s_dist_04` char(24) DEFAULT NULL,
 `s_dist_05` char(24) DEFAULT NULL,
 `s_dist_06` char(24) DEFAULT NULL,
 `s_dist_07` char(24) DEFAULT NULL,
 `s_dist_08` char(24) DEFAULT NULL,
 `s_dist_09` char(24) DEFAULT NULL,
 `s_dist_10` char(24) DEFAULT NULL,
 `t_2_id` int(11) DEFAULT NULL,
 `t_1_id` int(11) DEFAULT NULL,
 PRIMARY KEY (`s_w_id`,`s_i_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

```sql
Create Table: CREATE TABLE `t_4` (
 `w_name` varchar(10) DEFAULT NULL,
 `s_i_id` int(11) NOT NULL,
 `s_quantity` int(11) DEFAULT NULL,
 `s_ytd` int(11) DEFAULT NULL,
 `s_order_cnt` int(11) DEFAULT NULL,
 `s_remote_cnt` int(11) DEFAULT NULL,
 `s_data` varchar(50) DEFAULT NULL,
 `t_2_id` int(11) DEFAULT NULL,
 `i_name` varchar(24) DEFAULT NULL,
 `i_price` decimal(5,2) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

