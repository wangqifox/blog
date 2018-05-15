---
title: MySQL加锁
date: 2017/12/10 16:39:00
---

## MySQL加锁处理分析

MySQL InnoDB存储引擎，实现的是基于多版本的并发控制协议——MVCC(Multi-Version Concurrency Control)（注：与MVCC相对的，是基于锁的并发控制，Lock-Based Concurrency Control）。MVCC最大的好处：读不加锁，读写不冲突。在读多写少的OLTP应用中，读写不冲突是非常重要的，极大的增加了系统的并发性能。

在MVCC并发控制中，读操作可以分为两类：快照读(snapshot read)与当前读(current read)。快照读，读取的是记录的可见版本(有可能是历史版本)，不用加锁。当前读，读取的是记录的最新版本，并且，当前读返回的记录，都会加上锁，保证其他事务不会再并发修改这条记录。
<!--more-->
以MySQL InnoDB为例：

- 快照读：简单的select操作，属于快照读，不加锁

	- select * from table where ?;

- 当前读：特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁

	- select * from table where ? lock in share mode;
	- select * from table where ? for update;
	- insert into table values(...);
	- update table set ? where ?;
	- delete from table where ?;

	所有以上的语句，都属于当前读，读取记录的最新版本。并且，读取之后，还需要保证其他并发事务不能修改当前记录，对读取记录加锁。其中，除了第一条语句，对读取记录加S锁(共享锁)外，其他的操作，都加的是X锁(排他锁)。
	
当Update SQL被发给MySQL后，MySQL Server会根据where条件，读取第一条满足条件的记录，然后InnoDB引擎会将第一条记录返回，并加锁(current read)。待MySQL Server收到这条加锁的记录之后，会再发起一个Update请求，更新这条记录。一条记录操作完成后，再读取下一条记录，直至没有满足条件的记录为止。因此，Update操作内部，就包含了一个当前读。同理，Delete操作也一样。Insert操作会稍微有些不同，简单来说，就是Insert操作可能会触发Unique Key的冲突检查，也会进行一个当前读。

针对一条当前读的SQL语句，InnoDB与MySQL Server的交互，是一条一条进行的，因此，加锁也是一条一条进行的。先对一条满足条件的记录加锁，返回给MySQL Server，做一些DML操作；然后再读取下一条加锁，直至读取完毕。

### Cluster Index：聚簇索引

InnoDB存储引擎的数据组织方式，是聚簇索引表：完整的记录，存储在主键索引中，通过主键索引，就可以获取记录所有的列。

### 两阶段加锁

2PL(Two-Phase Locking)就是将加锁/解锁分为两个完全不相交的阶段。加锁阶段：只加锁，不放锁；解锁阶段：只放锁，不加锁

### 隔离级别

MySQL/InnoDB定义的4种隔离级别：

- Read Uncommited

	可以读取未提交记录。此隔离级别，不会使用，忽略
	
- Read Committed (RC)

	快照读忽略
	
	针对当前读，RC隔离级别保证对读取到的记录加锁(记录锁)，存在幻读现象

- Repeatable Read (RR)

	快照读忽略
	
	针对当前读，RR隔离级别保证对读取到的记录加锁(记录锁)，同时保证对读取的范围加锁，新的满足查询条件的记录不能够插入(间隙锁)，不存在幻读现象。
	
- Serializable

	从MVCC并发控制退化为基于锁的并发控制。部分快照读与当前读，所有的读操作均为当前读，读加读锁(S锁)，写加写锁(X锁)。
	
	Serializable隔离级别下，读写冲突，因此并发度急剧下降，在MySQL/InnoDB下不建议使用
	

语句`delete from t1 where id = 10`加锁分析

- 组合1：id主键 + RC

只需要将主键上，id=10的记录加上X锁即可

- 组合2：id唯一索引 + RC

首先会将unique索引上的id=10索引记录加上X锁，同时，会根据读取到的name列，回主键索引(聚簇索引)，然后将聚簇索引上的name='d'对应的主键索引项加X锁。

- 组合3：id非唯一索引 + RC

首先，id列索引上，满足id=10查询条件的记录，均已加锁。同时，这些记录对应的主键索引上的记录也都加上了锁。

- 组合4：id无索引 + RC

