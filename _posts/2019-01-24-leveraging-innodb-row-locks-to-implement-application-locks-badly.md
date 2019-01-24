---
layout: post
title: Leveraging InnoDB Row Locks to Implement Application Locks (Badly)
---

MySQL already has the locking functions `GET_LOCK()` and `RELEASE_LOCK()` that can be used for this purpose. (And, often, MySQL isn't the right tool for the job anyways).

But... I want to demonstrate some locking quirks that make a custom-rolled, pessimistic locking solution have bad behaviour.

Here's a table that keeps track of loans made to a particular customer:
```
CREATE TABLE `loans` (
    `id` BIGINT NOT NULL AUTO_INCREMENT,
    `customer_id` BIGINT NOT NULL,
    `amount` BIGINT NOT NULL,
    PRIMARY KEY (`id`)
);
```

The table is sparse; if the customer doesn't have any active loans, there won't be a row in the table.

## The Problem

We want to make sure that we don't accidentally approve a loan that will exceed a limit. 

You can imagine some code like this:
```
BEGIN;

SELECT sum(amount) FROM loans WHERE customer_id = 1234;

-- If sum(amount) + new_amount <= LIMIT
INSERT INTO loans (customer_id, amount) VALUES (1234, new_amount);

-- Else abort

COMMIT;
```

But wait! If two transactions are running concurrently, they won't see each others updates and we could potentially loan too much money.

## SELECT ... FOR UPDATE

An attempt to prevent this might use `SELECT ... FOR UPDATE`:
```
BEGIN;

-- This will block other transactions
SELECT sum(amount) FROM loans WHERE customer_id = 1234 FOR UPDATE;

-- If sum(amount) + new_amount <= LIMIT
INSERT INTO loans (customer_id, amount) VALUES (1234, new_amount);

-- Else abort

COMMIT;
```

But there's a catch. If there aren't any rows the `SELECT ... FOR UPDATE` trick doesn't work; there isn't anything to lock and therefore block a concurrent transaction.

Remember, the table is sparse, so if the first loan(s) are being taken out we might accidentally overcommit funds.

## "Mutexes" Table

Let's introduce a sparse `loan_mutexes` table:
```
CREATE TABLE `loan_mutexes` (
    `customer_id` BIGINT NOT NULL,
    PRIMARY KEY (`customer_id`)
);
```

```
BEGIN;

-- Inserting will grab a row lock, blocking other transactions
-- A deleted row is still locked until the transaction commits or rolls back
INSERT INTO loan_mutexes (customer_id) VALUES (1234);
DELETE FROM loan_mutexes WHERE customer_id = 1234;

SELECT sum(amount) FROM loans WHERE customer_id = 1234;

-- If sum(amount) + new_amount <= LIMIT
INSERT INTO loans (customer_id, amount) VALUES (1234, new_amount);

-- Else abort

COMMIT;
```

Cool, this blocks other transactions even when there aren't any rows in the `loans` table yet.

But there's still a problem.

If there is only one concurrent transaction, it will happily acquire the lock when the first transaction is done.

However, if there is more than one waiting transaction, a deadlock occurs and one transaction acquires the lock while the other receive an error.

What gives? Why don't the other transactions continue waiting for the lock?

It seems that at some point both the waiting transactions acquire shared locks. When the first transaction finishes, neither waiting transaction can upgrade to an exclusive lock because both hold shared locks. (this understanding is a bit fuzzy).

Checking the locks held while there are two waiting transactions:
```
mysql> select * from performance_schema.data_locks;
+----------------|-----------|--------------|------------|-----------|---------------|-------------|-----------+
| ENGINE_LOCK_ID | THREAD_ID | OBJECT_NAME  | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+----------------|-----------|--------------|------------|-----------|---------------|-------------|-----------+
| 2201:1100      |        64 | loan_mutexes | NULL       | TABLE     | IX            | GRANTED     | NULL      |
| 2201:43:4:2    |        64 | loan_mutexes | PRIMARY    | RECORD    | S             | WAITING     | 1         |
| 2200:1100      |        62 | loan_mutexes | NULL       | TABLE     | IX            | GRANTED     | NULL      |
| 2200:43:4:2    |        62 | loan_mutexes | PRIMARY    | RECORD    | S             | WAITING     | 1         |
| 2199:1100      |        60 | loan_mutexes | NULL       | TABLE     | IX            | GRANTED     | NULL      |
| 2199:43:4:2    |        62 | loan_mutexes | PRIMARY    | RECORD    | X,REC_NOT_GAP | GRANTED     | 1         |
+----------------|-----------|--------------|------------|-----------|---------------|-------------|-----------+
6 rows in set (0.00 sec)
```

And we can run `show engine innodb status;` to see information about the deadlock once it occurs.
```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2019-01-24 00:09:43 0x70000f674000
*** (1) TRANSACTION:
TRANSACTION 2200, ACTIVE 898 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1136, 3 row lock(s)
MySQL thread id 23, OS thread handle 123145561341952, query id 1765 localhost root update
insert into loan_mutexes (customer_id) values (1)
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 43 page no 4 n bits 72 index PRIMARY of table `test2`.`loan_mutexes` trx id 2200 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) TRANSACTION:
TRANSACTION 2201, ACTIVE 891 sec inserting
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1136, 3 row lock(s)
MySQL thread id 25, OS thread handle 123145560735744, query id 1766 localhost root update
insert into loan_mutexes (customer_id) values (1)
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 43 page no 4 n bits 72 index PRIMARY of table `test2`.`loan_mutexes` trx id 2201 lock mode S
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 43 page no 4 n bits 72 index PRIMARY of table `test2`.`loan_mutexes` trx id 2201 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** WE ROLL BACK TRANSACTION (2)
```

This deadlock problem doesn't occur if there's a existing mutex row that the concurrent transactions are all updating. (It's a technique discussed in the [manual](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks-handling.html))

