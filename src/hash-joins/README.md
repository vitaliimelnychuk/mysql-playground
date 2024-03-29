## Understanding Hash Joins in MySQL

[Original article](https://www.percona.com/blog/2019/10/30/understanding-hash-joins-in-mysql-8/)

Docs

- https://dev.mysql.com/worklog/task/?id=2241
- https://dev.mysql.com/doc/refman/8.0/en/nested-loop-joins.html#block-nested-loop-join-algorithm
- https://dev.mysql.com/doc/refman/8.0/en/hash-joins.html

## Main concepts

Hash Join Limitations:

- it only works on columns that do not have indexes (or you have to ignore them).
- It only works with equi-join conditions.
- It does not work with LEFT or RIGHT JOIN.

Key points of implementation:

- uses `std::unordered_multimap` as the hash table implementation
- uses `xxHash64` as hashing function, ref(https://github.com/rurban/smhasher)
- There is an ability to use `on-disk hash join
- new `BufferRow`, `HashJoinIterator` and `HashJoinRowBuffer` interfaces,
- Classic hash join scanning both inputs only once but it requires that the entire build input fits into memory.
- on-disk hash join works with several smaller files on disk
-  

## Running

- Run everything from `./init.sql` inside mysql container
- Generate random data, run following from root folder

```bash
    make shell
    python ./src/hash-joins/insert_random_data.py
```

## Examples

### Hash Joins

```
select count(*) from t1 join t2 on t1.c2 = t2.c2;
+----------+
| count(*) |
+----------+
| 39961411 |
+----------+
1 row in set (1.79 sec)
```

### Non-Hash Joins

```
mysql> select /*+ NO_HASH_JOIN (t1,t2) */ count(*) from t1 join t2 on t1.c2 = t2.c2;
+----------+
| count(*) |
+----------+
| 39961411 |
+----------+
1 row in set (41 min 52.24 sec)
```

### Joins Based on Indexes

```
mysql> select count(*) from t1_idx join t2_idx on t1_idx.c2 = t2_idx.c2;
+----------+
| count(*) |
+----------+
| 39961411 |
+----------+
1 row in set (6.89 sec)

```

### Ignore indexes and use hash joins

```
mysql> explain format=tree select count(*) from t1_idx ignore index (idx_c2) join t2_idx ignore index (idx_c2) on t1_idx.c2 = t2_idx.c2 where t1_idx.c2=t2_idx.c2\G

*************************** 1. row ***************************
EXPLAIN: -> Aggregate: count(0)
    -> Inner hash join (t2_idx.c2 = t1_idx.c2)  (cost=4007864388.16 rows=200196)
        -> Table scan on t2_idx  (cost=0.00 rows=200196)
        -> Hash
            -> Table scan on t1_idx  (cost=20123.85 rows=200196)

```

Query run:

```
mysql>  select count(*) from t1_idx ignore index (idx_c2) join t2_idx ignore index (idx_c2) on t1_idx.c2 = t2_idx.c2 where t1_idx.c2=t2_idx.c2;
+----------+
| count(*) |
+----------+
| 39961411 |
+----------+
1 row in set (1.73 sec)
```

## Left join


```
mysql> select count(*) from t1_idx left join t2_idx on t1_idx.c2 = t2_idx.c2;
+----------+
| count(*) |
+----------+
| 39961411 |
+----------+
1 row in set (6.70 sec)

```


```
select count(*) from t1 left join t2 on t1.c2 = t2.c2;
+----------+
| count(*) |
+----------+
| 39961411 |
+----------+
1 row in set (40 min 54.28 sec)

```