若id列上没有索引，SQL会走聚簇索引的全扫描进行过滤，由于过滤是由MySQL Server层面进行的。因此每条记录，无论是否满足条件，都会被加上X锁。但是，为了效率考量，MySQL做了优化，对于不满足条件的记录，会在判断后释放锁，最终持有的，是满足条件的记录上的锁，但是不满足条件的记录上的加锁/放锁动作不会省略。

- 组合5：id主键 + RR

加锁与"组合一：id主键，Read Committed"一致

- 组合6：id唯一索引 + RR

加锁与"组合二：id唯一索引，Read Committed"一致。

- 组合7：id非唯一索引 + RR

首先，通过id索引定位到第一条满足查询条件的记录，加记录上的X锁，加GAP上的GAP锁，然后加主键聚簇索引上的记录X锁，然后返回；然后读取下一条，重复进行。直至进行到第一条不满足条件的记录[11,f]，此时，不需要加记录X锁，但是仍旧需要加GAP锁，最后返回结束

- 组合8：id无索引 + RR

在Repeatable Read隔离级别下，如果进行全表扫描的当前读，那么会锁上表中的所有记录，同时会锁上聚簇索引内的所有GAP，杜绝所有的并发 更新/删除/插入 操作。当然也可以通过触发semi-consistent read，来缓解加锁开销与并发影响，但是semi-consistent read本身也会带来其他的问题，不建议使用

- 组合9：Serializable

对于SQL2: `delete from t1 where id = 10`来说，Serializable隔离级别与Repeatable Read隔离级别完全一致

Serializable隔离级别，影响的是SQL1: `select * from t1 where id = 10`这条SQL，在RC, RR隔离级别下，都是快照读，不加锁。但是在Serializable隔离级别，SQL1会加读锁，也就是说快照读不复存在，MVCC并发控制降级为Lock-Based CC

在MySQL/InnoDB中，所谓的读不加锁，并不适用于所有的情况，而是隔离级别相关的。Serializable隔离级别，读不加锁就不再成立，所有的读操作，都是当前读

### 复杂SQL分析

	Table: t1(id primary key, userid, blogid, pubtime,comment)
	Index: idx_t1_pu(puptime, userid)
	SQL: delete from t1 where pubtime > 1 and pubtime < 20 and userid = 'hdc' and comment is not NULL
	
where条件拆分：

- Index key: pubtime > 1 and pubtime < 20。此条件，用于确定SQL在idx_t1_pu索引上的查询范围
- Index Filter: userid = 'hdc'。此条件，可以再idx_t1_pu索引上进行过滤，但不属于Index Key
- Table Filter: comments is not NULL。此条件，在idx_t1_pu索引上无法过滤，只能在聚簇索引上过滤

在Repeatable Read隔离级别下，由Index Key所确定的范围，被加上了GAP锁；Index Filter锁给定的条件(userid='hdc')何时过滤，视MySQL的版本而定，在MySQL 5.6版本之前，不支持Index Condition Pushdown，因此Index Filter在MySQL Server层过滤，在5.6后支持了Index Condition Pushdown，则在index上过滤。若不支持ICP，不满足Index Filter的记录，也需要加上记录X锁，若支持ICP，则不满足Index Filter的记录，无需加记录X锁；而Table Filter对应的过滤条件，则在聚簇索引中读取后，在MySQL Server层面过滤，因此聚簇索引上也需要X锁。

在Repeatable Read隔离级别下，针对一个复杂的SQL，首先需要提取其where条件。Index Key确定的范围，需要加上GAP锁；Index Filter过滤条件，视MySQL版本是否支持ICP，若支持ICP，则不满足Index Filter的记录，不加X锁，否则需要X锁；Table Filter过滤条件，无论是否满足，都需要加X锁。

## SQL中的where条件，在数据库中提取与应用浅析

所有SQL的where条件，均可归纳为3大类：Index Key(First Key & Last Key), Index Filter, Table Filter

