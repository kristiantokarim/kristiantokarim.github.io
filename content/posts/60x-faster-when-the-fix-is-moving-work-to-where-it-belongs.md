+++
date = '2019-12-15T10:00:00+08:00'
draft = false
tags = ["engineering", "databases", "performance"]
title = '60x Faster: When the Fix Is Moving Work to Where It Belongs'
summary = "A booking search system was falling over under load. The fix wasn't clever, just moving pagination to the database, replacing a catch-all query pattern, and adding proper indexes. 60-second queries became sub-second."
+++

This is about a booking search system I worked on at a previous company that couldn't keep up with the growing data. It looked like the database was too slow. It wasn't. We were just using it wrong.

#### The Setup

The system powered a booking search tool used by both the ops team and end users. It worked fine early on when the table had tens of thousands of rows. But the table grew steadily, and by the time I looked at it, queries were timing out. Pages took 30-60 seconds to load. Sometimes they just failed.

The query was a catch-all: one query to handle every possible filter combination. It looked something like this:

```sql
SELECT * FROM bookings
WHERE (provided_start_date IS NULL OR start_date > provided_start_date)
  AND (provided_end_date IS NULL OR start_date < provided_end_date)
  AND (provided_booking_id IS NULL OR booking_id = provided_booking_id)
  -- ... and so on for every filter
```

Unused filters passed through as NULL. That meant the query shape changed depending on which filters were provided, so the database couldn't reliably use indexes.

#### Why It Got Worse

Each decision was fine at small scale. None were revisited as the data grew.

A few things made it worse:

- **The catch-all query pattern**. The `IS NULL OR` pattern meant the database couldn't pick the right index, because the effective query changed depending on which filters were filled in.
- **No useful indexes** on the columns being filtered. Most queries ended up as full table scans.
- **Frontend pagination**. The backend returned all matching rows. The frontend handled which page to show.

The database wasn't slow. We were just using it wrong.

#### The Fix

Once the problem was clear, the fix was straightforward:

**1. Split the end-user and ops queries apart.**

End users only ever looked up their own bookings, so we split that into its own simple query by user ID. That was already fast and didn't need the catch-all pattern at all. The ops tool was the one with all the flexible filters, so we focused the rest of the fixes there.

**2. Move pagination to the database and drop the catch-all query.**

Instead of returning all rows and paginating on the frontend, we moved pagination to the database with `LIMIT` and `OFFSET`. We also replaced the catch-all query with queries built from only the filters the ops team actually provided. No more `IS NULL OR` for every column.

```sql
SELECT * FROM bookings
WHERE status = 'CONFIRMED'
  AND created_at >= '2019-01-01'
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;
```

**3. Require date ranges and index for common queries.**

We looked at how the ops team actually used the tool to figure out which filter combinations were common and which indexes would help. We also added constraints on the UI: start date and end date became required fields. This meant every query had a bounded date range, which meant every query hit a bounded range instead of scanning the full table.

**4. Add proper indexes.**

Based on the usage patterns, we added composite indexes on the columns being filtered and sorted:

```sql
CREATE INDEX idx_bookings_status_created ON bookings (status, created_at DESC);
CREATE INDEX idx_bookings_user_id ON bookings (user_id);
```

Full table scans became index lookups.

None of this is new. The hard part was noticing it needed to be done.

#### The Result

Query times went from 30-60 seconds to under 1 second. The ops team went from avoiding the tool to using it every day.

A **60x improvement**. Not from rewriting the system, just from putting work where it belongs.

#### The Lesson

The biggest optimization here wasn't about being clever. It was about putting the work in the right layer. The database was already built to do this work. We just weren't letting it.

When something is slow, before reaching for caching or replicas, ask a simpler question: **is the work happening in the right place?**
