---
title: As Rails developers, why we are excited about PostgreSQL 17
description: Why we are excited about PostgreSQL 17 for our Rails applications.
date: 2024-09-19
tags:
  - PostgreSQL
  - Performance
  - Rails
---

At the time of writing this article, PostgreSQL 17 is nearly out. [On September 5th, the first release candidate was published](https://www.postgresql.org/about/news/postgresql-17-rc1-released-2926/). The final release is expected on September 26th, but we can already explain why we’ve been eagerly awaiting this release since 1 year.

At [Lifen](https://www.lifen.fr/), we’ve loved Rails from the beginning. We have several Rails applications, each with different scopes and teams, all using PostgreSQL as the main database. Some of these applications handle a significant amount of traffic, and their databases need to be properly monitored. This is done by the infrastructure team and the developers themselves using PgAnalyze, Grafana and sometimes AWS console with "Performance Insight".

More than a year ago, we started monitoring the 95th and 99th percentile response times (p95, p99) on an endpoint that was experiencing timeouts. These p99 times were important because they involved a client with significantly more data. While investigating the queries in AppSignal, we quickly found that the issue was on the database side, as is often the case.

The problematic query involved one table with over 40 GB of data, which was heavily distributed. The query in Rails looked like this:

```ruby
Doc.where(status: ['draft','sent'], sender_reference: ['Client/1', 'Client/2']).order(sent_at: :desc).limit(20)
```

Pretty simple, right?

Due to the nature of ActiveRecord DSL, this is something you see often, but without fully understanding the limitations of the underlying database. And those limitations can be surprising.

When fetching many documents, the query was very slow. We looked at the EXPLAIN plans and quickly identified the issue: **not all indexes were being used.**

You see plan like this one

```markdown
  ->  Index Scan using docs_sent_at_idx on public.docs  (cost=0.56..39748945.58 rows=29744 width=38) (actual time=116.924..218.784 rows=20 loops=1)
        Output: id, type, status, sender_reference, sent_at, created_at
        Filter: (((docs.status)::text = ANY ('{draft,sent}'::text[])) AND ((docs.sender_reference)::text = ANY ('{Client/1,Client/2}'::text[])))
        Rows Removed by Filter: 46354
        Buffers: shared hit=1421 read=45046
```

Index is used but you still need to perform expensive filtering.

After some exchanges with some great folks on the PostgreSQL Slack, it was mentioned that **PostgreSQL is not optimized for queries with multiple IN or ANY query parameters**. This was a problem, as such queries are common in Rails applications and aren’t easily optimized using indexes in PostgreSQL. But then, something unexpected happened…

I received a private message from [Peter Geoghegan](https://github.com/petergeoghegan). He was working on a patch that could help solve my issue. I tested the first version, then a second, and the results were very impressive, with a 90% reduction in query time (from 110ms to 10ms for some data). Indexes were fully utilized.

The patch is well explained [in a blog post on PgAnalyze](https://pganalyze.com/blog/5mins-postgres-17-faster-btree-index-scans) but the changelog entry speaks for itself: “Allow B-tree indexes to more efficiently find a set of values, such as those supplied by IN clauses using constants.”

Many iterations were made on this patch. I provided feedback on the PostgreSQL hackers [mailing list](https://www.postgresql.org/message-id/flat/CAHUgstCm94QCN3hrLgU9SkVxhLiuh2G_HsksffZwZWHhzJEreg%40mail.gmail.com#f3af55f2367f6477256fcea40ce586a8), explaining why I, as a developer, was interested in this patch, along with benchmarks and execution plan outputs. The patch was included in PostgreSQL 17 beta, and additional fixes were added.

With RC1 and a basic Rails application, here’s the improvement we saw:
We inserted 100 million rows into a table with three columns. This was done via SQL, but it could have been done in Rails as well.

```sql
INSERT INTO
  docs (status, sender_reference, sent_at, created_at, updated_at)
SELECT
  ('{sent,draft,suspended}'::text[])[ceil(random()*3)],
  ('{Custom,Client}'::text[])[ceil(random()*2)] || '/' || floor(random() * 2000),
  (LOCALTIMESTAMP - interval '2 years' * random())::timestamptz,
  LOCALTIMESTAMP,
  LOCALTIMESTAMP
FROM generate_series(1, 100_000_000) g;
```

Those columns are indexed with a multicolumn index.

```ruby
add_index :docs, [:sender_reference, :status, :sent_at], algorithm: :concurrently
```

With PostgreSQL 16 on a Rails 7.2 app, after several queries and with a warm cache, the response time was:

> Query executed in 8.45 seconds.

With PostgreSQL 17, the result is 🥁:

> Query executed in 0.11 seconds.

The query can still be optimized to get closer to 10ms, but the most challenging part to optimize has already been addressed by the patch.

With results like these, I highly encourage you to upgrade to PostgreSQL 17. There are many [other improvements](https://www.postgresql.org/docs/17/release-17.html) as well. I also recommend digging deeper into the database side of your application, as this is often the key to faster response times. I’ve learned that developers are welcome to share their feedback on the PostgreSQL hackers’ mailing list, providing benchmarks and feedback as end users.

Thanks to Peter Geoghegan, Matthias van de Meent, Hubert Depesz, Ants Aasma, and the rest of the PostgreSQL community for their work over the past year.

If you’d like to follow the benchmarks made since last year, there is a [gist](https://gist.github.com/benoittgt/ab72dc4cfedea2a0c6a5ee809d16e04d). You can also find the Rails app used to test response times [on Github](https://github.com/benoittgt/thanks-peter).