- Index Key

	用于确定SQL查询在索引中的连续范围(起始范围+结束范围)的查询条件，被称为Index Key。由于一个范围，至少包含一个起始与一个终止，因此Index Key也被拆分为Index First Key和Index Last Key，分别用于定位索引查找的起始，以及索引查询的终止条件。

	- Index First Key
		
		用于确定索引查询的起始范围。提取规则：
		
		- 从索引的第一个键值开始，检查其在where条件中是否存在，若存在并且条件是=, >=，则将对应的条件加入Index First Key之中，继续读取索引的下一个键值，使用同样的提取规则；
		- 若存在并且条件是>，则将对应的条件加入Index First Key中，同时终止Index First Key的提取；
		- 若不存在，同样终止Index First Key的提取。
		
	- Index Last Key

		Index Last Key的功能与Index First Key正好相反，用于确定索引查询的终止范围。提取规则：
		
		- 从索引的第一个键值开始，检查其在where条件中是否存在，若存在并且条件是=, <=，则将对应条件加入到Index Last Key中，继续提取索引的下一个键值，使用同样的提取规则；
		- 若存在并且条件是<，则将条件加入到Index Last Key中，同时终止提取；
		- 若不存在，同样终止Index Last Key的提取。
		
- Index Filter

	Index Filter的提取规则：同样从索引的第一列开始，检查其在where条件中是否存在：
	
	- 若存在并且where条件仅为=，则跳过第一列继续检查索引下一列，下一索引列采取与索引第一列同样的提取规则；
	- 若where条件为>=, >, <, <=其中的几种，则跳过索引第一列，将其余where条件中索引相关列全部加入到Index Filter之中；
	- 若索引第一列的where条件包含=, >=, >, <, <=之外的条件，则将此条件以及其余where条件中索引相关列全部加入到Index Filter之中；
	- 若第一列不包含查询条件，则将所有索引相关条件均加入到Index Filter之中

- Table Filter

	所有不属于索引列的查询条件，均归为Table Filter之中。

### Index Key/Index Filter/Table Filter小结

- Index First Key，只是用来定位索引的起始范围，因此只在索引第一次Search Path（沿着索引B+树的根节点一直遍历，到索引正确的叶节点位置）时使用，一次判断即可。
- Index Last Key，用来定位索引的终止范围，因此对于起始范围之后读到的每一条索引记录，均需要判断是否已经超过了Index Last Key的范围，若超过，则当前查询结束
- Index Filter，用于过滤索引查询范围中不满足查询条件的记录，因此对于索引范围中的每一条记录，均需要与Index Filter进行对比，若不满足Index Filter则直接丢弃，继续读取索引下一条记录
- Table Filter，则是最后一道where条件的防线，用于过滤通过前面索引的层层考验的记录，此时的记录已经满足了Index First Key与Index Last Key构成的范围，并且满足Index Filter的条件，回表读取了完整的记录，判断完整记录是否满足Table Filter中的查询条件，同样的，若不满足，跳过当前记录，继续读取索引的下一条记录，若满足，则返回记录，此记录满足了where的所有条件，可以返回给前端用户。

MySQL 5.6中引入的Index Condition Pushdown，究竟是将什么Push Down到索引层面进行过滤了？答案是Index Filter。在MySQL 5.6之前，并不区分Index Filter与Table Filter，统统将Index First Key与Index Last Key范围内的索引记录，回表读取完整记录，然后返回给MySQL Server层进行过滤。而在MySQL 5.6之后，Index Filter与Table Filter分离，Index Filter下降到InnoDB的索引层面进行过滤，减少了回表与返回MySQL Server层的记录交互开销，提高了SQL的执行效率

## InnoDB多版本(MVCC)实现

InnoDB为了实现多版本的一致读，采用的是基于回滚段的协议。

InnoDB表数据的组织方式为主键聚簇索引。由于采用索引组织表的结构，记录的ROWID是可变的(索引页分裂的时候，Structure Modification Operation, SMO)，因此二级索引中采用的是(索引键值，主键键值)的组合来唯一确定一条记录。

无论是聚簇索引，还是二级索引，其每条记录都包含了一个DELETED BIT位，用于标识该记录是否是删除记录。除此之外，聚簇索引记录还有两个系统列：DATA_TRX_ID, DATA_ROLL_PTR。

- DATA_TRX_ID表示产生当前记录项的事务ID；
- DATA_ROLL_PTR指向当前记录项的undo信息。

聚簇索引中包含版本信息（事务号+回滚指针），二级索引不包含版本信息

### Read View

InnoDB默认的隔离级别为Repeatable Read(RR)，可重复读。InnoDB在开始一个RR读之前，会创建一个Read View。Read View用于判断一条记录的可见性。Read View定义在read0read.h文件中，其中最主要的与可见性相关的属性如下：

