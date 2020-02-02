# Join 语句分析

使用如下存储过程创建两个表 t1 和 t2。

```mysql
CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

create table t1 like t2;
insert into t1 (select * from t2 where id<=100)
```

可以看到，这两个表都有一个主键索引 id 和一个索引 a，字段 b 上无索引。存储过程 idata() 往表 t2 里插入了 1000 行数据，在表 t1 里插入的是 100 行数据。

## Index Nested-Loop Join

执行以下语句，以 t1 作为驱动表，t2 作为被驱动表。

```mysql
select * from t1 straight_join t2 on (t1.a=t2.a);
```

explain 结果如下：

![](https://tva1.sinaimg.cn/large/006tNbRwgy1gbi9ro1uihj312q04j3z3.jpg)

可以看到，在这条语句里，被驱动表 t2 的字段 a 上有索引，join 过程用上了这个索引，因此这个语句的执行流程是这样的：

1. 从表 t1 中读入一行数据 R;
2. 从数据行 R 中取出 a 字段到表 t2 里去查找；
3. 取出表 t2 中满足条件的行，跟 R 组成一行，作为结果集的一部分；
4. 重复执行步骤 1 到 3，直到表 t1 的末尾循环结束。

这个过程是先遍历表 t1，然后根据从表 t1 中取出的每行数据中的 a 值，去表 t2 中查找满足条件的记录。在形式上，这个过程就跟我们写程序时的嵌套查询类似，并且可以用上被驱动表的索引，所以我们称之为“Index Nested-Loop Join”，简称 NLJ。

它对应的流程图如下所示：

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbi9vdzr7nj30vq0og76y.jpg" style="zoom:50%;" />

## Block Nested-Loop Join

现在，把 SQL 语句改成这样：

```mysql
select * from t1 straight_join t2 on (t1.a=t2.b);
```

这时候，被驱动表上没有可用的索引，算法的流程是这样的：

1. 把表 t1 的数据读入线程内存 join_buffer 中，由于这个语句中写的是 `select *`，因此是把整个表 t1 放入了内存；
2. 扫描表 t2，把表 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回。

这个过程的流程图如下：

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbi9zg6yqij30vq0ogwgq.jpg" style="zoom:50%;" />

对应地，这条 SQL 语句的 explain 结果如下所示：

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbi9zv146uj31cz04kwf6.jpg" style="zoom:50%;" />

可以看到，在这个过程中，对表 t1 和 t2 都做了一次全表扫描。

join_buffer 的大小是由参数 join_buffer_size 设定的，默认值是 256K。如果表 t1 的数据太多，join_buffer 放不下的话，就采取分段放。

假设把 join_buffer_size 改成 1200，再执行相同的 join 语句，那么执行流程就变成了：

1. 扫描表 t1，顺序读取数据行放入 join_buffer 中，放完第 88 行 join_buffer 满了，继续第 2 步；
2. 扫描表 t2，把 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回；
3. 清空 join_buffer；
4. 继续扫描表 t1，顺序读取最后的 12 行数据放入 join_buffer 中，继续执行第 2 步。

执行流程图也就变成了这样：

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbia47ifmdj30vq0ogn0g.jpg" style="zoom:50%;" />

## NLJ 优化 - Batched Key Access

Mysql 5.6 版本后开始引入 Batched Key Access(BKA) 算法，这个算法是对 NLJ 算法的优化。

### Multi-Range Read 优化

在介绍 BKA 之前，需要先介绍一个知识点，即 Multi-Range Read 优化(MRR)。这个优化的主要目的是尽量使用顺序读盘。

我们知道，InnoDB 非主键索引上的查询，会有一个回表的过程。

假设我们执行如下查询语句：

```mysql
select * from t1 where a>=1 and a<=100;
```

主键索引是一棵 B+ 树，在这棵树上，每次只能根据一个主键 id 查到一行数据。因此，回表肯定是一行行搜索主键索引的。

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbibr560l6j30vq0ogwh8.jpg" style="zoom:50%;" />

如果随着 a 的值递增顺序查询的话，id 的值就变成随机的，那么就会出现随机访问，性能相对较差。虽然“按行查”这个机制不能改，但是调整查询的顺序，还是能够加速的。

因为大多数的数据都是按照主键递增顺序插入得到的，所以我们可以认为，如果按照主键的递增顺序查询的话，对磁盘的读比较接近顺序读，能够提升读性能。

这，就是 MRR 优化的设计思路。此时，语句的执行流程变成了这样：

1. 根据索引 a，定位到满足条件的记录，将 id 值放入 `read_rnd_buffer` 中 ;
2. 将 `read_rnd_buffer` 中的 id 进行递增排序；
3. 排序后的 id 数组，依次到主键 id 索引中查记录，并作为结果返回。

这里，`read_rnd_buffer` 的大小是由 `read_rnd_buffer_size` 参数控制的。如果步骤 1 中，`read_rnd_buffer` 放满了，就会先执行完步骤 2 和 3，然后清空 `read_rnd_buffer`。之后继续找索引 a 的下个记录，并继续循环。

另外需要说明的是，如果你想要稳定地使用 MRR 优化的话，需要设置 `set optimizer_switch="mrr_cost_based=off"`。（官方文档的说法，是现在的优化器策略，判断消耗的时候，会更倾向于不使用 MRR，把 mrr_cost_based 设置为 off，就是固定使用 MRR 了。）

下面两幅图就是使用了 MRR 优化后的执行流程和 explain 结果。

<img src="https://static001.geekbang.org/resource/image/d5/c7/d502fbaea7cac6f815c626b078da86c7.jpg" style="zoom:50%;" />

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbibvufw41j317z045q3f.jpg" style="zoom: 50%;" />

MRR 能够提升性能的核心在于，这条查询语句在索引 a 上做的是一个范围查询（也就是说，这是一个多值查询），可以得到足够多的主键 id。这样通过排序以后，再去主键索引查数据，才能体现出“顺序性”的优势。



### Batched Key Access

NLJ 算法执行的逻辑是：从驱动表 t1，一行行地取出 a 的值，再到被驱动表 t2 去做 join。也就是说，对于表 t2 来说，每次都是匹配一个值。这时，MRR 的优势就用不上了。

BKA 算法把表 t1 的数据取出来一部分，先放到临时内存 join_buffer 中，然后一起到表 t2 去做一个索引上的范围查询。如果 join_buffer 一次放不下所有数据，就分段执行。

NLJ 算法优化后的 BKA 算法的流程如下：

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbibhfadmqj30vq0ogtc4.jpg" style="zoom:50%;" />

如果要使用 BKA 优化算法的话，你需要在执行 SQL 语句之前，先设置

```mysql
set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';
```

其中，前两个参数的作用是要启用 MRR。这么做的原因是，BKA 算法的优化要依赖于 MRR。

