在互联网大并发应用大行其道的今天，应用的开发总是离不开锁，在分布式应用中，最常见的莫过于基于数据库的行级锁了，由于互联网公司中比较主流的数据库还是mysql，所以这一话题绕不开的就是mysql了，但是由于mysql中innoDb引擎特殊的机制，经常一不小心就会发生死锁，本文咱们就来聊一聊基于mysql innodb 实现的行级锁，以及为什么会产生死锁，和如何避免死锁

首先，使用mysql实现行级锁的两大前提就是，innodb引擎并且开启事务。由于MySQL/InnoDB的加锁分析，一直是一个比较困难的话题。本次咱们暂时只讨论在日常应用中 select .... from table where ..... for update 语句并且在 Repeatable Read 事务隔离级别下

在此之前先明确几个概念：

1.Index Key：

用于确定SQL查询在索引中的连续范围(起始范围+结束范围)的查询条件，被称之为Index Key。

2.Index Filter：

在完成Index Key的提取之后，我们根据where条件固定了索引的查询范围，但是此范围中的项，并不都是满足查询条件的项。根据其他条件排除此范围中不满足的项

3.Table Filter：

所有不属于索引列的查询条件，均归为Table Filter之中。

 

一个sql的筛选过程就是先Index Key 到 Index Filter 再到 Table Filter。

其实死锁最大的难点可能就是很多人不知道一条for update到底是怎么加锁的，而在innodb引擎中行级锁分为以下三种锁

1.Record Lock 

单个行记录上的锁

2.Gap Lock

间隙锁，锁定一个范围，不包括记录本身

3.Next-Key Lock

锁定一个范围和记录本身

 

我们分以下举例说明：

 select * from table where id = 1 for update;

 

 id 是主键的时候，本条sql在Index Key阶段可以确定唯一一条数据，所以会在聚簇索引上加Record Lock

 id 是普通索引的时候，本条sql在Index Key阶段筛选出的数据不具有唯一性，所以Innodb为了防止幻度，会加Gap Lock+Next-Key Lock(Repeatable Read 事务隔离级别下,在Table Filter阶段对相应的聚簇索引上加Record Lock

 id 不是索引的时候，本条sql在Table Filter阶段进行全表扫描，会在所有的聚簇索引上加锁，相当于全表锁，这是由于MySQL的实现决定的。如果一个条件无法通过索引快速过滤，那么innodb引擎层面就会将所有记录对应的聚簇索引加锁后返回，然后由MySQL Server层进行过滤，在高版本的mysql中会将不符合的记录再解锁

 

select * from table where id = 1 and time = '2019-06-18' for update;

 

 id 是主键，time不是索引的时候，本条sql在Index Key阶段可以确定唯一一条数据，Index Filter,Table Filter 都只有一条数据,所以会在聚簇索引上加Record Lock.

 id 是普通索引，time不是索引的时候，本条sql在Index Key阶段筛选出的数据不具有唯一性，所以Innodb为了防止幻度，会加id和小于1的索引之间加Next-Key Lock锁，在大于id和下一个索引之间加和Gap Lock锁(Repeatable Read 事务隔离级别下),Table Filter阶段会扫描出id = 1 范围下所有的聚簇索引加锁

 id 不是索引，time不是索引的时候，本条sql在Table Filter阶段进行全表扫描，会在所有的聚簇索引上加锁，相当于全表锁

 id 和time都是普通索引的时候，会再id索引和time索引上分别加Next-Key Lock和Gap Lock,在Table Filter阶段对相应的聚簇索引上加Record Lock

 

由上两个例子得出，我们的for update 并不时都锁一条记录，也并不是只有一个锁，这两个例子基本上已经包含了for update中常见的锁，在此基础上我们可以根据MySQL的加锁规则，写出不会发生死锁的SQL，比如，只用聚簇索引做where条件，也可以根据MySQL的加锁规则，定位出线上产生死锁的原因。