```
dulint	low_limit_id;		// 事务号 >= low_limit_id的记录，对于当前Read View都是不可见的
dulint	up_limit_id;		// 事务号 < up_limit_id，对于当前Read View都是可见的
ulint	n_trx_ids;		// trx_ids数组元素的数量
dulint*	trx_ids;		// 另外对于读操作不可见的事务id，这些是读操作被序列化时活动的事务，除了读事务本身
dulint		creator_trx_id;	// 创建事务的事务id
```

简单来说，Read View记录读开始时，所有的活动事务，这些事务所做的修改对于Read View是不可见的。除此之外，所有其他的小于创建Read View的事务号的所有记录均可见。可见包括两层含义：

- 记录可见，且Deleted bit = 0; 当前记录是可见的有效记录
- 记录可见，且Deleted bit = 1; 当前记录是可见的删除记录。此记录在本事务开始之前，已经删除

### 案例

- create table and index

	```
	create table test(id int primary key, comment char(50)) engine=InnoDB
	create index test_idx on test(comment);
	```
- insert

	```
	insert into test values(1, 'aaa');
	insert into test values(2, 'bbb');
	```

- update primary key

	```
	update test set id = 9 where id = 1;
	```
	
	老版本仍旧存储在聚簇索引之中，其DATA_TRX_ID被设置为1811，Deleted bit设置为1，undo中记录了前镜像的事务id=1809。新版本DATA_TRX_ID也为1811。虽然新老版本是一条记录，但是在聚簇索引中是通过两条记录来标识的。同时，由于更新了主键，二级索引也需要做相应的更新(二级索引中包含主键项)。
	
- update non-primary key with different value

	```
	update test set comment = 'ccc' where id = 9;
	```
	
	更新二级索引的键值时，聚簇索引本身并不会产生新的记录项，而是将旧版本信息记录在undo之中。与此同时，二级索引将会产生新的索引项，其PK值保持不变，指向聚簇索引的同一条记录。二级索引页面中有一个MAX_TRX_ID，此值记录的是更新二级索引页面的最大事务ID。通过MAX_TRX_ID的过滤，InnoDB能够实现大部分的辅助索引覆盖性扫描(仅仅扫描辅助索引，不需要回聚簇索引)。

- update non-primary key with same value

	```
	update test set comment = 'bbb' where id = 2 and comment = 'bbb'
	```

聚簇索引仍旧会更新，但是二级索引保持不变

### 总结

1. 无论是聚簇索引，还是二级索引，只要其键值更新，就会产生新版本。将老版本数据deleted bit设置为1，同时插入新版本
2. 对于聚簇索引，如果更新操作没有更新primary key，那么更新不会产生新版本，而是在原有版本上进行更新，老版本进入undo表空间，通过记录上的undo指针进行回滚
3. 对于二级索引，如果更新操作没有更新其键值，那么二级索引记录保持不变
4. 对于二级索引，更新操作无论更新primary key，或者是二级索引键值，都会导致二级索引产生新版本数据
5. 聚簇索引设置记录deleted bit时，会同时更新DATA_TRX_ID列。老版本DATA_TRX_ID进入undo表空间；二级索引设置deleted bit时，不写入undo·

### 可见性判断

#### 主键查找

1. `select * from test where id = 1;`

	- 针对测试1，如果1811(DATA_TRX_ID) < read_view.up_limit_id，证明被标记为删除的记录1可见。删除可见 -> 无记录返回
	- 针对测试1，如果1811(DATA_TRX_ID) >= read_view.low_limit_id，证明被标记为删除的记录1不可见。通过DATA_ROLL_PTR回滚记录，得到DATA_TRX_ID = 1809。如果1809可见，则返回记录(1, aaa)，否则无记录返回。
	- 针对测试1，如果up_limit_id, low_limit_id都无法判断可见性，那么遍历read_view中的trx_ids，依次对比事务id，如果在DATA_TRX_ID在trx_ids数组中，则不可见（更新未提交）

2. `select * from test where id = 9;`

	- 针对测试2，如果1816可见，返回(9, ccc)
	- 针对测试2，如果1816不可见，通过DATA_ROLL_PTR回滚到1811，如果1811可见，返回(9, aaa)
	- 针对测试2，如果1811不可见，无结果返回

