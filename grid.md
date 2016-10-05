Testing MySQL transaction isolation levels
==========================================

These tests were run with MySQL 5.6.21.

Setup (before every test case):

```sql
create table test (id int primary key, value int) engine=innodb;
insert into test (id, value) values (1, 10), (2, 20);
```

To see the current isolation level:

```sql
select @@tx_isolation;
```

Read Committed basic requirements (G0, G1a, G1b, G1c)
-----------------------------------------------------

MySQL "read uncommitted" prevents Write Cycles (G0) by locking updated rows:

```sql
set session transaction isolation level read uncommitted; begin; -- T1
set session transaction isolation level read uncommitted; begin; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 12 where id = 1; -- T2, BLOCKS
update test set value = 21 where id = 2; -- T1
commit; -- T1. This unblocks T2
select * from test; -- T1. Shows 1 => 12, 2 => 21 *(Actual behavior 1 => 11, 2 =>21)*
update test set value = 22 where id = 2; -- T2
commit; -- T2
select * from test; -- either. Shows 1 => 12, 2 => 22 *(Correct)*
```

MySQL "read uncommitted" does not prevent Aborted Reads (G1a):

```sql
set session transaction isolation level read uncommitted; begin; -- T1
set session transaction isolation level read uncommitted; begin; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Shows 1 => 101 *(Actual behavior 1 => 10)*
rollback; -- T1
select * from test; -- T2. Shows 1 => 10 again *(Correct)*
commit; -- T2
```

MySQL "read committed" prevents Aborted Reads (G1a):

*(This test matches expected behavior exactly)*

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
update test set value = 101 where id = 1; -- T1  
select * from test; -- T2. Still shows 1 => 10
rollback; -- T1
select * from test; -- T2. Still shows 1 => 10
commit; -- T2
```

MySQL "read uncommitted" does not prevent Intermediate Reads (G1b):

```sql
set session transaction isolation level read uncommitted; begin; -- T1
set session transaction isolation level read uncommitted; begin; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Shows 1 => 101  *(Actual behavior 1 => 10)*
update test set value = 11 where id = 1; -- T1
commit; -- T1
select * from test; -- T2. Now shows 1 => 11 *(Correct)*
commit; -- T2
```

MySQL "read committed" prevents Intermediate Reads (G1b):

*(This test matches expected behavior exactly)*

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
update test set value = 101 where id = 1; -- T1
select * from test; -- T2. Still shows 1 => 10
update test set value = 11 where id = 1; -- T1
commit; -- T1
select * from test; -- T2. Now shows 1 => 11
commit; -- T2
```

MySQL "read uncommitted" does not prevent Circular Information Flow (G1c):

```sql
set session transaction isolation level read uncommitted; begin; -- T1
set session transaction isolation level read uncommitted; begin; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 22 where id = 2; -- T2
select * from test where id = 2; -- T1. Shows 2 => 22  *(Actual behavior 2=> 20)*
select * from test where id = 1; -- T2. Shows 1 => 11  *(Actual behavior 1=> 10)*
commit; -- T1
commit; -- T2
```

MySQL "read committed" prevents Circular Information Flow (G1c):

*(Matches expected behavior exactly)*

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 22 where id = 2; -- T2
select * from test where id = 2; -- T1. Still shows 2 => 20
select * from test where id = 1; -- T2. Still shows 1 => 10
commit; -- T1
commit; -- T2
```


Observed Transaction Vanishes (OTV)
-----------------------------------

MySQL "read uncommitted" does not prevent Observed Transaction Vanishes (OTV):

```sql
set session transaction isolation level read uncommitted; begin; -- T1
set session transaction isolation level read uncommitted; begin; -- T2
set session transaction isolation level read uncommitted; begin; -- T3
update test set value = 11 where id = 1; -- T1
update test set value = 19 where id = 2; -- T1
update test set value = 12 where id = 1; -- T2. BLOCKS
commit; -- T1. This unblocks T2
select * from test; -- T3. Shows 1 => 12, 2 => 19 *(Actual behavior 1=> 11, 2=>19)*
update test set value = 18 where id = 2; -- T2
select * from test; -- T3. Shows 1 => 12, 2 => 18 *(Actual behavior 1=>11, 2=>19)*
commit; -- T2
commit; -- T3  *(If tested after this commit, 1=>12, 2=>18)*
```

MySQL "read committed" prevents Observed Transaction Vanishes (OTV):

*(Exactly as expected)*

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
set session transaction isolation level read committed; begin; -- T3
update test set value = 11 where id = 1; -- T1
update test set value = 19 where id = 2; -- T1
update test set value = 12 where id = 1; -- T2. BLOCKS
commit; -- T1. This unblocks T2
select * from test; -- T3. Shows 1 => 11, 2 => 19 
update test set value = 18 where id = 2; -- T2
select * from test; -- T3. Shows 1 => 11, 2 => 19
commit; -- T2
select * from test; -- T3. Shows 1 => 12, 2 => 18
commit; -- T3
```


