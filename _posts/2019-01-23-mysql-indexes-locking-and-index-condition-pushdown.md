---
layout: post
title: 'MySQL: Indexes, Locking, and Index Condition Pushdown'
---

## Indexes and Locking

I spent the past weekend browsing [_High Performance MySQL_](http://www.highperfmysql.com/).

The chapter on indexes has a section describing the impact indexes can have on locking.

Particularly, indexes can mean fewer rows are locked as the result of a query.

How so? InnoDB locks rows when they're accessed and an index allows you reduce the number of rows accessed.

Less locking is good. We skip the overhead from locking and reduce lock contention (increasing concurrency).


Here's a simple example (<small>_nb. I'm using MySQL 8_</small>):
```
CREATE TABLE `payments` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `amount` BIGINT NOT NULL,
  `description` VARCHAR(255) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```
```
INSERT INTO `payments` (`amount`, `description`) VALUES
(100, "dolla"),
(200, "two dolla"),
(200, "another one"),
(300, "triple triple"),
(400, "4 more"),
(500, "feelin alive");
```

Selecting a single row with the primary key:
```
mysql> explain select * from payments where id = 3;
+----|-------------|----------|------------|-------|---------------|---------|---------|-------|------|----------|-------+
| id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----|-------------|----------|------------|-------|---------------|---------|---------|-------|------|----------|-------+
|  1 | SIMPLE      | payments | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL  |
+----|-------------|----------|------------|-------|---------------|---------|---------|-------|------|----------|-------+
1 row in set, 1 warning (0.00 sec)
```
It's examining 1 row.

If you run the above query as `BEGIN; SELECT ... FOR UPDATE` you can check the locks held with `select * from performance_schema.data_locks`.

What about a range query?
```
mysql> explain select * from payments where id < 4 and id <> 1;
+----|-------------|----------|------------|-------|---------------|---------|---------|------|------|----------|-------------+
| id | select_type | table    | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----|-------------|----------|------------|-------|---------------|---------|---------|------|------|----------|-------------+
|  1 | SIMPLE      | payments | NULL       | range | PRIMARY       | PRIMARY | 8       | NULL |    3 |   100.00 | Using where |
+----|-------------|----------|------------|-------|---------------|---------|---------|------|------|----------|-------------+
```
It's examining 3 rows instead of 2.

We can check the locks again with a `BEGIN; SELECT ... FOR UPDATE` and you'll see that the row with id 1 is locked even though it's excluded in the where clause.

Or, a more fun check: leave the transaction open, start another client, and watch `SELECT * FROM payments WHERE id = 1 FOR UPDATE` wait on the lock.

What gives? MySQL is telling the storage engine "hey start at the beginning of the index and keep going until payment id < 4 is false". MySQL then filters out `id <> 1`. But, that means the storage engine doesn't know it shouldn't lock the first row!

## Index Condition Pushdown

MySQL 5.6 introduced [Index Condition Pushdown (ICP)](https://dev.mysql.com/doc/refman/8.0/en/index-condition-pushdown-optimization.html) as an optimization for this (though, with the goal of reducing full-row reads and I/O, not necessarily locking).

Here's a new example, with two caveats to note:
1. MySQL will not use ICP if there is a covering index
2. InnoDB tables only use ICP for secondary indexes

```
-- `description` isn't indexed, so we don't have a covering index
-- Add a secondary index
ALTER TABLE `payments` ADD KEY `idx_amount` (`amount`);
```
```
mysql> explain select * from payments where amount < 400 and amount <> 100;
+----|-------------|----------|------------|-------|---------------|------------|---------|------|------|----------|-----------------------+
| id | select_type | table    | partitions | type  | possible_keys | key        | key_len | ref  | rows | filtered | Extra                 |
+----|-------------|----------|------------|-------|---------------|------------|---------|------|------|----------|-----------------------+
|  1 | SIMPLE      | payments | NULL       | range | idx_amount    | idx_amount | 8       | NULL |    4 |   100.00 | Using index condition |
+----|-------------|----------|------------|-------|---------------|------------|---------|------|------|----------|-----------------------+
1 row in set, 1 warning (0.00 sec)
```
The `Extra: Using index condition` indicates it's using ICP.

The locks are the fun part though!

Run the query with a `BEGIN; SELECT ... FOR UPDATE` and check the locks with `select * from performance_schema.data_locks`.

The rows that are returned (ids 2, 3, and 4) have two locks: one for the `idx_amount` and one for the `PRIMARY` index.
The other rows that are "touched" (ids 1 and 5) only have the lock on `idx_amount`.

This is ICP in action!

InnoDB uses a clustering index, so to fetch the full-row (for the description) we use the primary index. 

But, with ICP the first row is filtered at the storage engine layer so we don't need to fetch the full-row.

Test it out in another mysql client:
```
-- Use the idx_amount index
begin;
select * from payments where amount = 100 for update;
-- waiting for lock
```
vs
```
-- Use the PRIMARY index
begin;
select * from payments where id = 1 for update;
+----|--------|-------------+
| id | amount | description |
+----|--------|-------------+
|  1 |    100 | dolla       |
+----|--------|-------------+
1 row in set (0.00 sec)
```