3. `select * from test where id > 0`

	- 针对测试1，索引中，满足条件的同一记录，有两个版本（版本1，delete bit = 1）。那么是否会一条记录返回两次呢？必定不会，这是因为pk=1的可见性与pk=9的可见性是一致的，同时pk=1是标记了deleted bit的版本。如果事务ID=1811可见，那么pk=1 delete可见，无记录返回，pk=9返回记录；如果1811不可见，回滚到1809可见，那么pk=1返回记录，pk=9回滚后无记录

##### 总结

1. 通过主键查找记录，需要配合read_view，记录DATA_TRX_ID，记录DATA_ROLL_PTR指针共同判断。
2. read_view用于判断当前记录是否可见（判断DATA_TRX_ID）。DATA_ROLL_PTR用于将当前记录回滚到前一版本

#### 非主键查找

`select comment from test where comment > ""`

- 针对测试2，二级索引，当前页面的最大更新事务MAX_TRX_ID = 1816。如果MAX_TRX_ID < read_view.up_limit_id，当前页面所有数据均可见，本页面可以进行索引覆盖性扫描。丢弃所有deleted bit = 1的记录，返回deleted bit = 0的记录；此时返回(ccc)
- 针对测试2，二级索引，如果当前页面不能满足MAX_TRX_ID < read_view.up_limit_id，说明当前页面无法进行索引覆盖性扫描，此时需要针对每一项，到聚簇索引中判断可见性。回到测试2，二级索引中有两项pk=9（一项deleted bit = 1，另一个为0），对应的聚簇索引中只有一项pk=9。如果保证通过二级索引过来的同一记录的多个版本，在聚簇索引中最多只能被返回一次？如果当前事务id 1811可见。二级索引pk=9的记录（两项），通过聚簇索引的undo，都定位到了同一记录项。此时，InnoDB通过以下的一个表达式，来保证来自二级索引，指向同一聚簇索引记录的多个版本项，有且最多仅有一个版本将会返回数据：

	```
	if (clust_rec 
	&& (old_vers || rec_get_deleted_flag(rec, dict_table_is_comp(sec_index->table))) 
	&& !row_sel_sec_rec_is_for_clust_rec(rec, sec_index, clust_rec, clust_index))
	```

	满足if判断的所有聚簇索引记录，都直接丢弃，以上判断的逻辑如下：
	
	1. 需要回聚簇索引扫描，并且获得记录
	2. 聚簇索引记录为回滚版本，或者二级索引中的记录为删除版本
	3. 聚簇索引项，与二级索引项，其键值并不相等

	我们通过二级索引记录，定位聚簇索引记录，定位之后，还需要**再次检查聚簇索引记录是否仍旧是我在二级索引中看到的记录**。如果不是则直接丢弃；如果是，则返回。

	根据此条件，结合查询与测试2中的索引结构。可见版本为事务1811。二级索引中的两项pk=9都能通过聚簇索引回滚到1811版本。但是，二级索引记录(ccc, 9)与聚簇索引回滚后的版本(aaa, 9)不一致，直接丢弃。只有二级索引记录(aaa, 9)保持一致，直接返回。
	
	##### 总结
	
	1. 二级索引的多版本可见性判断，需要通过聚簇索引完成
	2. 二级索引页面中保存了MAX_TRX_ID，可以快速判断当前页面中，是否所有项均可见，可以实现二级索引页面级别的索引覆盖扫描。一般而言，此判断是满足条件的，保证了索引覆盖扫描(index only scan)的高效性。
	3. 二级索引中的项，需要与聚簇索引中可见性进行比较，保证聚簇索引中的可见项与二级索引中的数据一致

### Purge流程

InnoDB由于要支持多版本协议，因此无论是更新、删除，都只是设置记录上的deleted bit标记位，而不是真正地删除记录。后续这些记录的真正删除，是通过Purge后台进程实现的。Purge进程定期扫描InnoDB的undo，按照先读老undo，再读新undo的顺序，读取每条undo read。对于每一条undo record，判断其对应的记录是否可以被purge(purge进程有自己的read view，等同于进程开始时最老的活动事务之前的view，保证purge的数据，一定是不可见数据，对任何人来说)，如果可以purge，则构造完整记录(row_purge_parse_undo_rec)。然后按照先purge二级索引，最后purge聚簇索引的顺序，purge一个操作生成的旧版本完整记录。