Predicate-Many-Preceders (PMP)
------------------------------

MySQL "read committed" does not prevent Predicate-Many-Preceders (PMP):

*(Should work as expected)*

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Returns the newly inserted row
commit; -- T1
```

MySQL "repeatable read" prevents Predicate-Many-Preceders (PMP) for read predicates:

*(This isn't really doable.  Since each query is a snapshot, the two T1 queries are run independently, and will get different results)*

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where value = 30; -- T1. Returns nothing
insert into test (id, value) values(3, 30); -- T2 
commit; -- T2
select * from test where value % 3 = 0; -- T1. Still returns nothing (returns 3=>30)
commit; -- T1
```

MySQL "read committed" does not prevent Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

*(No delete ... where support, can't run)*

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
update test set value = value + 10; -- T1
select * from test; -- T2. Returns 1 => 10, 2 => 20
delete from test where value = 20;  -- T2, BLOCKS
commit; -- T1. This unblocks T2
select * from test; -- T2. Returns 2 => 30
commit; -- T2
```

MySQL "repeatable read" does not prevent Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

*(No delete ... where support, can't run)*

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
update test set value = value + 10; -- T1
select * from test where value = 20; -- T2. Returns 2 => 20
delete from test where value = 20;  -- T2, BLOCKS
commit; -- T1. This unblocks T2
select * from test; -- T2. Returns 2 => 20, despite rows with value 20 ostensibly being deleted
commit; -- T2
```

MySQL "serializable" prevents Predicate-Many-Preceders (PMP) for write predicates -- example from Postgres documentation:

*(No update ... where support, can't run)*

```sql
set session transaction isolation level serializable; begin; -- T1
set session transaction isolation level serializable; begin; -- T2
select * from test where value = 20; -- T2, returns 2 => 20
update test set value = value + 10; -- T1, BLOCKS
delete from test where value = 20; -- T2, causes T1 to print "ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction"
rollback; -- T1
commit; -- T2
```


Lost Update (P4)
----------------

MySQL "repeatable read" does not prevent Lost Update (P4):

*(No update ... where support, can't run)*

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where id = 1; -- T1
select * from test where id = 1; -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 11 where id = 1; -- T2, BLOCKS
commit; -- T1
commit; -- T2
```

MySQL "serializable" prevents Lost Update (P4):

*(No update ... where support, can't run)*

```sql
set session transaction isolation level serializable; begin; -- T1
set session transaction isolation level serializable; begin; -- T2
select * from test where id = 1; -- T1
select * from test where id = 1; -- T2
update test set value = 11 where id = 1; -- T1, BLOCKS
update test set value = 11 where id = 1; -- T2, prints "ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction"
commit; -- T1
rollback; -- T2
```


Read Skew (G-single)
--------------------

MySQL "read committed" does not prevent Read Skew (G-single):

*(No update ... where support, can't run)*

```sql
set session transaction isolation level read committed; begin; -- T1
set session transaction isolation level read committed; begin; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2
select * from test where id = 2; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 18
commit; -- T1
```

MySQL "repeatable read" prevents Read Skew (G-single) on a read-only transaction:

*(No update ... where support, can't run)*

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test where id = 1; -- T2
select * from test where id = 2; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
select * from test where id = 2; -- T1. Shows 2 => 20
commit; -- T1
```

MySQL "repeatable read" prevents Read Skew (G-single) -- test using predicate dependencies:

*(No update ... where support, can't run)*

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where value % 5 = 0; -- T1
update test set value = 12 where value = 10; -- T2
commit; -- T2
select * from test where value % 3 = 0; -- T1. Returns nothing
commit; -- T1
```

MySQL "repeatable read" does not prevent Read Skew (G-single) on a write predicate:

*(No update ... where support, can't run)*

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test; -- T2
update test set value = 12 where id = 1; -- T2
update test set value = 18 where id = 2; -- T2
commit; -- T2
delete from test where value = 20; -- T1. Doesn't delete anything
select * from test where id = 2;   -- T1. Shows 2 => 20
commit; -- T1
```

MySQL "serializable" prevents Read Skew (G-single) on a write predicate:

*(No update ... where support, can't run)*

```sql
set session transaction isolation level serializable; begin; -- T1
set session transaction isolation level serializable; begin; -- T2
select * from test where id = 1; -- T1. Shows 1 => 10
select * from test; -- T2
update test set value = 12 where id = 1; -- T2. BLOCKS
delete from test where value = 20; -- T1. Prints "ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction"
update test set value = 18 where id = 2; -- T2
rollback; -- T1
commit; -- T2
```


Write Skew (G2-item)
--------------------

MySQL "repeatable read" does not prevent Write Skew (G2-item):

*(No update ... where support, can't run)*

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where id in (1,2); -- T1
select * from test where id in (1,2); -- T2
update test set value = 11 where id = 1; -- T1
update test set value = 21 where id = 2; -- T2
commit; -- T1
commit; -- T2
```

MySQL "serializable" prevents Write Skew (G2-item):

*(No update ... where support, can't run)*

```sql
set session transaction isolation level serializable; begin; -- T1
set session transaction isolation level serializable; begin; -- T2
select * from test where id in (1,2); -- T1
select * from test where id in (1,2); -- T2
update test set value = 11 where id = 1; -- T1, BLOCKS
update test set value = 21 where id = 2; -- T2, prints "ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction"
commit;   -- T1
rollback; -- T2
```


Anti-Dependency Cycles (G2)
---------------------------

MySQL "repeatable read" does not prevent Anti-Dependency Cycles (G2):

*(Queries are independent, not governed by txns, this doesn't really apply)*

```sql
set session transaction isolation level repeatable read; begin; -- T1
set session transaction isolation level repeatable read; begin; -- T2
select * from test where value % 3 = 0; -- T1
select * from test where value % 3 = 0; -- T2
insert into test (id, value) values(3, 30); -- T1
insert into test (id, value) values(4, 42); -- T2
commit; -- T1
commit; -- T2
select * from test where value % 3 = 0; -- Either. Returns 3 => 30, 4 => 42
```

MySQL "serializable" prevents Anti-Dependency Cycles (G2):

*(Queries are independent, not governed by txns, this doesn't really apply)*

```sql
set session transaction isolation level serializable; begin; -- T1
set session transaction isolation level serializable; begin; -- T2
select * from test where value % 3 = 0; -- T1
select * from test where value % 3 = 0; -- T2
insert into test (id, value) values(3, 30); -- T1, BLOCKS
insert into test (id, value) values(4, 42); -- T2, prints "ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction"
commit;   -- T1
rollback; -- T2
```

MySQL "serializable" prevents Anti-Dependency Cycles (G2) -- Fekete et al's example with two anti-dependency edges:

*(Queries are independent, not governed by txns, this doesn't really apply)*

```sql
set session transaction isolation level serializable; begin; -- T1
select * from test; -- T1. Shows 1 => 10, 2 => 20
set session transaction isolation level serializable; begin; -- T2
update test set value = value + 5 where id = 2; -- T2, BLOCKS
set session transaction isolation level serializable; begin; -- T3
select * from test; -- T3, BLOCKS (eventually shows 1 => 10, 2 => 20)
update test set value = 0 where id = 1; -- T1, BLOCKS, causes T2 to abort with deadlock error, unblocks T3
commit; -- T3, unblocks T1
commit; -- T1
rollback; -- T2
```
