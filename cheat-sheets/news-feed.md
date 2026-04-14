# News Feed — System Design Cheat Sheet

## Core Problem

- 1 post write → N feed updates. Write amplification is the central challenge.
- Celebrity with 50M followers cannot do synchronous fan-out.
- Tension: ranking freshness vs. read latency vs. write cost.
- Strong consistency on post write. Eventual consistency on fan-out (10–30s lag is fine).

---

## Write Path

### Post persistence
- Write to `posts` table keyed by `creator_id`. Source of truth. Strong durability.

### Fan-out trigger
- CDC (Debezium) on posts table → Kafka topic.
- Avoids 2-phase write failure — post write commits, CDC fires the event, fan-out workers consume async.
- Recovery is free: replay Kafka on failure.

### Normal users — write fan-out
- Pre-populate feed cache per follower at write time.
- Fast reads, expensive writes.

### Celebrities (>10K followers) — read fan-in
- Skip fan-out entirely at write time.
- Pull celebrity posts at read time, merge into pre-built feed.
- Threshold is configurable and can be dynamic.

---

## Read Path

### Feed cache structure
- Per-user list of `(post_id, precomputed_score, insertion_ts)`.
- IDs only — not full payloads. Post content hydrated separately in parallel.

### Celebrity merge
- On feed load: fetch pre-built feed cache + pull celebrity posts from their tables → merge + re-rank → return top N.

### Pagination cursor — snapshot token
- On first open, server issues a feed snapshot timestamp T.
- All subsequent pages reference that snapshot — new posts arriving during scroll don't shift positions.
- New posts surface as a "X new posts" banner at the top, not mid-scroll injection.

### Post hydration
- After ranking, fetch content, author info, media from separate stores in parallel.
- Feed cache is a lightweight ID list, not a document store.

---

## Ranking

### Two-pass model
- **Offline / candidate generation**: pull ranked post IDs from feed cache.
- **Online / scoring**: lightweight re-score at read time using precomputed + dynamic features.
- Heavy ML reranking runs async in background, writes updated scores back to feature store.

### Static features (precomputed, updated daily)
- Author affinity score
- Relationship strength
- Historical engagement rate of the post/author
- Stored alongside `post_id` in feed cache

### Dynamic features (fetched at score time)
- Current like/comment velocity
- Close-friend engagement in last 2h
- Session-level signals: what you just liked in this session (strongest real-time signal)

### Feature store
- Low-latency KV store (Redis / Meta's Monarch).
- User preference vector keyed by `user_id`.
- Must be <10ms p99 — it's on the critical read path.
- Updated daily or on significant behavior event.

### Scoring mechanics
- Lightweight dot product or shallow model on the read path.
- Not a full deep neural net per request.
- Heavy model runs offline, pushes updated scores back to feature store periodically.

---

## Key Data Stores

| Store | Purpose | Notes |
|---|---|---|
| Posts table | Source of truth | Sharded by `creator_id`, strong consistency |
| Feed cache | Per-user ranked post ID list | Redis/Memcached, pre-built for normal users |
| Following index | "Who does user X follow" | Used at fan-out time by workers |
| Follower index | "Who follows user X" | Used to identify celebrity followers |
| Feature store | User preference vectors + engagement signals | KV, <10ms reads, updated async |
| Kafka | CDC events → fan-out workers | Durable, ordered, replayable |

---

## Indexes

- **Following index**: `user_id → [list of followed user_ids]` — fan-out worker looks this up to know whose feeds to update.
- **Follower index**: `celebrity_id → [list of follower_ids]` — needed if you ever want to do a backfill or forced push.
- These are separate stores, not the same index read both ways.

---

## Key Trade-offs

**Write fan-out vs read fan-in**

| | Write fan-out | Read fan-in |
|---|---|---|
| Read cost | Cheap (pre-built) | Expensive (merge at read time) |
| Write cost | Expensive (N writes per post) | Cheap (1 write) |
| Stale feeds | Risk for inactive users | Less risk |
| Best for | Normal users (<10K followers) | Celebrities |

**Hybrid is the answer.** Know the threshold and be able to defend it.

---

## IC5 Talking Points

- Most candidates describe fan-out correctly and stop. Ranking layer + feature store is the differentiator.
- Snapshot cursor is a real Meta pattern — most candidates have no answer for the scroll pagination problem.
- Following vs follower index distinction shows implementation depth.
- Frame the feature store like a Pinot/OLAP read layer: precomputed, low-latency, high fan-out reads. Direct parallel to HubSpot work.
- The interesting failure mode to mention: what happens to the feed cache for a user who hasn't opened the app in 6 months? (Lazy rebuild on next open, or TTL eviction and cold start.)
