# Chapter 7: Redis and Caching (Redis is not the point — “why caching exists” is) 🧊

> Your `outline.md` is spot on: Redis is not the point.
> The point is why caches exist, when you should cache, and when you absolutely should not.
> AI can generate “some caching code”, but the real pain is system-level: consistency, stampedes, avalanches, hot keys, capacity, and fallback strategy.

We follow your outline flow:

`Memory -> Redis -> Distributed Cache -> Penetration/Breakdown/Avalanche -> Consistency/Eviction -> LRU/LFU/TTL -> Pipeline/Lua/Transactions -> When not/when must cache -> Cache DB/API/HTML`

---

## The Three Questions

### 1) Why do caches exist? 🤔

Because “always hit the database or remote services” is expensive:

- slow: disk + network latency
- costly: database connections and CPU are precious
- fragile: one traffic spike can crush your backend

Caching is basically one simple wish:

> **Do not recompute and refetch the same thing repeatedly.**

Think of it like keeping frequently used items within reach:

- database = archive room (reliable but far)
- cache = sticky note on your desk (fast but not forever)

### 2) Why Redis? Are there alternatives? 🧰

Caching is layered. Redis is a major layer, not the only one.

If you think “closer to the user = faster”, caching layers look like:

```text
Browser cache (closest)
  |
CDN cache
  |
Nginx / reverse proxy cache
  |
In-process memory cache (single instance)
  |
Redis (shared distributed cache)
  |
Database (source of truth)
```

Redis is common because:

- rich data structures
- in-memory performance
- shared across many app instances
- TTL, eviction policies, scripting/atomic operations

Alternatives:

- in-process cache: fastest but not shared across instances
- Memcached: simpler KV caching

Plain English:

> Redis is a high-ROI, widely-used distributed cache component.

### 3) How does caching work in practice (for developers)? 🔍

In practice, caching design is mostly about:

- what to cache
- how to invalidate
- what consistency you can tolerate
- how to protect the backend during spikes and failures

---

## Part 0: Two boundaries you must learn 🧠

### 0.1 Cache is not the source of truth

In most systems:

- database = truth
- cache = acceleration layer

You can drop caches. You cannot drop the database.

### 0.2 Cache trades consistency for performance

Most caching designs accept:

> short-lived staleness for better overall performance and stability.

Some domains cannot accept this (money, inventory). We cover those below.

---

## Part 1: Start from memory (why it’s fast) ⚡

You do not need exact numbers, but you do need the order of magnitude:

```text
CPU/RAM: ns–µs
Network: ms (and jitter)
Disk: ms
```

Caching is about moving hot data closer to compute and users.

---

## Part 2: What does Redis do (think in use-cases, not commands) 🎯

Common Redis use-cases:

1. hot-read caching (product info, config, public content)
2. sessions/login state
3. counters/rate limiting
4. leaderboards (sorted sets)
5. distributed locks (use carefully; semantics matter)
6. queues / async patterns (depending on feature choice)

Redis is not “only a cache”. It is a high-performance in-memory data structure toolbox.

---

## Part 3: Distributed cache: what problem does it solve? 🌍

If you only use in-process memory caches:

```text
Request -> App1: cache hit ✅
Request -> App2: cache miss ❌ (hits DB again)
```

Distributed cache solves:

> **multiple app instances sharing the same cache.**

```text
            +-------> App1 ----+
User -----> |                 |----> Redis (shared cache)
            +-------> App2 ----+
                               |
                               v
                              DB (truth)
```

---

## Part 4: The classic pattern: Cache Aside 🛣️

### 4.1 Read path

```text
Request
  |
  v
Check Redis
  | hit  -> return
  | miss -> query DB -> write Redis -> return
```

### 4.2 Write path (common safe choice)

Often:

1. write DB
2. delete cache (force next read to repopulate)

Plain reason:

> updating cache correctly under concurrency is tricky; deleting is safer.

### 4.3 Node pseudo-code (library-agnostic)

