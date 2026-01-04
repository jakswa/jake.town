---
layout: post
title: "Avoid Costly Rails Logging"
slug: costly-logging
description: "Efficiency gains with Rails.logger block syntax."
---

Let's be clear: This post is *not* targeting logger lines like this, with short/quick strings:

```ruby
Rails.logger.debug 'Rare Event!'
```

## Some Logs Can Be Costly

But every now and then you might stumble upon a more-intense logger line. Maybe you are serializing objects for debugging in your logs, or doing IO to load DB rows for debugging purposes. To simulate this, I will go to an extreme and just use `sleep`:

```ruby
Rails.logger.level = :info # simulating production

Benchmark.measure do
  Rails.logger.debug "Log: #{sleep 1}"
end.real

# 1 second passes...
=> 1.0011528139948496
```

This log does not print because it is `.debug` and our `logger.level` is `:info`. But we still had to endure the string-building logic.

## Possible Solution: Use Block Syntax

This syntax passes a block, which is only called if the logging actually runs.

```ruby
Rails.logger.level = :info # simulating production

Benchmark.measure do
  Rails.logger.debug { "Log: #{sleep 1}" }
end.real

# instantly returns a way smaller value
=> 1.4374993043020368e-05
```

If your logging level is `:debug` (usually in development), you will see the time go back up because the logging is actually happening:

```ruby
Rails.logger.level = :debug # simulating development

Benchmark.measure do
  Rails.logger.debug { "Log: #{sleep 1}" }
end.real

# back up to 1s, we are in debug and want our log
=> 1.0006488429935416
```

## Code Coverage Concern

One downside to this path is code-coverage, especially if you suppress `:debug`-level logging in your build environment. Unless your block is executed on a real case, you might not know for sure that it works. Maybe you have a typo or `nil.method` error in your log line and you will only find it in whatever special environment wants to actually log your string.

In my head you can make that more visible by making sure to add a newline on your block, so that automated code coverage tools will call you out. This way, the line will be red unless your build pipeline actually built you a string successfully:

```ruby
Rails.logger.debug do
  "Log: #{sleep 1}"
end
```
