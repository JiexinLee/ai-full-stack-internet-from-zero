# Chapter 6: Databases (Not “SQL Syntax”, but “Why It’s Fast and Correct”) 📚

> The goal of this chapter is not “I can write SELECT”.
> The goal is: when the system is slow, data is wrong, or concurrency rises, you can reason from a high-level viewpoint and know where to look next.
> AI can write SQL for you, but it cannot take responsibility for performance and correctness. That part is still on you. 🙂

This chapter follows the five stages in `outline.md`:

- Stage 1: CRUD / Join / Group / Index
- Stage 2: B+Tree / Hash Index / Clustered Index / Covering Index
- Stage 3: Transactions / ACID / MVCC / Locks / Deadlocks
- Stage 4: Explain / Execution plan / Slow queries / Optimization
- Stage 5: Connection pools / Why pools exist / Why connections cannot be infinite / Leaks / Reuse

---

## The Three Questions

### 1) Why do databases exist? 🤔

Without a database, systems quickly become one of these:

- store in memory: restart = data gone
- store in files: you can store, but querying/updating is painful
- store in random JSON blobs: eventually nobody dares to change anything

Real systems need more than “can store data”. They need:

- durability
- fast querying
- safe concurrency
- correctness guarantees
- recovery and auditing

A database is a “source of truth” system that provides these guarantees.

### 2) Why choose relational databases? Are there alternatives? 🧰

Relational databases (MySQL/PostgreSQL) are great when you need:

- structured data (tables/columns)
- relationships (user–order–product)
- transactions (money, inventory, permissions)
- complex querying (filtering, sorting, aggregates)

Alternatives exist, each with trade-offs:

- key-value stores (Redis): fast, but not ideal as durable truth for complex relational queries
- document stores (MongoDB): flexible schema, different trade-offs for joins/transactions
- OLAP/column stores (ClickHouse): great analytics, not for heavy OLTP transactions
- search engines (Elasticsearch): great search, not a source of truth

Plain English:

> Relational DBs are your ledger. Many other systems are acceleration/search/analytics tools around it.

### 3) How do databases work, and how do we use them in practice? 🔍

Now we open the hood.

---

## A core instinct: databases are often slow because of IO + plans + locks (not “math”)

When a database is slow, common causes are:

- unfriendly IO patterns
- missing or unused indexes
- poor execution plans
- lock contention
- connection misuse

So think of a database as a machine that is extremely good at managing IO.

```text
CPU compute: fast (ns–µs)
RAM access: fast (ns)
Disk access: slow (ms)

Databases try to:
use RAM / read sequentially / avoid random IO
```

---

## Stage 1: CRUD / Join / Group / Index

### 1. CRUD

```sql
INSERT INTO users (id, email) VALUES (1, 'a@b.com');
SELECT id, email FROM users WHERE id = 1;
UPDATE users SET email = 'new@b.com' WHERE id = 1;
DELETE FROM users WHERE id = 1;
```

Useful instincts:

- `WHERE` often determines whether an index can be used
- `SELECT *` is often a production smell (more data, worse chances for covering indexes)

### 2. Join

Given:

```text
users(id, email)
orders(id, user_id, total)
```

Query:

```sql
SELECT u.id, u.email, o.id AS order_id, o.total
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.id = 1;
```

```text
users(1) ----(u.id = o.user_id)----> orders(many)
```

Beginner traps:

- missing join condition -> cartesian explosion
- join columns not indexed -> scans + heavy comparisons

### 3. Group By

```sql
SELECT user_id, COUNT(*) AS cnt
FROM orders
GROUP BY user_id;
```

Instinct:

- `GROUP BY` often involves sorting or hash aggregation, expensive at large scale

### 4. Index

Index is like a book’s table of contents:

```text
No index: flip pages from start to end
Index: use the contents -> jump to the right page
```

Common instinct:

- index columns used in `WHERE` / `JOIN` / `ORDER BY`
- higher selectivity tends to help more

---

## Stage 2: Why indexes are fast (B+Tree / Hash / Clustered / Covering)

### 1. B+Tree

B+Trees fit disk/page storage well:

- low height (often 3–4 levels for huge tables)
- leaf nodes linked in order (range queries are great)
- reduces random IO

```text
            [  50  ]
           /       \
      [10 20 30]  [60 70 80]
        | | |        | | |
        v v v        v v v
  [1..9] <-> [10..19] <-> [20..29] <-> ...
```

Two key values:

- equality lookups are fast
- range lookups are also fast

### 2. Hash index

Great for equality:

```text
key -> hash(key) -> bucket -> record
```