```js
async function getUser(userId) {
  const cacheKey = `user:${userId}`;
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const user = await db.queryOne("SELECT id, email FROM users WHERE id = ?", [
    userId,
  ]);
  await redis.set(cacheKey, JSON.stringify(user), { EX: 60 });
  return user;
}

async function updateUserEmail(userId, email) {
  await db.exec("UPDATE users SET email = ? WHERE id = ?", [email, userId]);
  await redis.del(`user:${userId}`);
}
```

The point is the ordering and the mental model, not the exact client API.

---

## Part 5: Cache Penetration / Breakdown / Avalanche (the three nightmares) 👻

### 5.1 Penetration: the key does not exist

Symptoms:

- repeated requests for non-existent ids
- Redis always misses
- everything hits DB

```text
Request user:-1
  |
  v
Redis miss
  |
  v
DB miss (but still forced to work)
  |
  v
Repeat -> DB gets hammered
```

Mitigation:

- input validation
- cache empty results with short TTL
- Bloom filters

### 5.2 Breakdown (stampede): a hot key expires at the worst moment

Symptoms:

- one extremely hot key expires
- many concurrent requests miss at once
- thundering herd hits DB

Mitigation:

- singleflight / mutex rebuild
- TTL jitter
- warmup

### 5.3 Avalanche: many keys expire together, or the cache layer fails

Symptoms:

- many keys share the same TTL -> expire together
- or Redis is down
- huge traffic falls back to DB -> DB collapses

Mitigation:

- TTL jitter
- multi-level caching (local + Redis)
- graceful degradation
- rate limiting

---

## Part 6: Cache consistency: how consistent do you need to be? ⚖️

There is no silver bullet. There are trade-offs.

Ask first:

> can this data be slightly stale?

Common targets:

- strong consistency: money, inventory, permissions
- eventual consistency: product pages, configs, public content

Common strategy:

- write DB, delete cache

Two real-world pitfalls:

1. what if cache deletion fails? (retry/compensation)
2. what if concurrent writes cause stale re-population? (versioning / more advanced patterns)

---

## Part 7: Eviction: LRU / LFU / TTL (remember the instinct, not the formula) 🧹

Redis memory is finite, so eviction happens.

- TTL: expires by time
- LRU: evict what was not used recently
- LFU: evict what is used least frequently

Developer focus:

- is your cache primarily time-based or capacity-based?
- what happens under memory pressure?
- can evictions cause a fallback storm?

---

## Part 8: Pipeline: why can it be faster? 🚄

Redis is fast, but network round trips are expensive.

Pipeline is:

> do not ask one question per trip; send many, then receive many.

```text
No pipeline:
req1 -> res1 -> req2 -> res2 -> ...

Pipeline:
req1..req100 (one trip) -> res1..res100 (one trip)
```

---

## Part 9: Lua: the “atomic multi-step” superpower 🧠

Single Redis commands are atomic, but business logic often needs multiple steps.

Lua lets you bundle multiple steps into one atomic script.

Classic example (rate limiting skeleton):

```lua
local current = redis.call("INCR", KEYS[1])
if current == 1 then
  redis.call("EXPIRE", KEYS[1], ARGV[2])
end
if current > tonumber(ARGV[1]) then
  return 0
end
return 1
```

Instinct:

> Lua turns “multiple operations” into “one atomic operation”.

---

## Part 10: Redis transactions (MULTI/EXEC) are not DB transactions 🧾

Plain model:

- `MULTI` begins queuing
- commands queue up
- `EXEC` runs them as a batch

This is not the same as full ACID DB transactions.

For strict atomic multi-step logic, Lua is often simpler.

---

## Part 11: When should you NOT cache? When must you cache? 🚦

### 11.1 Avoid caching or be very careful

- money, inventory, permissions (strong correctness)
- private/sensitive user data (leaks and mix-ups)
- highly real-time state
- write-heavy, read-light data (low ROI, high consistency cost)

### 11.2 You should cache (often)

- read-heavy, write-light data with clear hot keys
- downstream is expensive (DB heavy, third-party expensive)
- traffic is spiky and needs smoothing

### 11.3 Cache DB vs API vs HTML

Think by how close it is to the user:

```text
HTML cache (CDN/Nginx) -> closest
API response cache (Nginx/app/Redis)
DB query result cache (Redis)
DB (truth) -> most reliable but most expensive
```

