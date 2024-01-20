# MySql 死锁时的一种解决办法

查看数据库的线程情况

```sql
show processlist;
```

| Id       | User         | Host               | db          | Command | Time | State | Info |
|----------|--------------|--------------------|-------------|---------|------|-------|------|
| 9930577  | business_web | 192.168.1.21:45503 | business_db | Sleep   | 153  |       | NULL |
| 9945825  | business_web | 192.168.1.25:49518 | business_db | Sleep   | 43   |       | NULL |
| 9946322  | business_web | 192.168.1.23:44721 | business_db | Sleep   | 153  |       | NULL |
| 9960167  | business_web | 192.168.3.28:2409  | business_db | Sleep   | 93   |       | NULL |
| 9964484  | business_web | 192.168.1.21:24280 | business_db | Sleep   | 7    |       | NULL |
| 9972499  | business_web | 192.168.3.28:35752 | business_db | Sleep   | 13   |       | NULL |
| 10000117 | business_web | 192.168.3.28:9149  | business_db | Sleep   | 6    |       | NULL |
| 10002523 | business_web | 192.168.3.29:42872 | business_db | Sleep   | 6    |       | NULL |
| 10007545 | business_web | 192.168.1.21:51379 | business_db | Sleep   | 155  |       | NULL |

查看 `State` 字段有没有 wait 状态的线程，如果没有看到正在执行的慢SQL记录线程，再去查看innodb的事务表INNODB_TRX，看下里面是否有正在锁定的事务线程，看看ID是否在show
full processlist里面的sleep线程中，如果是，就证明这个sleep的线程事务一直没有commit或者rollback而是卡住了，我们需要手动kill掉。

```sql
SELECT * FROM information_schema.INNODB_TRX;
```
```Bash
*************************** 1. row ***************************
trx_id: 20866
trx_state: LOCK WAIT
trx_started: 2014-07-31 10:42:35
trx_requested_lock_id: 20866:617:3:3
trx_wait_started: 2014-07-30 10:42:35
trx_weight: 2
trx_mysql_thread_id: 9930577
trx_query: delete from dltask where id=1
trx_operation_state: starting index read
trx_tables_in_use: 1
trx_tables_locked: 1
trx_lock_structs: 2
trx_lock_memory_bytes: 376
trx_rows_locked: 1
trx_rows_modified: 0
trx_concurrency_tickets: 0
trx_isolation_level: READ COMMITTED
trx_unique_checks: 1
trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
trx_adaptive_hash_latched: 0
trx_adaptive_hash_timeout: 10000
trx_is_read_only: 0
trx_autocommit_non_locking: 0
```

终止上面的线程

```sql
kill 9930577;
```

[MySql 死锁时的一种解决办法](https://www.cnblogs.com/farb/p/MySqlDeadLockOneOfSolutions.html)