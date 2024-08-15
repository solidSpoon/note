# 插入操作中的死锁现象

[InnoDB 中不同 SQL 语句设置的锁](https://dev.mysql.com/doc/refman/8.4/en/innodb-locks-set.html)

在带有唯一索引的表中执行 INSERT 操作时，数据库会在间隙上设置插入意图间隙锁（insert intention gap lock），并对插入的行设置独占锁。然而，这并不会阻止其他事务在相同间隙内插入不同的行。

比如下面这种，间隙为 4 到 7，两个事务分别在这个间隙插入 5 和 6，由于这些行没有冲突，因此不会相互阻塞

```text
索引记录: |  4  |     间隙     |  7  |
事务 1 插入:        ^5
事务 2 插入:            ^6
```

当三个事务插入同一行时，可能会报错死锁

```text
索引记录: |  4  |  间隙  |  7  |
事务 1 插入:       ^5
事务 2 插入:       ^5
事务 3 插入:       ^5
```

下面探究一下报错死锁的原因

## 实验步骤

### 创建报错环境

**创建表：**
```sql
CREATE TABLE t1 (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;
```

**会话 1**

```sql
# 启动事务
START TRANSACTION;

# 插入记录
INSERT INTO t1 VALUES(1);
```

此操作会为值为 1 的行获取一个独占锁。

**会话 2**

```sql
# 启动事务
START TRANSACTION;

# 尝试插入相同的记录
INSERT INTO t1 VALUES(1);
```
由于会话 1 已经持有独占锁，这次插入操作会导致重复键错误，并请求一个共享锁。

**会话 3**


```sql
# 启动事务
START TRANSACTION;

# 尝试插入相同的记录
INSERT INTO t1 VALUES(1);
```

同样，这也会导致重复键错误，并请求一个共享锁。

### 制造报错并分析结果

此时使用 `SHOW ENGINE INNODB STATUS;`，会话 1 的部分如下：

```text
---TRANSACTION 2704, ACTIVE 13 sec
2 lock struct(s), heap size 1128, 1 row lock(s), undo log entries 1
MySQL thread id 47, OS thread handle 140111910045440, query id 4213 10-148-0-7.cilium-agent.kube-system.svc.cluster.local 10.148.0.7 root starting
/* ApplicationName=DataGrip 2024.2.1 */ SHOW ENGINE INNODB STATUS
```

1 row lock(s)
: 持有一个行锁。后面的 s 表示可能是复数，不是共享锁的 s，这里看不出是啥锁。


发现会话 2 和会话 3 都显示类似下面的信息

```text
---TRANSACTION 2713, ACTIVE 2 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1128, 1 row lock(s)
MySQL thread id 57, OS thread handle 140111910315776, query id 4202 10-148-0-7.cilium-agent.kube-system.svc.cluster.local 10.148.0.7 root update
/* ApplicationName=DataGrip 2024.2.1 */ INSERT INTO t1 VALUES(1)
------- TRX HAS BEEN WAITING 2 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 3 page no 4 n bits 72 index PRIMARY of table `mydb`.`t1` trx id 2713 lock mode S locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 000000000a90; asc       ;;
 2: len 7; hex 81000001110110; asc        ;;
```

TRX HAS BEEN WAITING 2 SEC FOR THIS LOCK TO BE GRANTED
: 该线程正在等待锁

index PRIMARY of table `mydb`.`t1` trx id 2713 lock mode S
: 正在等待获取表mydb.t1的PRIMARY索引上的共享锁（S锁）

locks rec but not gap waiting
: 锁定记录但不锁定间隙

**会话 1**

```sql
# 回滚事务
ROLLBACK;
```

这将释放会话 1 持有的独占锁，允许会话 2 和会话 3 获取共享锁。

**观察死锁**

在任何一个会话中运行以下命令以查看 InnoDB 状态
```sql
SHOW ENGINE INNODB STATUS;
```

```text
------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-08-15 02:43:23 140111312307968
*** (1) TRANSACTION:
TRANSACTION 2569, ACTIVE 34 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1128, 2 row lock(s)
MySQL thread id 44, OS thread handle 140111909775104, query id 2361 10-148-0-7.cilium-agent.kube-system.svc.cluster.local 10.148.0.7 root update
/* ApplicationName=DataGrip 2024.2.1 */ INSERT INTO t1 VALUES(1)

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 3 page no 4 n bits 72 index PRIMARY of table `mydb`.`t1` trx id 2569 lock mode S
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;


*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 3 page no 4 n bits 72 index PRIMARY of table `mydb`.`t1` trx id 2569 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;


*** (2) TRANSACTION:
TRANSACTION 2573, ACTIVE 17 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1128, 2 row lock(s)
MySQL thread id 57, OS thread handle 140111910315776, query id 2434 10-148-0-7.cilium-agent.kube-system.svc.cluster.local 10.148.0.7 root update
/* ApplicationName=DataGrip 2024.2.1 */ INSERT INTO t1 VALUES(1)

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 3 page no 4 n bits 72 index PRIMARY of table `mydb`.`t1` trx id 2573 lock mode S
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;


*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 3 page no 4 n bits 72 index PRIMARY of table `mydb`.`t1` trx id 2573 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** WE ROLL BACK TRANSACTION (2)
```
HOLDS THE LOCK(S)
: 会话 1 回滚后，会话 2 和会话 3 都获得了共享锁（S)

WAITING FOR THIS LOCK TO BE GRANTED -> X insert intention waiting (等待获取插入意图锁)
: 可见会话 2 和会话 3 都想把共享锁（S）升级为独占锁（X），导致死锁

WE ROLL BACK TRANSACTION (2)
: 其中一个会话被回滚

## 原因

- **死锁原因：** 插入时如果发生重复键错误，则会在重复索引记录上设置共享锁。会话 2 和会话 3 都持有共享锁，但由于需要升级为独占锁才能插入相同的行，所以相互阻塞，导致死锁。
- **死锁检测和处理：** InnoDB 通常会自动检测到死锁，并回滚其中一个事务以解决问题。