---
title: Measuring SELECT ... FOR UPDATE Latency in PostgreSQL
description: Are queries slow because they’re waiting to acquire locks, or because they’re taking a long time to find the actual rows?
date: 2025-07-28
tags:
  - PostgreSQL
  - Performance
  - Rails
---

At [Lifen](https://www.lifen.fr/), we love digging into database performance issues. Recently, while monitoring one of our Rails applications using PgAnalyze and specific logging, we noticed something concerning: around **13–16% of a basic but frequently run query** was taking more than 100ms to complete… and sometimes up to 1500ms. The culprit? `SELECT ... FOR UPDATE` queries that were experiencing intermittent slowdowns. The average time was normally 6ms.

### Slow lock **acquisition** ?

`SELECT ... FOR UPDATE` [is used to acquire a lock](https://www.postgresql.org/docs/current/sql-select.html) on row during a transaction. We use it to prevent concurrent async tasks from modifying the same row. This can happen, for example, when consuming lots of events from a queue system with an “at least once” delivery guarantee.

The obvious question was: *Are these queries slow because they’re waiting to acquire locks, or because they’re taking a long time to find the actual rows?*

This distinction matters a lot. If it’s **lock contention**, we need to optimize our locking strategy. If it’s **row lookup performance**, we need better indexes or query optimizations.

PostgreSQL's built-in `log_lock_waits` parameter seemed like the natural solution, but it requires setting a low `deadlock_timeout`, which makes it impractical for production environments. We needed a different approach.

From the doc on `log_lock_waits`

> Controls whether a log message is produced when a session waits longer than [**deadlock_timeout**](https://www.postgresql.org/docs/17/runtime-config-locks.html#GUC-DEADLOCK-TIMEOUT) to acquire a lock.
>

### Measuring

To distinguish between lock acquisition time and row lookup time, we replaced the `FOR UPDATE` with a `pg_advisory_xact_lock`, then monitored both parts of the query separately.

The SQL transformation looked like this:

```sql
-- Original
BEGIN;
SELECT id FROM steps WHERE id = 123_456 FOR UPDATE; -- this is sometime slow
UPDATE status = 'in_progress' FROM steps WHERE id = 123_456;
COMMIT;

-- Temporary new version
BEGIN;
SELECT pg_advisory_xact_lock(hashtext('update step status'), hashtext('123456');
UPDATE steps SET status = 'in_progress' WHERE id = 123_456;
COMMIT;
```

And in the Rails app:

```ruby
def process_with_xact_lock(step_id)
  start_time = Time.current
  hash_key = "step:#{step_id}".hash.abs

  ActiveRecord::Base.transaction do
    connection.select_all("SELECT pg_advisory_xact_lock(#{connection.quote(hash_key)})")
    lock_duration = Time.current - start_time

    query_start = Time.current
    step = Step.find(step_id)
    query_duration = Time.current - query_start

    step.update!(status: 'in_progress')

    log_timing(step_id, lock_duration, query_duration) if slow_operation?(lock_duration, query_duration)

    # ...
  end
end

private

def slow_operation?(lock_time, query_time)
  (lock_time + query_time) > 0.1 # 100ms threshold
end

def log_timing(id, lock_ms, query_ms)
  Rails.logger.warn(
    "Slow operation: id=#{id} lock=#{(lock_ms*1000).round}ms query=#{(query_ms*1000).round}ms"
  )
end

def connection = ActiveRecord::Base.connection

```

The code is not bulletproof, but we only used it for a short period to gather data. We knew the chance of lock issues was low, and it was acceptable for this part of the application.

### **What we learned**

The results were interesting: when we experienced slowdowns with `SELECT ... FOR UPDATE`, we **also** saw slowness with the new `pg_advisory_xact_lock` version both on **lock acquisition** and **row lookup**.

So, **latency is global**, not specific to one part of the mechanism. That means we needed to look into other database activity that could be causing this broader slowdown.

In our case, it appeared related to heavy DML activity and [IO:WALWrite](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/wait-event.iowalwrite.html). To mitigate this, we’re now looking at reducing the number of INSERT and UPDATE queries by grouping them.

### Key Takeaways

1. **Separate concerns when debugging**: Don't assume slow `SELECT ... FOR UPDATE` means lock contention. Measure lock acquisition and query execution independently.
2. **Production-friendly monitoring**: This technique works in production without requiring changes to PostgreSQL configuration like `deadlock_timeout`. We hope that it will be possible to let PostgreSQL report this by itself (see [discussion on pg-hackers](https://www.postgresql.org/message-id/flat/b8b8502915e50f44deb111bc0b43a99e2733e117.camel%40cybertec.at))
3. pg_advisory_xact_lock is a great tool for precise, manual locking.

If you’re experiencing similar issues with SELECT ... FOR UPDATE, first check your logs, database metrics, and DML activity. Then try this measurement approach, you might discover, like we did, that the real bottleneck is elsewhere.

---

*Thanks to the PostgreSQL community on Slack for the debugging suggestions that led to this technique.*