---
layout: post
title: "Your First DB Index, by Example"
slug: first-db-index
description: "Going from nothing to index-only scanning, EXPLAINing throughout."
---

I wanted to explore how simply I could illustrate the benefits of an index-only scan, and landed on this quick walk-through. To make it quick, we'll keep things simple. We'll have one SQL query that we `EXPLAIN` in a few different scenarios. It's going to start off at sub-millisecond speed (empty table). We'll add 1 million rows, making our query 1000 times slower. Then we'll add a btree index to speed it back up.

If you want, you can follow along in your own `psql` console. To start things off, I've already ran `create database my_db;` and switched to it.

## Setup + Baseline

First, the empty table will have no indexes, and a single column:

```sql
create table users (email text);
```

Let's focus on a small piece of SQL:

```sql
my_db=# select count(1) from users where email = 'bob@bob.bob';
 count
-------
     0
(1 row)
```

And the `EXPLAIN ANALYZE` for that, which we'll run repeatedly:

```sql
my_db=# EXPLAIN ANALYZE select count(1) from users where email = 'bob@bob.bob';
                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Aggregate  (cost=27.02..27.03 rows=1 width=8) (actual time=0.009..0.009 rows=1 loops=1)
   ->  Seq Scan on users  (cost=0.00..27.00 rows=7 width=0) (actual time=0.005..0.005 rows=0 loops=1)
         Filter: (email = 'bob@bob.bob'::text)
 Planning time: 0.095 ms
 Execution time: 0.058 ms
(5 rows)
```

(the `ANALYZE` in `EXPLAIN ANALYZE` just means it actually performs the query, and provides 'actual' timing)

The table is empty, so it took 58 microseconds for postgres to give us zero rows. This is our starting point.

## Slow it down

Let's slow our query down by adding 1 million rows:

```sql
INSERT INTO users (email)
SELECT 'bob' || x.id::varchar || '@bob.bob'
FROM generate_series(1,1000000) AS x(id);
```

And get our new explain output:

```sql
my_db=# EXPLAIN ANALYZE select count(1) from users where email = 'bob@bob.bob';
                                                 QUERY PLAN
-------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=19628.50..19628.51 rows=1 width=8) (actual time=114.858..114.859 rows=1 loops=1)
   ->  Seq Scan on users  (cost=0.00..19628.50 rows=1 width=0) (actual time=114.854..114.854 rows=0 loops=1)
         Filter: (email = 'bob@bob.bob'::text)
         Rows Removed by Filter: 1000000
 Planning time: 0.154 ms
 Execution time: 114.904 ms
```

It took 114ms for postgres to scan the table, rejecting all rows for our count condition. It even tells us how many rows it removed.

Alright we went from 58 to 114904 microseconds. We slowed it down by a few orders of magnitude. So far these queries are sequentially scanning all rows, and this doesn't scale very well when you have a ton of rows. It scales even worse when your table is a real example, as opposed to this single-column table we're using.

## Adding index

Let's add our index, and then `EXPLAIN` one more time:

```sql
my_db=# CREATE index my_cool_index ON users (email);
my_db=# EXPLAIN ANALYZE select count(1) from users where email = 'bob@bob.bob';
                                                           QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=4.45..4.46 rows=1 width=8) (actual time=0.091..0.091 rows=1 loops=1)
   ->  Index Only Scan using my_cool_index on users  (cost=0.42..4.44 rows=1 width=0) (actual time=0.087..0.087 rows=0 loops=1)
         Index Cond: (email = 'bob@bob.bob'::text)
         Heap Fetches: 0
 Planning time: 0.442 ms
 Execution time: 0.154 ms
```

Woo, back down to sub-millisecond response time. Notice the output no longer includes the 'Seq Scan on users', and instead does 'Index Only Scan using my_cool_index'. Postgres has changed how it performs our query. It's now traversing our btree (balanced [tree](https://en.wikipedia.org/wiki/Tree_(data_structure))) index, scanning it for entries matching our condition.

A query can be index-only when the `SELECT` portion doesn't require you to visit the rows. If we were `SUM`ing a number in a separate column, we'd be leaving 'Index Only Scan' behind:

```sql
my_db=# ALTER TABLE users ADD COLUMN age integer;
my_db=# EXPLAIN ANALYZE select SUM(age) from users where email = 'bob5@bob.bob';
                                                        QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=8.45..8.46 rows=1 width=8) (actual time=0.144..0.144 rows=1 loops=1)
   ->  Index Scan using my_cool_index on users  (cost=0.43..8.45 rows=1 width=4) (actual time=0.134..0.136 rows=1 loops=1)
         Index Cond: (email = 'bob5@bob.bob'::text)
 Planning time: 0.138 ms
 Execution time: 0.202 ms
```

You've made it! We've covered a simple scenario where an index speeds up your SQL. We also inadvertently exposed ourselves to index-only scans, the pinnacle of `COUNT` performance as far as I know.
