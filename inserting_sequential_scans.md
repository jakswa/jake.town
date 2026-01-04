---
title: INSERTing Sequential Scans
date: Sep 21, 2022
published: true
description: Showing one way `pg_stats` can ruin your query performance with sequential scans.
tags: postgresql, postgres, pg, database, heuristics
# cover_image: https://direct_url_to_image.jpg
# Use a ratio of 100:42 for best results.
---

Do you see "sequential scan" in your `EXPLAIN` output while hoping for index scans? This post dives into one reason why, and aims to be a quick tour, showing SQL you can run on your local `psql`.

If you're a good candidate for this blog post, you might know that `ANALYZE` is a vital feature of PostgreSQL, but maybe you cannot say why, or are unsure of a case where it matters.

We'll go about this in steps:
1. Set up the schema + index up front
2. Test out fast queries on data
3. Create a slowdown
4. See the `pg_stats`



## 1. The setup

This is the only section where we're changing the table schema. This is important: The planner is going to ruin our day, not a schema change.

```sql
create table dogs_seen (id serial, num_seen bigint);
create index dogs_idx on dogs_seen (num_seen);

-- inserts 1mil rows with num_seen values
-- that are weghted towards low, positive integers
insert into dogs_seen (num_seen) select (1/random())
  from generate_series(1,1000000);

-- update pg_stats to reflect our table state
analyze dogs_seen;
```


## 2. Initial testing

The test query we will focus on is this:
```sql
select count(*) from dogs_seen where num_seen = 1;
-- this should be around 300k-400k
```

You expect that query to use the index we created during setup, and it should be:
```sql
explain analyze select count(*) from dogs_seen where num_seen = 1;

         QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=6320.28..6320.29 rows=1 width=8) (actual time=25.344..26.535 rows=1 loops=1)
   ->  Gather  (cost=6320.07..6320.28 rows=2 width=8) (actual time=25.296..26.527 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=5320.07..5320.08 rows=1 width=8) (actual time=19.418..19.418 rows=1 loops=3)
               ->  Parallel Index Only Scan using dogs_idx on dogs_seen  (cost=0.42..4973.54 rows=138611 width=0) (actual time=0.062..11.770 rows=111153 loops=3)
                     Index Cond: (num_seen = 1)
                     Heap Fetches: 0
 Planning Time: 0.292 ms
 Execution Time: 26.595 ms
(10 rows)
```

Yup, it shows `Index` scanning. One thing to note here is that this query has pretty bad performance, relatively speaking. During setup, I mentioned that we generated `num_seen` values _weighted towards one_, so there are way more rows matching our condition than, say, this one:

```sql
explain analyze select count(*) from dogs_seen
  where num_seen = 6;
                                                               QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=650.42..650.43 rows=1 width=8) (actual time=5.709..5.710 rows=1 loops=1)
   ->  Index Only Scan using dogs_idx on dogs_seen  (cost=0.42..580.67 rows=27900 width=0) (actual time=0.040..3.502 rows=28235 loops=1)
         Index Cond: (num_seen = 6)
         Heap Fetches: 0
 Planning Time: 0.233 ms
 Execution Time: 5.754 ms
(6 rows)
```

I draw a comparison here as a hint to say: This query is about as bad as it can get for the planner. It wants to use the index to reduce the amount of data it is scanning, but we just made it scan hundreds of thousands of entries.

What happens if we make things even worse on the planner? What if we made an even larger chunk of this table match our query?


## 3. Create a slowdown

Again, we are going to create a slowdown purely by inserting more rows, but we want to make things worse on our planner, so we'll weight these rows differently:

```sql
-- insert 1mil more rows, even more weight on low values
-- (0.5 instead of 1, is the difference)
insert into dogs_seen (num_seen) select (0.5/random()) from generate_series(1,1000000);

-- don't forget to analyze
analyze dogs_seen;
```

