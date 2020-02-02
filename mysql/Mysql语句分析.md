# Mysql 语句分析

## Join 语句分析

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

### Index Nested-Loop Join

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

### Block Nested-Loop Join

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

### NLJ 优化

Mysql 5.6 版本后开始引入 Batched Key Access(BKA) 算法，这个算法是对 NLJ 算法的优化。

#### Multi-Range Read 优化

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



#### Batched Key Access

NLJ 算法执行的逻辑是：从驱动表 t1，一行行地取出 a 的值，再到被驱动表 t2 去做 join。也就是说，对于表 t2 来说，每次都是匹配一个值。这时，MRR 的优势就用不上了。

BKA 算法把表 t1 的数据取出来一部分，先放到临时内存 join_buffer 中，然后一起到表 t2 去做一个索引上的范围查询。如果 join_buffer 一次放不下所有数据，就分段执行。

NLJ 算法优化后的 BKA 算法的流程如下：

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbibhfadmqj30vq0ogtc4.jpg" style="zoom:50%;" />

如果要使用 BKA 优化算法的话，你需要在执行 SQL 语句之前，先设置

```mysql
set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';
```

其中，前两个参数的作用是要启用 MRR。这么做的原因是，BKA 算法的优化要依赖于 MRR。



## Group by 分析

为了便于量化分析，用下面的表 t1 来举例。

```mysql
create table t1(id int primary key, a int, b int, index(a));
delimiter ;;
create procedure idata()
begin
  declare i int;

  set i=1;
  while(i<=1000)do
    insert into t1 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();
```

执行如下语句：

```mysql
select id%10 as m, count(*) as c from t1 group by m;
```

这个语句的逻辑是把表 t1 里的数据，按照 id%10 进行分组统计，并按照 m 的结果排序后输出。它的 explain 结果如下：

