---
layout: post
title: "Battling RecordNotUnique in Rails"
slug: battling-uniqueness
description: "A quick tour of the unique-constraint battlefield in Rails."
---

This aims to be a quick post that:
- explains `RecordNotUnique` and my old, ugly patterns
- explains a newer (~2018) built-in rails helper
- explains a custom helper

## What is `RecordNotUnique`?

Typically you attempt to `SELECT` the row, and then `INSERT` under the assumption/hope that it's rare to worry about racing `INSERT`s. In rails there have been longstanding helpers to reach for here:

```ruby
Model.where(unique_col: val)
  .first_or_create { |new_record| ... }
```

This does a `SELECT` and then `INSERT`. This code might raise `RecordNotUnique`, and it'd mean you are `INSERT`ing a record that violates a unique constraint in your database. You're hoping that it's rare to have multiple pieces of code doing this at the same time, for the same rows. In my experience, this hope falls apart more often than I'd like:

```shell
ActiveRecord::RecordNotUnique: PG::UniqueViolation: ERROR:  duplicate key value violates unique constraint "index_models_on_unique_col"
DETAIL:  Key (unique_col)=(val) already exists.
```

Take my recent case as an example. I'm in SOA land writing rails code. The latest app is processing events from multiple sources that occur very closely together. From the start I was writing code to rescue `RecordNotUnique`:

```ruby
begin
  Model.where(...).first_or_create { ... }
rescue ActiveRecord::RecordNotUnique => exception
  # Retry first_or_create here.
  # Blow up if still not found (broken code).
end
```

This gets ugly pretty quickly. Any time you retry, you make sure you're not creating an infinite loop. And you re-raise the exception if the 1st retry fails. The rescue code might look like this:

```ruby
begin
  tries ||= 0
  Model.where(...).first_or_create { ... }
rescue ActiveRecord::RecordNotUnique => exception
  retry if (tries += 1) == 1
  raise exception
end
```

That is how I used to *hope* to tackle `RecordNotUnique`. It was rare and handled on a case-by-case basis. You end up writing complicated tests for the above that:
- test each line in your rescue
- try to test actual racing inserts if it's important (integration specs, I typically need more code/hooks to help there)

## Enter `create_or_find_by`

`first_or_create` will `SELECT` and then `INSERT`, but let's say you don't want that. Let's say you rarely find anything in your `SELECT` and almost always `INSERT`. An example of this is a dedicated, high-volume ruby process whose job is to process just *new* records. If it's always doing the `INSERT`, then you are wasting the up-front `SELECT`.

If you can infer this up front, then in DHH's words, you might want to ["lean on unique constraints"](https://github.com/rails/rails/pull/31989):

```ruby
Model.where(unique_attr: val).create_or_find_by do |new_record|
  new_record.assign_attributes(initial_attributes)
end
```

The quick summary here is that rails will do `first_or_create` in reverse. You'll see an `INSERT` go by with your full set of attributes. If a unique constraint gets you, you'll see a `SELECT` go by with just the `unique_attr` condition.

It's handy. I've used it in some cases. And if you can use it, you're leaning on rails to handle `RecordNotUnique`, letting you throw away that manual `retry` code above (and the associated tests).

It's not without [gotchas](https://github.com/rails/rails/issues/35543) though. I personally had problems with `create_or_find_by` as I leaned into it:
1. I wanted to lean harder! I needed to know how the `create_or_find_by` went. Did the record just get inserted? I don't know of a way to tell. If you want to answer this question, suddenly you are going back to `rescue RecordNotUnique` and all that ugliness.
2. I wanted to go back to `SELECT`ing first, and still not worry about `RecordNotUnique`. My code doesn't `INSERT` often, but when it does, it's usually racing to `INSERT` with other code.

## Custom `take_or_create!` Helper

So far I've been using these without any new gotchas compared to `create_or_find_by`. The style is a bit different: The block argument needs to return an attribute hash. It solves my situation so far:

```ruby
# in application_record.rb, or in a concern/mixin

  attr_writer :created_moments_ago

  def created_moments_ago?
    @created_moments_ago
  end

  def self.create_or_take!
    where(block_given? && yield).crumby_create!
  rescue ActiveRecord::RecordNotUnique
    take!
  end

  def self.crumby_create!
    instance = transaction(requires_new: true) { create! }
    instance.created_moments_ago = true
    instance
  end

  def self.take_or_create!(&block)
    take || create_or_take!(&block)
  end
```

Let's do a quick tour of these helpers, and explain how they solve my 1 & 2 above.

```ruby
# INSERT first. If INSERT fails, try to SELECT
Model.where(unique_attr: val)
  .create_or_take! { creation_attrs_hash }

# SELECT first, then INSERT.
# If INSERT fails, try to SELECT one more time.
Model.where(unique_attr: val)
  .take_or_create! { creation_attrs_hash }
```

*Note:* The `creation_attrs_hash` piece diverges from `create_or_find_by`, which passes in a new record to the block.

That gives me a `SELECT`-first path, addressing my #2 problem above. But what about #1? What about knowing how it went? If you use these helpers to create your models, you'll get a breadcrumb to read, that is `true` if it successfully `INSERT`ed the record:

```ruby
model = Model.where(unique: val).take_or_create! { ... }
model.created_moments_ago? # naming is hard!
```

That's it! These are purpose-built. Maybe you have a slightly different scenario and need it to work differently. Maybe one day I'll learn of a new rails helper that removes the need for my custom helper. Hopefully this helps someone else in their battle against `RecordNotUnique`.

---

## 2023 Update: Rails 7.1 to the Rescue

If you're reading this in 2023+, Rails 7.1 adds built-in support for my #2 case above. `find_or_create_by` now retries the find if it hits a uniqueness constraint. From the [Rails 7.1 changelog](https://github.com/rails/rails/blob/914130a9f54fb45fa9d56cfdd0b838e403946e8c/activerecord/CHANGELOG.md?plain=1#L1255-L1271):

> `find_or_create_by` always has been inherently racy, either creating multiple duplicate records or failing with `ActiveRecord::RecordNotUnique` depending on whether a proper unicity constraint was set.
>
> `create_or_find_by` was introduced for this use case, however it's quite wasteful when the record is expected to exist most of the time, as INSERT requires sending more data than SELECT and requires more work from the database. Also on some databases it can actually consume a primary key increment which is undesirable.
>
> So for cases where most of the time the record is expected to exist, `find_or_create_by` can be made race-condition free by re-trying the `find` if the `create` failed with `ActiveRecord::RecordNotUnique`. This assumes that the table has the proper unicity constraints, if not, `find_or_create_by` will still lead to duplicated records.

So if you're on Rails 7.1+, you can likely ditch the custom helpers and just use `find_or_create_by` with confidence.
