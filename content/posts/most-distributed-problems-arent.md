+++
date = '2025-08-24T09:00:00+08:00'
draft = false
tags = ["engineering", "distributed-systems", "architecture"]
title = "Most Distributed Problems Aren't"
summary = "One service needed a consumer with ordering guarantees. Another just needed an in-memory cache. The work is knowing which one you're looking at."
+++

I got pulled into a design review for a neighbor team. They believed wrapping external calls in a database transaction made them atomic.

A few months later, I inherited a service from that team. Same belief, baked into the code.

#### The Inheritance

The service was an internal gRPC service for suspending and unsuspending users. End-user traffic was too high for the database to serve reads directly, so user state was served through a separate read-only service backed by Redis.

The service layer would start a db transaction, update the database, update Redis, call another team's service to notify the state change, then commit. On failure, rollback. But the commit only guarantees the database. Redis can fail independently. The downstream call can fail.

The database powered internal dashboards. Redis served end users. From time to time, another team would reach out because the states didn't match. Service restarts mid-operation, race conditions, network issues. Every layer of rollback logic you add has the same failure modes as the original problem.

#### The Actual Fix

Suspension needs to take effect immediately, so we couldn't just accept stale state while the other writes caught up.

We moved the writes out of the gRPC handler entirely. On a suspend or unsuspend request, it just publishes an event. A consumer picks it up and does the actual work: DB write, Redis update, downstream service call. Retries on failure until all three succeed.

That required a few things. Partition ordering, so suspend and unsuspend events for the same user are processed in the order they were sent. Idempotent DB updates so retries don't corrupt state. Confirming the external service is safe to call with the same payload multiple times.

It works. But it's not free. The right cost for suspension. Not always the right cost.

#### Simplify First

Not every multi-write path needs a consumer.

When we looked at the suspend service, the first question was whether we could eliminate the second store entirely. We couldn't. Too much read traffic for the database alone. The next was whether partial failure was actually a crisis. For suspension, it was. Users needed to see the state change immediately, and diverged state between Redis and Postgres meant support tickets. A reconciliation job wouldn't cut it.

But we applied the same questions to other services with the same two-store pattern and got different answers.

#### The Settings Cache

We had application settings stored in Postgres. Read-heavy, roughly 100k rpm. The first instinct was a Redis cache in front of it.

But now the update path writes to both the database and Redis. Fix it with a consumer? A cron job? That's another service to maintain for a setting that changes at most twice a day.

We stepped back. The data was small. Fits in memory. Changes rarely. But when it does change, it sometimes needs to take effect quickly.

The solution was an in-memory cache in the reader service. On startup, pull from Postgres. Every hour, compare `last_updated` and only pull if the data is newer. For the rare case where a change needed to take effect immediately, we restart the reader service. It boots in seconds.

There are still two data stores: Postgres and in-memory. But the tradeoff changed. The database is the source of truth. The in-memory cache is a copy that self-heals. The failure mode is stale data, bounded by the refresh interval. Not silently diverged state where Redis says one thing and Postgres says another.

No Redis. No consumer. No distributed consistency pattern. The problem didn't need one.

#### The Takeaway

Wrapping external calls in a database transaction doesn't give you atomicity. Sometimes the fix is a consumer with ordering and idempotency guarantees. Sometimes you can just remove the write. The work is knowing which one you're looking at.
