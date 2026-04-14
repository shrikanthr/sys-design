# Live Comments — System Design Cheat Sheet

## Core Problem

- Asymmetric traffic: viewers receive far more than they send.
- Two distinct read modes: historical load (scroll-back) vs. live stream. Design them separately.
- Comment write fan-out to potentially millions of concurrent viewers in near real-time.
- Viewership spikes: celebrity goes live, 2M people join in 60 seconds. Can't pre-compute anything.

---

## Client Communication — SSE not WebSocket

- Comments are asymmetric — viewers mostly receive, rarely send.
- SSE is one-directional server → client. Correct choice. WebSocket is overkill and wastes resources.
- Comment writes are a direct HTTP POST to the comments service — separate from the SSE connection.

---

## Write Path

### Ordering matters for dual-write
1. Write comment to Cassandra first — durable commit.
2. Best-effort publish to Redis Streams for live fan-out.
3. If Redis publish fails: comment is durable, viewer just missed the live push. Acceptable loss.
4. If Cassandra write fails: nothing published. Comment never existed. Correct behavior.

### Why not CDC?
- CDC latency (100–500ms) is unacceptable for live comment delivery.
- At 1000 comments/sec on a hot stream, you need near-instant fan-out.

### Why not write Redis first?
- Redis is not a durability guarantee. If Redis goes down before Cassandra write, comments lost forever.
- Always commit to durable store first, fan-out second.

---

## Pub-Sub Tier — Redis Streams

- **Redis Pub/Sub**: ephemeral, low-latency, no durability. Messages lost on crash.
- **Kafka**: durable and ordered but topic proliferation at millions of concurrent streams is operationally ugly.
- **Redis Streams**: right answer. Persistent within a window, ordered, consumer groups, replayable. Lighter than Kafka.
- One stream per video. SSE servers subscribe to the stream for the video(s) they are serving.

---

## Read Path — Two Separate Systems

### Historical load (first 50 comments on join)
- Redis cache of last N comments per video, populated when stream starts.
- 2M people joining all hit the cache, not Cassandra. Cache hit rate ~100% during spike.
- Rate limiting + request queuing protects Cassandra for cache misses.
- Cache is relatively stable — scrollback comments don't change once a viewer joins.

### Live stream
- SSE servers consume from Redis Streams for the relevant video.
- Push new comments to connected clients as they arrive.
- Live window is constantly moving — separate concern from historical cache.

---

## Schema — Cassandra / DDB

```
partition key: video_id
sort key:      (timestamp, comment_id)   ← composite to avoid ms-level collisions
columns:       user_id, comment_text, created_at
```

- **Do not use timestamp alone as sort key** — at high comment velocity, millisecond collisions are guaranteed.
- Use `(timestamp, comment_id)` composite sort key, or a time-ordered UUID (ULID) as sort key.
- Query pattern: `WHERE video_id = X ORDER BY sort_key DESC LIMIT 50` for historical load.

---

## SSE Server Fleet

### Assignment
- New viewer joins → load balancer or coordination service (Redis connection-count key) assigns to SSE server via round robin.
- Round robin is correct here — SSE connections are long-lived but equal weight.

### Stateless servers
- No sticky routing needed. Any SSE server can serve any viewer.
- All SSE servers read from the same Redis Stream for a given video.
- On crash/reconnect, client reassigns to healthy server seamlessly.

### Scaling — pre-warming
- Round robin doesn't solve hotspot at stream start.
- High-follower accounts trigger SSE fleet pre-warming when they start a stream.
- Know who the likely hotspot creators are ahead of time — scale proactively, not reactively.

---

## Key Data Stores

| Store | Purpose | Notes |
|---|---|---|
| Cassandra | Durable comment storage | Partitioned by `video_id`, sorted by `(ts, comment_id)` |
| Redis Streams | Live comment fan-out | One stream per video, consumed by SSE servers |
| Redis Cache | Last N comments per video | Populated on stream start, protects Cassandra on spike |
| Load balancer / Redis | SSE server assignment | Connection count tracking, round robin |

---

## Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Redis publish fails after Cassandra write | Viewer misses comment in live stream | Acceptable — comment exists, shows in scroll-back |
| SSE server crashes | Connected clients lose stream | Clients reconnect, reassign to healthy server |
| Cassandra write fails | Comment lost | Nothing published — correct behavior |
| Cache cold start on spike | 2M reads hit Cassandra | Rate limit + queue reads, cache warms quickly |
| Hot stream start — fleet undersized | SSE servers overwhelmed | Pre-warm fleet for known high-follower creators |

---

## IC5 Talking Points

- Separating historical load from live stream is the key design insight. Most candidates design one system that does both badly.
- SSE over WebSocket shows you understand the access pattern, not just the technology.
- Write Cassandra first, Redis second — failure mode reasoning is what interviewers want to hear.
- Redis Streams over Kafka for this use case — know why (topic proliferation, operational overhead).
- Pre-warming is the mature answer to spike handling. Reactive autoscaling alone isn't enough at Meta scale.
- Composite sort key `(timestamp, comment_id)` shows you've thought about write collision at scale.