Not good for ranges.

That is why B+Tree is the default hero in many relational engines.

### 3. Clustered index

Some engines store the _table data itself_ ordered by the primary key.

Instinct:

- great for primary-key range scans
- random primary keys (like random UUID) can fragment inserts

```text
Page 1: [id 1..100]
Page 2: [id 101..200]
...
```

### 4. Covering index

If all columns needed by the query are contained in the index, the engine can:

> read only the index and avoid reading the base table rows.

Example idea:

```sql
SELECT user_id, created_at, total
FROM orders
WHERE user_id = 1
ORDER BY created_at DESC
LIMIT 20;
```

Instinct:

- `SELECT *` often kills covering index opportunities

---

## Stage 3: Transactions (ACID / MVCC / Locks / Deadlocks)

### 1. Why transactions exist

Many business actions require “all-or-nothing”, such as:

- create an order
- decrement inventory
- record a payment

Transactions ensure:

- either everything succeeds
- or nothing is committed

### 2. ACID in plain language

- Atomicity: all or nothing
- Consistency: constraints/invariants remain true
- Isolation: concurrency does not create weird reads
- Durability: committed data survives crashes

### 3. Weird things that happen without proper isolation

- dirty reads
- non-repeatable reads
- phantom reads

```text
Time ->

Tx A: read balance=100 -------- read balance=80
Tx B:         update balance=80 and commit
```

### 4. MVCC: why reads can be cheap

MVCC uses multiple row versions:

- writes create new versions
- reads see a snapshot version (depending on isolation level)

```text
Row X:
v1 -> v2 -> v3

Read: snapshot -> picks a version
Write: creates a new version
```

### 5. Locks

MVCC does not replace locks entirely.

Writes and certain isolation guarantees still require locking.

### 6. Deadlocks

Deadlock is:

> A waits for B, B waits for A, nobody moves.

```text
Tx A: locks row1 -> waits for row2
Tx B: locks row2 -> waits for row1
=> deadlock
```

Common practice:

- database detects deadlocks and aborts one transaction
- apps often retry
- consistent lock ordering reduces probability

---

## Stage 4: Explain plans, slow queries, optimization

### 1. Why do we need execution plans?

The same SQL can be executed in different ways:

- scan vs index
- join order
- sort vs no sort

The execution plan is the database’s “solution steps”.

### 2. How to read EXPLAIN (the simple version)

Different DBs print different formats, but the core questions are:

- did it use an index?
- how many rows were scanned?
- did it sort or build temp structures?
- what is the join order?

```text
SQL: fetch one record
Plan A: scan 10M rows
Plan B: use index -> jump to 1 row

EXPLAIN tells you which plan it chose
```

### 3. A practical slow query workflow

1. confirm where time is spent (DB vs app vs network)
2. capture the slow SQL
3. EXPLAIN it and check indexes / rows / sorts

### 4. Common optimization directions (high ROI)

- select only needed columns
- add the right index (WHERE/JOIN/ORDER BY)
- avoid functions/implicit casts on indexed columns
- limit result size (LIMIT)
- avoid N+1 queries
- split complex queries (fetch ids first, then details)

---

## Stage 5: Connection pools (why pools exist, leaks, reuse)

### 1. Why pools exist

Database connections are expensive:

- handshake and authentication
- server resource allocation

Pools let you:

> reuse a limited number of connections to serve many requests, with a hard cap.

### 2. Why connections cannot be infinite

Each connection consumes memory and management overhead.

Too many connections can make the database spend more time managing itself than doing work.

So you need:

- limits
- queues
- timeouts

```text
Request
  |
  v
[Pool]
  | idle conn -> borrow it
  | no conn   -> wait or timeout
  v
[Database]
```

### 3. Connection leaks

A leak is: you borrow a connection and never return it.

Symptoms:

- pool runs out of connections
- everything stalls waiting for a connection

### 4. Pool thinking in Node (pseudo-code)

```js
async function handler(req, res) {
  const client = await pool.acquire();
  try {
    const result = await client.query("SELECT ...");
    res.end(JSON.stringify(result));
  } finally {
    pool.release(client);
  }
}
```

The `finally` is the leak-prevention muscle memory.

---

## What you should truly know in the AI era (the “god view”) 🧠

You do not need to memorize every engine detail, but you do need these instincts:

1. data model + constraints: what is truth vs cache vs derived data
2. indexing intuition: why a query uses/does not use an index; how to design composite indexes
3. transaction semantics: where you need strong correctness; what isolation is acceptable
4. explain-plan habit: performance = explain first, not “scale up first”
5. pool + capacity thinking: connections, slow queries, and locks amplify each other