---

## Extra 1: Key design (Redis feels “easy” until your keys become chaos) 🗝️

Most Redis pain in production starts with messy keys.

### 1) The three-part habit: namespace / identity / version

Start early with a pattern like:

```text
{namespace}:{entity}:{id}:v{version}
```

Examples:

```text
user:profile:123:v1
product:detail:888:v2
order:summary:10001:v1
```

Why version?

Because you will change cache payload shape sooner than you think.

Versioning lets you roll forward without decoding old values incorrectly.

### 2) Key granularity: too coarse vs too fine

- too coarse: giant JSON blobs -> big keys
- too fine: one page needs 30 keys -> too many round trips

Rule of thumb:

- read-heavy display data can be a bit coarser
- correctness-critical state should not be “cleverly assembled” in Redis

### 3) Serialization choice

- JSON is fine for many cases
- if you frequently update individual fields, hashes can be more natural

---

## Extra 2: TTL strategy (the most common root cause of incidents) ⏱️

### 1) Do not make TTLs perfectly aligned

If many keys share the same TTL (like 60s), you invite avalanches.

Add jitter:

```text
ttl = base + random(0..base*0.1)
```

### 2) TTL is also “how stale can this be?”

Define TTL in business terms:

- config: can it be 1 minute stale?
- product detail: can it be 5 minutes stale?
- order status: can it be 1 second stale? (often no)

### 3) Logical expiration

Idea:

- keep the key “alive”
- store `expireAt` inside the value
- serve slightly stale data while rebuilding asynchronously

This is a “availability first” trick:

> better slightly stale than a DB meltdown.

---

## Extra 3: Hot keys / Big keys (two very common Redis production traps) 🔥🪨

### 1) Hot key

Symptoms:

- Redis CPU spikes due to one extremely hot key

Mitigations:

- multi-level caching (local + Redis)
- sharding a hot key into multiple keys and merging in the app
- warmup + mutex rebuild
- move caching up to CDN/Nginx if possible

### 2) Big key

Symptoms:

- single operations become slow
- network payload is large
- replication/AOF pressure increases

Mitigations:

- split structures (hash/list sharding)
- store only what is needed
- avoid indefinite big keys

---

## Extra 4: Redis is not “never losing data” (persistence + HA basics) 🧯

### 1) Persistence

Redis is in-memory, but persistence exists:

- RDB snapshots
- AOF append-only logs

Plain model:

- RDB: periodic photos
- AOF: write-ahead diary

For caching, losing data may be acceptable.

For “state” usage (queues, locks, sessions), you must be more careful.

### 2) High availability

Common shapes:

- replication
- sentinel failover
- cluster sharding + fault isolation

You do not need to memorize commands here, but you should know the problems these modes solve.

---

## Extra 5: Distributed locks (easy to write, easy to get wrong) 🔒

Typical pattern starts with:

- `SET lock_key value NX PX 30000`

Minimum pitfalls you must know:

1. locks need expiration (avoid deadlocks)
2. unlocking must validate ownership (do not delete someone else’s lock)
3. idempotency and retries still matter (locks are not magic)

Sometimes simpler approaches are better:

- idempotency tokens
- DB unique constraints

---

## Extra 6: Caching and security (often ignored, often disastrous) 🧨

Two instincts:

1. do not cache sensitive user data casually
2. do not let different users share the same key space accidentally

Classic incident:

- a key misses `userId` -> user A reads user B’s cached data

So you often see:

```text
user:{userId}:...
tenant:{tenantId}:...
```

---

## What you should truly know in the AI era (the “god view”) 🧠

1. caching layers: browser/CDN/Nginx/local/Redis/DB
2. incident models: penetration/breakdown/avalanche
3. consistency trade-offs: what can be eventual vs must be strict
4. backend protection: rate limit, degrade, warmup, jitter, mutex rebuild
5. atomic tools: Lua, pipeline, and sane key design

---

## Next questions 🚀

1. How do Redis HA modes (replica/sentinel/cluster) map to real problems?
2. How do you find and fix hot keys in production?
3. Why are distributed locks easy to get wrong, and what does “correct semantics” mean?