And now explain that original 25ms SQL:
```sql
explain analyze select count(*) from dogs_seen where num_seen = 1;

                                                                 QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=23271.91..23271.92 rows=1 width=8) (actual time=76.779..77.476 rows=1 loops=1)
   ->  Gather  (cost=23271.69..23271.90 rows=2 width=8) (actual time=76.730..77.471 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=22271.69..22271.70 rows=1 width=8) (actual time=70.811..70.811 rows=1 loops=3)
               ->  Parallel Seq Scan on dogs_seen  (cost=0.00..21227.67 rows=417611 width=0) (actual time=0.024..57.560 rows=333558 loops=3)
                     Filter: (num_seen = 1)
                     Rows Removed by Filter: 333109
 Planning Time: 0.599 ms
 Execution Time: 77.524 ms
(10 rows)
```

The same SQL changed to sequential scan! There is dialog in my head here that I imagine the PG planner saying:
- We made our index less valuable for this SQL. We inserted a ton of rows with lots of duplicate values, which just happened to match our SQL condition.
- If postgres thinks it needs to scan a lot of a table, it might skip index-scanning work and scan the table directly instead.

But how does postgres decide the above?


## 4. The `pg_stats`

When we ran `ANALYZE dogs_seen` above, postgresql updated our `pg_stats` rows for this table. These columns are all useful to know and feel free to [read up on them](https://www.postgresql.org/docs/current/view-pg-stats.html). The ones we care about here are `most_common_vals` and its companion `most_common_freqs`:

```sql
select left(most_common_vals::text, 10) as most_common_vals,
  left(most_common_freqs::text, 50) as most_common_freqs
from pg_stats
where tablename = 'dogs_seen' and attname = 'num_seen';

 most_common_vals |                 most_common_freqs
------------------+----------------------------------------------------
 {1,2,3,4,5       | {0.5011333,0.19793333,0.08653333,0.047066666,0.029
(1 row)
```

This reads as:
- the value `1` is seen in 50% of sampled rows
- the value `2` is seen in 20% of sampled rows
- and so on

By inserting lots of rows with the same values, we affected the entries in this list when `ANALYZE` populates it. And the planner adjusted to sequentially scan for the value `1` instead of using the index.

Just to drive this home, let's delete a bunch of `1`s and look at things again:

```sql
-- delete 500k rows, make "1" great again
delete from dogs_seen where id in (select id from dogs_seen where num_seen = 1 limit 500000);

-- re-analyze
analyze dogs_seen;

-- check out the new stats
select left(most_common_vals::text, 10) as most_common_vals, left(most_common_freqs::text, 50) as most_common_freqs from pg_stats where tablename = 'dogs_seen' and attname = 'num_seen';

 most_common_vals |                 most_common_freqs
------------------+----------------------------------------------------
 {1,2,3,4,5       | {0.32923332,0.27103335,0.11643333,0.0607,0.0405666
(1 row)

-- did we switch it back onto the index?
explain analyze select count(*) from dogs_seen where num_seen = 1;
                                                                            QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=9520.65..9520.66 rows=1 width=8) (actual time=33.478..34.535 rows=1 loops=1)
   ->  Gather  (cost=9520.44..9520.65 rows=2 width=8) (actual time=33.384..34.527 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=8520.44..8520.45 rows=1 width=8) (actual time=28.603..28.603 rows=1 loops=3)
               ->  Parallel Index Only Scan using dogs_idx on dogs_seen  (cost=0.43..8006.01 rows=205771 width=0) (actual time=0.265..17.126 rows=166891 loops=3)
                     Index Cond: (num_seen = 1)
                     Heap Fetches: 0
 Planning Time: 0.417 ms
 Execution Time: 34.570 ms
(10 rows)
```

It's back to `Index` scanning! There you have it. We inserted rows and changed the query plan just by running `ANALYZE`. If you have a sequential scan haunting you, maybe it is due to this!