#### 总结

1. purge是通过遍历undo实现的
2. purge的粒度是一条记录上的一个操作。如果一条记录被update了3次，产生了3个old版本，均可purge。那么purge读取undo，对于每一个操作，都会调用一次purge。一个purge删除一个操作产生的old版本（按照操作从老到新的顺序）
3. purge按照先二级索引，最后聚簇索引的顺序进行
4. purge二级索引，通过构造出的索引项进行查找定位。不能直接针对某个二级页面进行，因为不知道记录的存放page
5. 对于二级索引设置deleted bit而不需要记录undo，因为purge是根据聚簇索引undo实现。因此二级索引deleted bit被设置为1的项，没有记录undo，仍旧可以被purge
6. purge是一个耗时的操作。二级索引的purge，需要search_path定位数据，相当于每个二级索引，都做了一次index unique scan
7. 一次delete操作，IO翻番。第一次IO是将记录的deleted bit设置为1；第二次的IO是将记录删除。

## 一个最不可思议的MySQL死锁分析

unique查询，三种情况，对应三种加锁策略，总结如下：

- 找到满足条件的记录，并且记录有效：则对记录加X锁，No Gap锁(lock_mode X locks rec but not gap)
- 找到满足条件的记录，但是记录无效（标识为删除的记录）：则对记录加next key锁（同时锁住记录本身，以及记录之前的Gap: lock_mode X）
- 未找到满足条件的记录：则对第一个不满足条件的记录加Gap锁，保证没有满足条件的记录插入(locks gap before rec)

### 死锁预防策略

InnoDB引擎内部(或者说是所有)，有多种锁类型：事务锁(行锁、表锁)，Mutex(保护内部的共享变量操作)，RWLock(又称之为Latch，保护内部的页面读取与修改)。

InnoDB每个页面为16K，读取一个页面时，需要对页面加S锁，更新一个页面时，需要对页面加上X锁。任何情况下，操作一个页面，都会对页面加锁，页面锁加上之后，页面内存储的索引记录才不会被并发修改。

因此，为了修改一条记录，InnoDB内部如何处理：

1. 根据给定的查询条件，找到对应的记录所在页面
2. 对页面加上X锁(RWLock)，然后在页面内寻找满足条件的记录
3. 在持有页面锁的情况下，对满足条件的记录加事务锁（行锁：根据记录是否满足查询条件，记录是否已经被删除，分别对应于上面提到的3种加锁策略之一）
4. 死锁预防策略：相对于事务锁，页面锁是一个短期持有的锁，而事务锁（行锁、表锁）是长期持有的锁。因此，为了防止页面锁与事务锁之间产生死锁。InnoDB做了死锁预防的策略：持有事务锁（行锁、表锁），可以等待获取页面锁；但反之，持有页面锁，不能等待持有事务锁。
5. 根据死锁预防策略，在持有页面锁，加行锁的时候，如果行锁需要等待，则释放页面锁，然后等待行锁。此时，行锁获取没有任何锁保护，因此加上行锁之后，记录可能已经被并发修改。**因此，此时要重新加回页面锁，重新判断记录的状态，重新在页面锁的保护下，对记录加锁。**如果此时记录未被并发修改，那么第二次加锁能够很快完成，因为已经持有了相同模式的锁。**但是，如果记录已经被并发修改，那么，就有可能导致本文前面提到的死锁问题。**

此类死锁产生的几个前提：

- delete操作，针对的是唯一索引上的等值查询的删除；(范围下的删除，也会产生死锁，但是死锁的场景，跟文本分析的场景有所不同)
- 至少有3个(或以上)的并发删除操作
- 并发删除操作，有可能删除到同一条记录，并且保证删除的记录一定存在
- 事务的隔离级别设置为Repeatable Read，同时未设置innodb_locks_unsafe_for_binlog参数(此参数默认为false)；(Read Committed隔离级别，由于不会加Gap锁，不会有next key，因此不会产生死锁)
- 使用的是InnoDB存储引擎



引用：

- http://hedengcheng.com/?p=844
- http://hedengcheng.com/?p=148
- http://hedengcheng.com/?p=771