---

## Extra 1: Data modeling (how to not sabotage your future self) 🧱

Many “database problems” are not SQL syntax problems.

They are modeling problems.

You do not need to memorize academic normal forms. You do need a few practical rules.

### 1) Model facts and relationships first

Plain English:

- facts: records that must be stored as reality (orders, payments, inventory moves)
- relationships: how facts connect (user_id, order_id, product_id)

```text
User ----< Order ----< OrderItem >---- Product
```

If you can draw this correctly before writing tables, you avoid a huge amount of future pain.

### 2) Constraints are not “annoying”. They protect you.

Beginner mindset:

> “Constraints are annoying, I’ll add them later.”

Then later you get:

- duplicates
- dirty data
- broken relationships
- reconciliation nightmares

At minimum, build comfort with:

- `NOT NULL`
- `UNIQUE`
- (optional) foreign keys, depending on your system boundaries

### 3) Normalization vs denormalization is a trade-off

- normalization: less duplication, safer updates, but more joins
- denormalization: simpler reads, but harder correctness on writes

Rule of thumb:

- ledger-like facts (payments, orders) lean normalized
- read-heavy views (product pages) can denormalize with a consistency strategy

### 4) Common field design traps

- storing money in float/double (precision issues)
- inconsistent timezone handling
- inconsistent collations/charsets (weird comparisons, index surprises)
- soft delete without consistent filtering

---

## Extra 2: Primary keys: auto-increment vs UUID (performance implications) 🆔

### Auto-increment ids

Pros:

- insert patterns are more sequential
- indexes are more compact

Cons:

- easy to guess scale / enumerate ids
- distributed systems need extra id strategy

### Random UUIDs

Pros:

- globally unique
- harder to guess

Cons (important):

- more random insert behavior in B+Trees
- more fragmentation and page splits

### Common compromise: time-ordered ids

Goal:

> global uniqueness + insert friendliness.

You do not need to implement Snowflake on day one, but you should know why teams build these.

---

## Extra 3: Pagination: why OFFSET gets slow, and why cursor pagination is popular 📄

### OFFSET pagination (common, often slow)

```sql
SELECT id, created_at
FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 100000;
```

Often the engine still has to walk through many rows before returning your page.

### Cursor pagination (keyset pagination)

Idea:

> do not say “skip N rows”; say “continue after this point”.

```sql
SELECT id, created_at
FROM orders
WHERE created_at < ?
ORDER BY created_at DESC
LIMIT 20;
```

```text
OFFSET: walk to row 100000, then start
CURSOR: start from last seen created_at, continue
```

---

## Extra 4: N+1 queries (a silent performance killer) 🕳️

Symptom:

- query a list of orders (1 query)
- loop and query each user (N queries)
- total = N+1 queries

Fix patterns:

- use a join
- or batch with `IN (...)` and merge in app memory

---

## Extra 5: Read replicas and replication lag (why “just wrote, can’t read it”) 🔁

Why replicas:

- reads scale more than writes

Why lag:

```text
primary write -> log -> ship -> apply on replica -> readable
```

Practical approaches:

- read-your-writes: route reads to primary after a write for a window
- accept eventual consistency for non-critical pages
- keep critical flows on primary reads

---

## Extra 6: Slow query incident checklist (production firefighting) 🧯

When “DB is slow” alerts fire:

1. confirm it is DB time (not app/event loop/network)
2. capture the slow SQL (do not guess)
3. EXPLAIN it: index usage, rows, sorts/temp tables, join order
4. check system signals: connections, lock waits, IO pressure
5. apply high-ROI mitigations: limit/degrade, better indexes, fewer columns, split queries

---

## Extra 7: Backups and restore (you will be asked) 🧰

Backups are not “exists” — they are “restorable”.

Two key questions:

1. can you restore? (test restores)
2. how recent can you restore? (RPO)

Concepts to know:

- full vs incremental backups
- backup frequency
- restore drills
- backup encryption/access control

---

## Extra 8: ORMs are not magic (use them with eyes open) 🧙

ORMs are useful, but can hide:

- N+1 queries
- over-selecting columns
- overly large transaction scopes
- missed indexing opportunities

Mature habit:

> use an ORM for productivity, but be able to read its SQL and validate the plan with EXPLAIN.

---

## Next questions 🚀

1. How do WAL/redo/undo logs enable crash recovery?
2. Why does replication lag exist, and how do we design consistency around it?
3. What do sharding and partitioning actually solve, and what do they break?
