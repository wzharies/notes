

## 第21章 事务隔离级别和MVCC

### MVCC版本链

InnoDB的存储引擎的每个记录都有trx_id还有roll_pointer2个隐藏列

**trx_id**：当事务对某条记录进行更改的时候，会把事务id赋值给trx_id。

**roll_pointer**：修改记录的时候，roll_pointer会指向上次修改前的记录。

每次更改记录的时候，都会记录一条undo log，然后用roll_pointer从最新的修改指向上一条修改。这个链表就是**版本链**。版本链的头指针指针当前版本



### ReadView

RU：运行读到未提交的数据，所以直接读取最新的数据就可以了

RC：RR：必须保证读到已提交的数据（需要ReadView）

SR：用加锁的方式来访问



ReadView会用到下面4个变量

m_ids：当前活跃的事务id列表

min_trx_id：当前活跃的事务id中的最小值min(m_ids)

max_trx_id: 系统应该分配的**下一个事务id**

create_trx_id: 生成ReadView的事务id

具体的比较过程：

1. 如果trx_id == create_trx_id，说明正在方面它自己修改的记录，所以该版本可以被当前事务访问。
2. 如果trx_id < min_trx_id。说明是已经提交过的事务，可以访问
3. 如果trx_id > max_trx_id。说明事务是在ReadView之后才开启，不能访问
4. 如果 min_trx_id<=trx_id <=max_trx_id。需要二次判断
   * 如果trx_id在m_ids中。说明事务是活跃的，不能访问
   * 如果trx_id不在m_ids中。说明事务已经结束，可以访问

RC：**每次读取**前都生成一个ReadView

RR：只在**第一次读取**时生成一个ReadView



### Purge

insert undo日志在事务提交之后就可以立即删除了，而update undo log需要支持MVCC，因此不能立即删除。

事务提交后，会把这个事务产生的update undo log放入History链表中。

删除支持：为了支持MVCC，删除是采用delete Mark标记一个log被删除

update undo log删除需要系统**最早产生的ReadView不在访问它们**了，最早的ReadView的创建

## 第22章 锁

### 一致性读或者快照读

普通的select语句在RC或者RR级别下都是一致性读

### 锁定读

包括X锁和S锁

锁定读的语句：

select .... LOCK IN SHARE MODE;（加S锁）

select .... FOR UPDATRE; (加X锁)

### 多粒度锁

表级别X和S锁

为了判断表级锁和行级锁是否冲突，并且**不需要遍历判断**，可以采用IS和IX这2种意向锁

之后再使用行锁的时候，需要先对表使用意向锁，减少表级锁的判断。

### InnoDB中的锁

#### 表级锁

alter table、drop table等DDL语句中会使用Metadata Lock，不会用表级锁。用到的场景很少

#### 行级锁

* Record Lock
* Gap Lock(解决幻读)
* Next-Key Lock（Record Lock + Gap Lock）
* Insert Intention Lock（插入的时候需要用到，如果碰到Gap Lock，会生成一个Insert Intention Lock）