![](https://tva1.sinaimg.cn/large/006tNbRwgy1gbicjmft8xj31ba03zq3h.jpg)

在 Extra 字段里面，我们可以看到三个信息：

- Using index，表示这个语句使用了覆盖索引，选择了索引 a，不需要回表；
- Using temporary，表示使用了临时表；
- Using filesort，表示需要排序。

这个语句的执行流程是这样的：

1. 创建内存临时表，表里有两个字段 m 和 c，主键是 m；
2. 扫描表 t1 的索引 a，依次取出叶子节点上的 id 值，计算 id%10 的结果，记为 x；
   - 如果临时表中没有主键为 x 的行，就插入一个记录 (x,1);
   - 如果表中有主键为 x 的行，就将 x 这一行的 c 值加 1；
3. 遍历完成后，再根据字段 m 做排序，得到结果集返回给客户端。

这个流程的执行图如下：

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbicpet4xlj30vq0ogdhz.jpg" style="zoom:50%;" />

图中最后一步，对内存临时表的排序流程如下图虚线中所示：

<img src="https://static001.geekbang.org/resource/image/b5/68/b5168d201f5a89de3b424ede2ebf3d68.jpg" style="zoom:50%;" />

接下来，我们再看一下这条语句的执行结果：

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbicsy1gbqj30is09o0t4.jpg" style="zoom:50%;" />

如果你的需求并不需要对结果进行排序，那你可以在 SQL 语句末尾增加 order by null，也就是改成：

```mysql
select id%10 as m, count(*) as c from t1 group by m order by null;
```

这样就跳过了最后排序的阶段，直接从临时表中取数据返回。

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbicu02rlij30mv0agaal.jpg" style="zoom:50%;" />

内存临时表的大小由参数 `tmp_table_size` 控制，默认是 16M。如果内存临时表不够存放所有的数据，就会把内存临时表转成磁盘临时表。

### Group by 优化

#### 索引

group by 的语义逻辑，是统计不同的值出现的个数。但是，由于每一行的 id%100 的结果是无序的，所以我们就需要有一个临时表，来记录并统计结果。

假设，现在有一个类似下图的这么一个数据结构，我们来看看 group by 可以怎么做。

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbid0c5uisj30vq0og758.jpg" style="zoom:50%;" />

可以看到，如果可以确保输入的数据是有序的，那么计算 group by 的时候，就只需要从左到右，顺序扫描，依次累加。也就是下面这个过程：

- 当碰到第一个 1 的时候，已经知道累积了 X 个 0，结果集里的第一行就是 (0,X);
- 当碰到第一个 2 的时候，已经知道累积了 Y 个 1，结果集里的第二行就是 (1,Y);

按照这个逻辑执行的话，扫描到整个输入的数据结束，就可以拿到 group by 的结果，不需要临时表，也不需要再额外排序。

InnoDB 的索引，就可以满足这个输入有序的条件。

在 MySQL 5.7 版本支持了 generated column 机制，用来实现列数据的关联更新。你可以用下面的方法创建一个列 z，然后在 z 列上创建一个索引（如果是 MySQL 5.6 及之前的版本，你也可以创建普通列和索引，来解决这个问题）。

```mysql
alter table t1 add column z int generated always as(id % 100), add index(z);
```

这样，索引 z 上的数据就是类似上图这样有序的了。上面的 group by 语句就可以改成：

```mysql
select z, count(*) as c from t1 group by z;
```

优化后的 group by 语句的 explain 结果，如下图所示：

![](https://tva1.sinaimg.cn/large/006tNbRwgy1gbid4s5vvjj311h03zgm0.jpg)

从 Extra 字段可以看到，这个语句的执行不再需要临时表，也不需要排序了。

#### 直接排序

所以，如果可以通过加索引来完成 group by 逻辑就再好不过了。但是，如果碰上不适合创建索引的场景，我们还是要老老实实做排序的。

如果我们明明知道，一个 group by 语句中需要放到临时表上的数据量特别大，却还是要按照“先放到内存临时表，插入一部分数据后，发现内存临时表不够用了再转成磁盘临时表”，看上去就有点儿傻。所以此时我们可以让 MySQL 直接走磁盘临时表。

在 group by 语句中加入 `SQL_BIG_RESULT` 这个提示，就可以告诉优化器：这个语句涉及的数据量很大，请直接用磁盘临时表。

MySQL 的优化器一看，磁盘临时表是 B+ 树存储，存储效率不如数组来得高。所以，既然你告诉我数据量很大，那从磁盘空间考虑，还是直接用数组来存吧。

```mysql
select SQL_BIG_RESULT id%100 as m, count(*) as c from t1 group by m;
```

这个语句的执行流程是这样的：

1. 初始化 sort_buffer，确定放入一个整型字段，记为 m；
2. 扫描表 t1 的索引 a，依次取出里面的 id 值, 将 id%100 的值存入 sort_buffer 中；
3. 扫描完成后，对 sort_buffer 的字段 m 做排序（如果 sort_buffer 内存不够用，就会利用磁盘临时文件辅助排序）；
4. 排序完成后，就得到了一个有序数组。

根据有序数组，得到数组里面的不同值，以及每个值的出现次数。此时就和上面的索引优化一样了。

下面两张图分别是执行流程图和执行 explain 命令得到的结果。

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbidau0ppij30vq0og0v6.jpg"  />

![](https://tva1.sinaimg.cn/large/006tNbRwgy1gbidb51vsrj312p03n0t7.jpg)

从 Extra 字段可以看到，这个语句的执行没有再使用临时表，而是直接用了排序算法。

## Union 分析

为了便于量化分析，用下面的表 t1 来举例。

```mysql
create table t1(id int primary key, a int, b int, index(a));
delimiter ;;
create procedure idata()
begin
  declare i int;

  set i=1;
  while(i<=1000)do
    insert into t1 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();
```

执行如下语句：

```mysql
(select 1000 as f) union (select id from t1 order by id desc limit 2);
```

这条语句用到了 union，它的语义是，取这两个子查询结果的并集。并集的意思就是这两个集合加起来，重复的行只保留一行。

下图是这个语句的 explain 结果。

![](https://tva1.sinaimg.cn/large/006tNbRwgy1gbiderol4yj315x057wfb.jpg)

可以看到：

- 第二行的 key=PRIMARY，说明第二个子句用到了索引 id。
- 第三行的 Extra 字段，表示在对子查询的结果集做 union 的时候，使用了临时表 (Using temporary)。

这个语句的执行流程是这样的：

1. 创建一个内存临时表，这个临时表只有一个整型字段 f，并且 f 是主键字段。
2. 执行第一个子查询，得到 1000 这个值，并存入临时表中。
3. 执行第二个子查询：
   - 拿到第一行 id=1000，试图插入临时表中。但由于 1000 这个值已经存在于临时表了，违反了唯一性约束，所以插入失败，然后继续执行；
   - 取到第二行 id=999，插入临时表成功。
4. 从临时表中按行取出数据，返回结果，并删除临时表，结果中包含两行数据分别是 1000 和 999。

这个过程的流程图如下所示：

<img src="https://tva1.sinaimg.cn/large/006tNbRwgy1gbidi0ca1cj30vq0ogjsu.jpg" style="zoom:50%;" />

可以看到，这里的内存临时表起到了暂存数据的作用，而且计算过程还用上了临时表主键 id 的唯一性约束，实现了 union 的语义。

顺便提一下，如果把上面这个语句中的 union 改成 union all 的话，就没有了“去重”的语义。这样执行的时候，就依次执行子查询，得到的结果直接作为结果集的一部分，发给客户端。因此也就不需要临时表了。

![](https://tva1.sinaimg.cn/large/006tNbRwgy1gbidisy71aj313504ot9b.jpg)

可以看到，第二行的 Extra 字段显示的是 Using index，表示只使用了覆盖索引，没有用临时表了。