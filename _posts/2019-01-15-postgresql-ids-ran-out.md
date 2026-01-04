---
layout: post
title: "The Night the PostgreSQL IDs Ran Out"
slug: postgresql-ids-ran-out
---

It's a story no one wants to encounter: Your primary key column runs out of values and your database starts exploding. Alarms go off as people start poking their heads into developer chat:

```
10:50pm [me] we haven't created a new row since 10:30
10:50pm [me] the last ID was 2147483647
```

| Type    | Size    | Range                           |
|---------|---------|----------------------------------|
| integer | 4 bytes | -2147483648 to +2147483647       |

No one should run into this. Everyone should see it coming a mile away (maybe as far back as the starting line), and slap a bigint/uuid type into that primary key column. But shit happens.

Any hope to quickly migrate the column fades with the size of the table. This table has 2+ billion rows, and it's going to take time to rewrite. The `ALTER TABLE` is run out of desperation and the wait begins. The realization settles in that it's going to take more downtime than anyone is comfortable with. What can be done? You can change application code, and deploy something to work around the problem. How long will that take?

One desperate idea stuck out: The int column has a minimum value of -2147483648, and all those negative values are unused. A postgres sequence controls the generation of each ID value.

```
2:11am [me] Oo I wonder if we can set sequence to min, negative value
```

You can run `psql` and `\d <table>` to see the sequence name. Turns out you can reset the sequence to a negative value. This saved us from prolonged downtime and gave us another couple billion IDs, enough time to work out a better solution later:

```sql
-- after running \d <table> and finding your sequence name
alter sequence <sequence_name> minvalue -2147483648;
select setval('<sequence_name>', -2147483647);
```

I have yet to find this recommended anywhere, probably because the situation is rare, and because this change ruins queries that `ORDER BY id` (since the new rows are suddenly negative). Given time to think, we eventually moved to bigint by following these steps for close-to-zero downtime:

1. Create new table with bigint, and same schema otherwise.
2. Create function to copy data into new table
3. Create trigger on update / insert for old table that calls the copy function
4. Back-fill the old data
5. Rename the table

I've seen similar ideas that rename a column instead of a table. The big takeaway is to check in on tables every now and then, to see if any are getting close to their max values. You might not end up as lucky as us!

*"Well, this happened at night at least. Not during peak."*
