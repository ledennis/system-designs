# Rate Limiter Mock Interview: Full Notes

These notes cover the whole mock interview: the prompt, the requirements the interviewer gave, the estimation, the design, the follow-up questions with strong answers, the deep dives we did afterward, a glossary, general tips and cheat sheets, and the debrief.

## Contents

1. The prompt and how the interview ran
2. Step 1: Requirements
3. Step 2: Estimation
4. Step 3: High-level design
5. The interviewer's follow-up questions, with strong answers
6. Consistent hashing
7. Rate limiting algorithms
8. Atomicity
9. Where the limiter lives
10. Topics not covered in depth
11. Glossary
12. Interview tips and cheat sheets
13. Debrief

---

# 1. The prompt and how the interview ran

**Prompt:** "Design a rate limiter. It should limit how many requests a client can make to an API within a given time window, and reject requests that go over the limit."

The interview followed five steps:

1. Clarify requirements
2. Estimate scale
3. High-level design
4. Deep dive
5. Bottlenecks, scale, and failure

## How long a real interview lasts

A system design round is usually 45 to 60 minutes, and some companies use 35 to 45. That includes introductions and a few minutes at the end for your questions, so you get roughly 35 to 50 minutes of actual design time. A typical budget:

| Phase | Time |
|---|---|
| Requirements | 5 min |
| Estimation | 3-5 min |
| High-level design | 10 min |
| Deep dive | 15 min |
| Bottlenecks, failure, wrap-up | 5-10 min |

## How this mock differed from a real interview

This mock was much deeper and slower than a real round. There was no time limit, and the interviewer probed every gap with several questions at once. A real interviewer has maybe 10 minutes for the deep dive, picks one or two areas to probe, and often leaves unaddressed gaps unmentioned. A strong 45-minute answer covers requirements, a rough estimate, a clean high-level design, one deep dive, and a brief mention of failure and scale. Nobody expects every detail to be resolved. Going slowly and thoroughly while learning is the right way to practice. Speed comes later, from repetition.

---

# 2. Step 1: Requirements

## What the interviewer told you

| Question | Interviewer's answer |
|---|---|
| Default limit per client | 100 requests per client. Fixed for the exercise (in real life limits get raised per customer, but that's out of scope). |
| Time window | 1 minute |
| Error when the limit is hit | HTTP 429 Too Many Requests, with a `Retry-After` header saying when the client can try again |
| Authentication | The client sends a user token that resolves to a client ID. Limit per client ID. Authentication itself is out of scope. |
| Kind of requests | Ordinary HTTP API calls (REST over HTTPS, JSON bodies). Each carries the token, which resolves to a client ID, plus the endpoint and method. |
| Scope of the limit | 100 per minute across all of the client's requests. No per-endpoint limits for now. |
| Data the limiter needs | Only who sent the request (client ID) and when. Payload contents don't matter. |
| Where the API runs | A fleet of application servers behind a load balancer, so one client's requests can land on different servers |
| Traffic shape | Clients are not uniform: some steady, some bursty, a few heavy users who regularly hit the limit |
| Read/write mix of the API | Generic mix (your assumption, accepted). It doesn't change the limiter design. |
| Scaling | Dynamic. Adding capacity must not need downtime or dropped limits that are already being tracked. |
| Scale numbers | About 10M registered clients, about 1M active in any given minute, doubling over the next year. About 1M requests/sec at peak. |
| Latency | The check should add no more than a few milliseconds |
| Availability | High. The limiter can't be something that takes the API down. |

## What each answer changed in the design

| Requirement | Design consequence |
|---|---|
| 100/min per client | Token bucket parameters (refill rate and burst capacity) |
| 429 with `Retry-After` | The node that makes the decision must also compute the retry time |
| Authenticated token | The limiting key is the client ID, and the token should be verifiable at the gateway without a network call |
| Fleet behind a load balancer | Counters must be shared across gateways, not kept per server |
| Bursty and heavy clients | Choose a burst allowance on purpose, and be ready for hot-client concerns |
| Dynamic scaling | Consistent hashing, so adding nodes doesn't reshuffle everything |
| 1M req/sec | Roughly 20 counter nodes today, driven by throughput |
| A few milliseconds | One round trip for the check, no extra hops |
| High availability | Fail-open with a local fallback limit and alerting |

## Questions worth asking for a rate limiter

You asked about the limit, window, error, authentication, request types, and scaling. These are others to run through next time:

- Is the limit per user, per IP, per API key, or per endpoint?
- Hard limit or soft limit (can it be briefly exceeded)?
- One data center or multiple regions?
- What should happen if the limiter itself fails?
- Do rejected requests count toward the limit?
- How tolerant of bursts should it be?
- Should responses tell clients how many requests remain?
- Can limits be changed without a redeploy?

For this problem the read/write ratio mattered much less than it did in the URL shortener. The more useful question is about the limiter's own workload: what does it do for every incoming request, and how often?

You don't need to ask everything. Pick the two or three that would change the architecture and state assumptions for the rest.

---

# 3. Step 2: Estimation

## Numbers you were given

- About 10M registered clients, about 1M active per minute
- About 1M requests/sec at peak
- Client base doubles in a year, so about 20M registered, about 2M active, and about 2M requests/sec at peak next year

## Your work, and what held up

- Average from peak: if peak is 2x the average, average is 500K requests/sec. Correct. Capacity gets sized for peak, so the average matters little here.
- Consistency check: 1M active clients sending 1M requests/sec is about 1 request per second per client, or about 60 per minute, under the limit of 100. So most clients are well-behaved. Rejected requests still cost the limiter a check, so rejected traffic counts toward its load.
- Peak is an assumption you state, not something you derive. A common rule of thumb is 2 to 5x the average.

## Node sizing

The method:

1. Assume what one node can handle. State it out loud.
2. Divide peak load by it.
3. Add headroom (2x is common) and replicas for availability.
4. Plan for next year's growth.

The per-node figure isn't derived from first principles. It's a rule of thumb: Redis is commonly benchmarked at around 100K simple operations per second per instance, and one limit check costs one or two operations, so plan on about 50K checks per second per node, and benchmark on your own hardware before committing.

> "I'll assume a single in-memory node handles about 100K simple operations per second, and each limit check costs one or two operations, so I'll plan for roughly 50K checks per second per node. In practice I'd benchmark it on our hardware."

Result: 1M / 50K = 20 nodes minimum today. Next year's 2M/sec means about 40 nodes, before headroom and replicas.

## State size

What each client needs: a key (the client ID, tens of bytes), a token count, a timestamp, and the store's own per-key overhead. A realistic planning number is about 100 bytes per client.

| Population | Memory |
|---|---|
| 10M registered clients | about 1 GB |
| 1M active in a minute | about 100 MB |
| After doubling | about 2 GB and about 200 MB |

Your answer was "a byte per client" and "10M bytes = 10 GB". The count itself does fit in a byte, but the real record is bigger, and 10M bytes is 10 MB, not 10 GB.

The conclusion matters more than the number: all of this state fits on a single node, so the 20 nodes exist for throughput, not memory. Saying that out loud is a strong point, and it shapes how you shard.

---

# 4. Step 3: High-level design

## The design

```
client
  -> API gateway
       1. verify the signed token locally -> client ID
       2. consistent hash(client ID) -> counter node
       3. one atomic check (Redis Lua script) -> allow or reject + retry time
  -> allowed:  API service -> response to client
  -> rejected: 429 + Retry-After straight back to the client
```

Components and the reason for each:

- **API gateway** is where the limit is enforced. It's the earliest point where the client ID is known, so rejected requests never touch the services.
- **Signed token (JWT style)**, verified locally by the gateway. No database call per request.
- **Consistent hashing** so each client's checks go to the same counter node, and adding nodes moves only about 1/N of the clients.
- **Counter nodes (Redis)** hold one bucket per client and run the check atomically.
- **Replicas** for each node, so a failure promotes a replica.
- **Fail-open fallback**: if the shared store can't be reached, gateways apply a local in-memory limit and page the team.

## Walking one request through

Allowed path:

1. The client sends a request with its token.
2. The gateway verifies the token locally and gets the client ID.
3. The gateway hashes the ID to a counter node and sends one atomic check.
4. The node refills the bucket from elapsed time, sees at least one token, takes it, and replies "allowed."
5. The gateway forwards the request to the API service, and the response goes back to the client.

Rejected path:

1. Steps 1 to 3 are the same.
2. The node sees an empty bucket and replies "rejected" with the retry time. It doesn't take a token.
3. The gateway answers 429 with `Retry-After`, and the API service never sees the request.

## Hops and latency

The limiter adds one round trip (gateway to counter node and back), roughly 1 to 2 ms inside a data center. Everything else on the path was already there: client to gateway, gateway to service, and back. That fits the few-millisecond budget.

## Your original design versus the improved one

| Your first design | Improved version |
|---|---|
| A coordinator next to the load balancer looks up each client's node | Gateways compute the node themselves with consistent hashing. This removes an extra hop, a bottleneck at 1M req/sec, and a component that needs its own failover. |
| A token service with a database of client IDs | Signed tokens verified locally. A database call per request wouldn't survive 1M/sec. |
| A lock around the check | One atomic Lua script in Redis |
| Hash plus salt for the client mapping | A plain, evenly spreading hash. No salt needed. |
| Bucket capacity 200 | Capacity chosen on purpose for the burst you intend |
| The token bucket "authorizes" requests | It rate limits (admits or throttles) requests. Authorization is a different thing. |

---

# 5. The interviewer's follow-up questions, with strong answers

## Which algorithm, and what exactly does the node do on a check?

Token bucket with lazy refill. Each client's record holds a token count and the time of the last refill. When a check arrives:

```
tokens = min(capacity, tokens + (now - last_refill_time) x rate)
last_refill_time = now
if tokens >= 1: tokens -= 1 -> allow
else: reject
```

Nothing runs per bucket in the background. Idle clients cost nothing, and capacity and rate are global config rather than per-client data.

## What are the bucket parameters?

- Refill rate: 100 per minute is 100/60, about 1.67 tokens per second. That's one token every 0.6 seconds. Your 1 token per second was 60 per minute, which falls short of the requirement.
- Capacity is the largest burst allowed from a full bucket. Capacity 200 means 200 requests in one second, not 100.
- Worst case in one window: a client starting from a full bucket can send about capacity + refill x window. With capacity 100 and refill 100/60, that's up to 200 in the worst minute. With capacity 200 it's up to 300. With capacity 20 it's about 120.
- So capacity is a tuning knob. State the burst you intend, then derive the numbers. If the business needs "never more than 100 in any 60 seconds," that's a different algorithm (sliding window counter or log).

## How do 1M buckets get refilled?

They don't. Refill is computed lazily when a request arrives, from `now - last_refill_time`.

## What guarantees only one of two simultaneous requests gets the last token?

The check must be atomic, not idempotent. Read, refill, compare, decrement, and write happen as one indivisible step, so two checks can't both see "1 token left." The standard way is a Redis Lua script, since Redis runs one command or script at a time. The full explanation and code are in section 8.

## What does the lock cover, and what happens with a very busy client?

If you did use a lock, it should cover one client's bucket, never the whole node, so a busy client doesn't block others. The better answer avoids the lock entirely and uses the atomic script. A very busy client is still a concern for its node, which is the hot-client topic in section 10.

## How many hops is a request, and where does the latency budget go?

Count each stop: client, gateway, counter node, back to the gateway, API service, back to the client. Only the gateway-to-node round trip is added by the limiter (about 1 to 2 ms). The rest would exist without a limiter. If you add a coordinator in the path, that's one more hop on every request.

## Does the token service get called on every request?

It shouldn't. At 1M requests/sec a database lookup per request would be the bottleneck. Use signed tokens (JWT style) that the gateway verifies with a local cryptographic check and no network call. The client ID is inside the token.

## Why salt the client-to-node hash, and what happens going from 20 to 40 nodes?

A salt isn't encryption. It's random data mixed into a hash input so identical inputs don't produce identical hashes, which mainly defends stored secrets like passwords against precomputed lookup tables. Routing only needs a hash that spreads clients evenly, so leave the salt out. Going from 20 to 40 nodes with plain `hash mod N` would remap about half the clients (section 6 explains why mod N reshuffles so much), and with consistent hashing it moves only about the share the new nodes take over, who simply start with fresh buckets.

## Does a rejected request count toward the client's total?

No. A rejected request doesn't consume a token, since there's none to take. Counting rejections would keep a throttled client throttled even after it backs off.

## What goes in `Retry-After`?

The number of seconds until a token is available. The node needs only the bucket's current token count and the global refill rate:

```
wait = (1 - tokens) / rate
Retry-After = ceil(wait)
```

With a refill of one token every 0.6 seconds the wait is at most 0.6 s, so the header will almost always say 1. Your earlier answer divided the wrong way around (10/6 seconds instead of 6/10).

## Coordinator questions

The interviewer asked about the coordinator you proposed:

- What does it mean that every request goes through it at 1M req/sec? It becomes a bottleneck and needs its own scaling and failover.
- Where does a lookup table of every client live (10M entries, doubling next year), and how is it kept current? That's a large, constantly changing table that every request depends on.
- What happens to a client's count when you add a node? With a lookup table, you'd have to move entries.

The way to avoid all three is to remove the coordinator. Each gateway computes the target node from the client ID with consistent hashing, so there's no table to store or synchronize. Adding a node moves only about 1/N of the clients, and those clients just start with a full bucket, so no counts need migrating.

## Failover questions

The interviewer asked what a follower knows when it takes over, what happens to in-flight counts, and how the system decides a leader is dead. For this design:

- Each counter node has a replica that replicates asynchronously. On failure the replica is promoted. It may be missing the last few decrements, so clients briefly get a few extra tokens. That's acceptable for a rate limiter.
- Checks in flight during the failure may time out. The gateway handles those through the fail-open path.
- Death is detected with heartbeats or health checks and a timeout, typically by a cluster manager (Redis Sentinel or Redis Cluster does this), with several observers agreeing so one flaky network link doesn't trigger a false failover. The timeout is a trade-off: too eager causes needless failovers, too slow lengthens the outage.
- Even with no replica at all, losing a node only resets its clients' buckets. That's much less severe than losing a database.

## Fail-open: what protects the service, and who enforces the quota?

Fail-open means that when the limiter fails, requests are allowed through. Fail-closed means they're rejected. For a general API, fail-open is the common choice because a limiter outage shouldn't become an API outage. For something security-sensitive, like login attempts, you might choose fail-closed.

The real risk of failing open is that nothing protects the backend or stops abuse while the limiter is down, not that it costs more money. The mitigation:

- Each gateway instance counts requests in its own memory and applies a local limit. This is enforced by the gateway itself, since the shared store is the thing that's down.
- The local limit is approximate, because a client's traffic is spread across gateway instances. Set it conservatively. Dividing the limit by the number of gateways is a starting point if the load balancer spreads requests evenly.
- The API can also shed load when overloaded, which covers many clients spiking at once.
- Page someone, because failing open silently is dangerous.
- The quota number comes from a benchmark or load test.

## If you don't know the term for a mechanism

You asked whether you could simply assume a way to cap requests exists. Assuming a mechanism is fine for peripheral parts (like authentication), but risky for the thing being tested, since the interviewer will likely ask "how?". Better to describe the mechanism in plain words:

> "If the shared store is unreachable, each gateway falls back to counting requests in its own memory. It's approximate, since a client's requests are spread across instances, but it still puts a ceiling on how much any one client can send."

That earns the credit even without the term (it's usually called a local fallback limit or degraded mode). If the interviewer supplies the name, say "right, that's what I meant" and move on.

---

# 6. Consistent hashing

## The problem with `hash(client) mod N`

With 4 nodes, client 10 goes to node `10 mod 4 = 2`. Add a fifth node and it goes to `10 mod 5 = 0`. Changing N changes almost every client's answer: going from 4 to 5 nodes moves about 80% of clients, and going from 20 to 21 moves about 95%. Every moved client lands on a node with no knowledge of them, which for a rate limiter means a wave of fresh buckets.

## The ring

1. Picture the hash output range (0 to 2^32 - 1) bent into a circle, so the top value wraps back to 0.
2. Hash each node's name to put it at a point on the circle.
3. Hash each client ID to put it at a point too. The client belongs to the first node you meet walking clockwise from there.

Adding a node places it at one new point. It takes over only the arc between itself and the node before it, and every other client keeps its owner. On average, adding the (N+1)th node moves about 1/(N+1) of the clients, which is the theoretical minimum. Moves only go to the new node. No client shuffles between existing nodes.

## Virtual nodes

With one point per node, random placement gives some nodes big arcs and others tiny ones, so the busiest node can carry well over its fair share of clients. The fix is to give each physical node many points on the ring (virtual nodes) by hashing `node-3#0`, `node-3#1`, and so on. At around 100 or more virtual nodes per node:

- The load evens out, and the busiest node approaches its fair share (1.0x).
- A new node takes many small slices from every existing node instead of one big slice from a single neighbor.
- When a node dies, its clients spread across all the survivors. With one point per node, the entire load would land on one neighbor, which can overload it and cause a cascade.
- A bigger machine can get more virtual nodes so it takes a proportionally larger share.

The percentage of clients that move stays close to the ideal either way. Virtual nodes improve the evenness of the spread.

## How it works in code

Keep a sorted array of `(position, node)` pairs. To route a client, hash its ID and binary-search for the first position at or after that hash, wrapping to the start if you fall off the end. With 20 nodes x 100 virtual nodes that's 2,000 entries, so a lookup costs about 11 comparisons. Each gateway builds the same ring from the same node list, and that list comes from config or service discovery. The requirement is that every gateway uses the same hash function and the same membership.

## What this means for the rate limiter

- **No data migration.** A client who moves to a new node just starts with a full bucket. The cost is slightly extra permissiveness for one window, which is fine here. A database or a cache of values you can't recompute would need to copy data, and that's why consistent hashing is harder for those.
- **Brief disagreement is tolerable.** Gateways may briefly disagree about membership during a change. A client could hit two nodes for a few seconds and get a few extra tokens.
- **Replicas fall out of the ring.** Replicas go on the next distinct physical nodes clockwise, so a failover target is already determined.

## Pitfalls

- The hash must be identical everywhere and spread evenly. Python's built-in `hash()` for strings is randomized per process, so two gateways would disagree. Use a stable hash like murmur or xxhash.
- Hashing doesn't fix hot keys. One client sending thousands of requests per second still lands on a single node, however evenly the others are spread.
- Alternatives exist (rendezvous hashing, jump consistent hash). They solve the same problem with different trade-offs, and the ring is the one interviewers expect by name.

## Interview tip: how to say it

> "With mod N, changing the node count remaps most clients. I'd use consistent hashing: nodes and clients hash onto a ring, and each client belongs to the next node clockwise, so adding a node moves only about 1/N of the clients. I'd use around 100 virtual nodes per physical node to even out the load and to spread a failed node's clients across many survivors. For a rate limiter, a moved client just gets a fresh bucket, so there's no data to migrate."

---

# 7. Rate limiting algorithms

Each algorithm answers the same question, "should this request be allowed?", with different state and trade-offs.

## Fixed window counter

Cut time into fixed windows (for example, each calendar minute). Keep one counter per client per window, increment it on each request, and reject once it passes 100. The counter expires when the window ends.

- **State:** one number per client.
- **Strength:** trivially simple, and a single atomic increment does the work.
- **Weakness:** the boundary problem. A client sends 100 requests at 12:00:59 and 100 more at 12:01:00. Each window sees only 100, but the service took 200 within two seconds. In the worst case a client gets double the limit.

## Sliding window log

Store a timestamp for every accepted request. On a new request, drop timestamps older than 60 seconds, count what's left, and allow it only if the count is under 100.

- **State:** up to 100 timestamps per client.
- **Strength:** exact. It is never possible to exceed 100 in any 60-second span.
- **Weakness:** memory. As a rough order of magnitude, a sorted set of 100 timestamps per client is around 50x the memory of a counter. For 1M active clients that's gigabytes where counters would be about 100 MB. Raising the limit raises memory linearly.

## Sliding window counter

Keep two fixed-window counters, the current window and the previous one, and estimate the rolling count by weighting the previous window by how much of it still overlaps your look-back span.

Worked example: limit 100, the previous minute had 84 requests, and you're 25% into the current minute with 36 requests so far.

```
estimate = 84 x (1 - 0.25) + 36 = 99
```

99 is under 100, so the request is allowed.

- **State:** two numbers per client.
- **Strength:** smooths out the boundary problem at fixed-window cost.
- **Weakness:** it's an approximation. It assumes the previous window's requests were spread evenly. In practice the error is small, and many teams accept it.

## Token bucket

Each client has a bucket with a capacity and a refill rate. Each request spends a token, and if the bucket is empty the request is rejected.

- **State:** two numbers per client (token count and last refill time), with lazy refill so nothing runs per bucket.
- **Strength:** permits bursts up to capacity while enforcing an average rate, which matches how real clients behave. Memory is tiny.
- **Weakness:** no guarantee of "at most 100 in any 60 seconds." With capacity C and refill rate r, a client starting from a full bucket can send up to C + r x T in a window of length T. With capacity 100 and refill 100/60, that's up to 200 in the worst minute. Capacity (the burst) and refill rate (the sustained average) are two separate knobs.

## Leaky bucket

Requests go into a queue (the bucket) that drains at a constant rate. If the queue is full, new requests are dropped.

- **Strength:** the output rate is perfectly smooth, which is ideal when a downstream system can only handle a steady flow (a payment processor, a legacy backend).
- **Weakness:** queued requests wait, which adds latency and holds requests in memory. A burst gets spread out instead of served immediately. A meter version with no queue behaves essentially like a token bucket.

## Side by side

| Algorithm | State per client | Accuracy | Bursts | Main weakness |
|---|---|---|---|---|
| Fixed window | 1 counter | Loose (up to 2x at boundaries) | Allowed at window edges, by accident | Boundary spike |
| Sliding window log | Up to 100 timestamps | Exact | Not beyond the limit | Memory grows with the limit |
| Sliding window counter | 2 counters | Close approximation | Smoothed | Assumes even spread |
| Token bucket | 2 values | Average rate plus burst size | Allowed, up to capacity | No hard cap per rolling window |
| Leaky bucket | Queue plus timer | Steady output rate | Smoothed into a queue | Added latency, queue memory |

## How to choose

- **Real-world APIs where bursts are normal:** token bucket. This is the default answer for a general API.
- **A hard rolling cap with little memory:** sliding window counter.
- **Exactness matters and the limit is small:** sliding window log. Think "5 login attempts per hour."
- **Simplicity above all, and boundary spikes are tolerable:** fixed window.
- **Protecting a fragile, steady-rate downstream:** leaky bucket.

## Interview tip: how to say it

A window algorithm counts requests in a time span, while a token bucket tracks a balance that refills over time. Keep those two ideas separate.

> "I'd use a token bucket. It allows the short bursts real clients produce, enforces the average rate through the refill, and needs only two values per client, so state stays tiny at a million active clients. The trade-off is that it doesn't guarantee a hard cap in any rolling minute. If the business needed that, I'd switch to a sliding window counter."

---

# 8. Atomicity

## The race

Take a client with one token left, and suppose the check is written as two steps: read the count, then write the new count. Two requests arrive at the same moment, possibly through different gateways.

```
Request A: read tokens -> 1
Request B: read tokens -> 1
Request A: 1 >= 1, so allow; write tokens = 0
Request B: 1 >= 1, so allow; write tokens = 0
```

Both were allowed, but only one token existed. This is a lost update. At 100 requests per minute one extra request is harmless, but a client firing hundreds of requests in parallel can blow well past the limit.

Terminology: **idempotent** means repeating an operation gives the same result as doing it once. **Atomic** means a multi-step operation behaves as one indivisible step that nothing can interleave with. The check needs to be atomic.

## Four ways to make the check safe

### 1. A distributed lock

Acquire a lock on the client's key, read, write, release.

- It's correct, but each check becomes four round trips (acquire, read, write, release) instead of one.
- A holder that crashes leaves the lock stuck, so it needs a timeout. A timeout that fires mid-operation lets a second holder in, which brings back the race.
- Contention from a heavy client queues everyone behind it.

Locks suit rare, slow critical sections. For a hot per-request path they're usually the wrong tool.

### 2. Optimistic concurrency (compare-and-set)

Read the bucket along with a version, then write only if the version hasn't changed. If it has, retry.

- No locks are held, and it works well when conflicts are rare.
- A busy client creates constant conflicts, and each retry is another round trip.

### 3. One atomic operation inside the store

Send the whole check to the store and let it run as a single step.

- For a fixed window counter, Redis `INCR` is already atomic.
- A token bucket needs a read-modify-write (refill, compare, decrement), so you use a Lua script, which Redis runs as one uninterruptible unit.
- DynamoDB's equivalent is a conditional update: the write only succeeds if the condition still holds.

This is the standard answer. One round trip, no locks, no retries.

### 4. Serialize by ownership

Redis processes commands one at a time, and consistent hashing sends all of a client's checks to the same node. So concurrent requests from one client queue up in a single place and are handled in order. Together with option 3, this is why there's no race in the design, even with many gateways.

## What the atomic check looks like

```lua
-- KEYS[1] = client bucket; ARGV[1] = capacity, ARGV[2] = tokens per second
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local t = redis.call('TIME')
local now = t[1] * 1000 + math.floor(t[2] / 1000)

local d = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(d[1])
local ts = tonumber(d[2])
if tokens == nil then tokens = capacity; ts = now end

tokens = math.min(capacity, tokens + math.max(0, now - ts) / 1000 * rate)

local allowed, retry = 0, 0
if tokens >= 1 then
  tokens = tokens - 1
  allowed = 1
else
  retry = math.ceil((1 - tokens) / rate)
end

redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('EXPIRE', KEYS[1], math.ceil(capacity / rate) + 1)
return {allowed, retry}
```

Details worth noticing:

- This is the lazy refill plus the decrement, all in one step. The gateway gets back a yes/no and the `Retry-After` value.
- The clock is read from the store, not passed in from the gateway. Gateways' clocks drift apart, and a single clock source avoids refill math going wrong. The `max(0, ...)` guards against time appearing to go backward.
- The `EXPIRE` is set to slightly more than the time needed to refill a bucket completely. A bucket that expires would have been full anyway, so deleting it loses nothing, and idle clients take no memory.
- The script has no loops, so it finishes in microseconds. Redis can't run anything else while a script executes, which is why scripts should stay short.

## Failure edges

- A node fails over to an asynchronous replica and loses a few of the latest decrements. Clients get a few extra tokens, which is acceptable for a rate limiter.
- A gateway times out waiting for the node and retries, so the same request could be counted twice. Slight over-counting is fine. A timeout is also exactly what triggers the fail-open path.
- In a Redis Cluster, a script's keys must live on the same shard. That's naturally true here, since each check touches one client's key.

## Interview tip: how to say it

> "A read-then-write check has a race: two requests can both read one token left and both be allowed. I'd avoid a distributed lock because it adds round trips and failure modes on the hot path. Instead the whole check runs as one atomic Lua script in Redis: refill, compare, decrement, return allow or reject with the retry time. And because consistent hashing sends a client's checks to one node, they're serialized there."

If the interviewer pushes on durability, add: "A failover can lose a few recent decrements, so clients might briefly get a few extra tokens, which I'd accept for a rate limiter."

---

# 9. Where the limiter lives

There are several defensible placements, and the usual strong answer combines more than one. Placement is a trade-off between how early you can reject a request and how much you know about it at that point.

## The options

| Placement | What it knows | Strength | Weakness |
|---|---|---|---|
| Client-side (SDK) | Its own calls | Spares bandwidth, enables backoff | Not enforceable, since clients can be modified or hostile |
| Edge, CDN, or WAF | IP, headers, path | Stops floods before they reach your infrastructure | Coarse, and usually doesn't know your authenticated client ID |
| API gateway or load balancer | Client identity (after auth), endpoint | One enforcement point for every service, rejects before any app work | Sits on the request path, and needs shared counters |
| Inside the service (middleware) | Everything: endpoint cost, user tier, business rules | Fine-grained limits like "5 exports per hour" | Each team reimplements it, and the request has already consumed a connection and some work |
| Sidecar or service mesh | Service-to-service calls | Enforced outside app code, good for internal traffic | More configuration and moving parts |
| Dedicated limiter service | Whatever the caller passes in | Reusable by gateways and services, owns the counter logic | An extra network call, and one more thing that must stay available |
| At the resource (DB limits, queue backpressure) | Resource load | Last line of defense | Too late to protect anything upstream |

## Why more than one answer is right

The deciding questions:

- **What are you protecting?** Raw volume (an edge concern), fairness between customers (a gateway concern), or one expensive operation (a service concern).
- **What identity do you have at that point?** The edge sees an IP. After authentication, the gateway sees a client ID. A service sees which endpoint and what it costs.
- **How expensive is a rejected request?** A request dropped at the edge costs almost nothing. One rejected inside a service has already used a connection, parsing, and maybe a database call.
- **Who owns it?** A central platform team favors one gateway policy. Many independent teams may end up with limits in each service.

A rule of thumb: reject as early as possible, but as late as necessary to have the information you need.

## Layering in practice

1. **Edge:** per-IP limits and DDoS protection.
2. **Gateway:** per-client quotas.
3. **Service:** limits tied to expensive endpoints or business rules.
4. **Resource:** connection pools and queue limits, so an overload doesn't take down the database.

## For this problem

Enforce at the API gateway. The requirement is a per-client limit across all endpoints, and the gateway is the earliest point where the client ID is known, so rejected requests never touch the services. The counters live in a separate shared store (the counter nodes), which makes the design a hybrid: the gateway is where the limit is enforced, and the counter nodes are where the state lives. Treat those as two separate placement decisions.

## Interview tip: how to say it

> "I'd enforce at the API gateway because that's the earliest point where we know the client ID, and rejecting there means no backend work is spent on rejected requests. The counters live in a shared store so the limit holds across gateway instances. I'd also put coarse per-IP limits at the edge, and add service-level limits later for expensive endpoints if needed."

---

# 10. Topics not covered in depth

These came up as possible follow-ups. We didn't drill into them, so treat these as short primers.

## Hot clients

One client sending thousands of requests per second to a single node. Consistent hashing spreads clients evenly but can't split one client's load. Ways to protect the other clients on that node:

- Reject early and cheaply. Once a client is over the limit, the gateway can remember "blocked until time T" (from the `Retry-After` value) in its own memory, so repeated rejected requests don't hit the node at all.
- Keep per-client work small and atomic (the Lua script already does this), so one client can't hold up others on the node.
- Edge-level limits (per IP) stop floods before they reach the gateway.

## Multi-region

Should the limit be global or per region? Keeping a single global count means every check crosses regions, which adds tens to hundreds of milliseconds. Common compromises: accept approximate global limits by syncing counters periodically, or give each region a share of the quota.

## Local counters with periodic sync

An alternative to the shared store: each gateway counts locally and syncs with others occasionally. It's faster and has no shared-store dependency, but it's approximate, so a client can exceed the limit between syncs. This is the main alternative design, so know the trade-off.

## Clock issues

Lazy refill depends on time. Which clock do you trust? The standard answer is to use the store's own clock (as the Lua script does), not each gateway's, and to guard against time appearing to go backward.

## Layered limits

Per IP, per user, per endpoint, and a global cap that protects the backend, all working together (see section 9).

## Client-facing details

- Standard headers such as `X-RateLimit-Remaining` tell clients how much budget is left.
- Well-behaved clients use exponential backoff with jitter after a 429, which spreads out retries instead of creating synchronized waves.

## Operations

Changing limits without a redeploy, per-customer overrides, and the metrics and alerts you'd want (rejection rate, limiter latency, fallback activations).

---

# 11. Glossary

| Term | Meaning |
|---|---|
| QPS / RPS | Queries (or requests) per second. QPS is often database-flavored, RPS more general. |
| Peak vs average | Systems are sized for peak load. Peak is an assumption you state, often 2 to 5x the average. |
| Request path (critical path) | The chain of components a request must pass through, or wait on, before the client gets a response. An in-path component adds latency to every request, and if it's down the request fails unless there's a fallback. Off the path means the request finishes without waiting for it (for example, analytics sent through a queue). |
| Latency budget | The time a component is allowed to add to each request. Here, a few milliseconds for the limiter. |
| Hop | One network call between components. Within a data center, about a millisecond or less. |
| Throughput vs capacity | Throughput is how much per second flows through (bytes/sec, requests/sec). Capacity is how much is held (bytes stored). Mixing them up is a common estimation slip. |
| Headroom | Extra capacity above the expected peak, such as running at half of a node's maximum. |
| Token bucket | A bucket of tokens that refills at a fixed rate. Each request spends one. Allows bursts up to the bucket's capacity. |
| Lazy refill | Computing a bucket's refill when it's next used, from the elapsed time, instead of running a timer per bucket. |
| Fail-open / fail-closed | What the limiter does when it breaks: allow requests (open) or reject them (closed). |
| Atomic | A multi-step operation that behaves as one indivisible step. |
| Idempotent | Repeating an operation gives the same result as doing it once. Not the same thing as atomic. |
| Lost update | Two concurrent read-modify-write operations where one overwrites the other. |
| Lua script (Redis) | A script that Redis runs as one uninterruptible unit, so it can safely do a read-modify-write. |
| Consistent hashing | Mapping keys to nodes on a ring so adding or removing a node moves only about 1/N of the keys. |
| Virtual nodes | Many points on the ring per physical node, which evens out the load and spreads a failed node's keys across many survivors. |
| Hot key | A single key (client) that receives a disproportionate share of traffic and overloads the node it lives on. |
| Signed token (JWT) | A token carrying the client ID plus a signature, which can be verified locally with no network call. |
| Salt | Random data mixed into a hash input to defeat precomputed lookup tables. Not encryption, and not needed for routing. |
| 429 / `Retry-After` | The HTTP status for "too many requests," and the header telling the client how long to wait. |
| Replica / failover | A copy of a node's data, promoted to primary if the primary fails. Asynchronous replication can lose the most recent writes. |
| Eventual consistency | Reads may briefly be stale after a write, and all copies converge over time. |

---

# 12. Interview tips and cheat sheets

## The framework

1. Clarify requirements (functional and non-functional)
2. Estimate scale (QPS, storage)
3. High-level design (the major components first, kept simple)
4. Deep dive (pick one or two components that matter and go deep)
5. Bottlenecks, trade-offs, scaling, and failure

## Habits that interviewers reward

- State assumptions out loud ("I'll assume reads are much heavier than writes, does that sound right?").
- Narrow scope deliberately, and say what you're leaving out.
- Commit to a choice, give the reason, and say what would make you switch.
- You don't need to ask about every possible feature. Stating "I'll assume X, and we can revisit it" shows good judgment. An interviewer may nudge you or may let you walk into a gap, so run your own checklist before leaving the clarifying step.
- If a new requirement is added mid-interview, adapt calmly. A design that can accommodate it is what matters.
- If you don't know a term, describe the mechanism and say "I don't remember the exact name for this." That's better than a confident wrong term, which invites a follow-up you can't answer.
- Answering in a few sentences per point beats long description. For each component, give one sentence for the choice and one for the reason.

## Questions to ask about every box in your design

1. What does it add to latency?
2. What happens when it's down?
3. Does it need to be in the request path at all?
4. Does this number measure throughput or capacity?
5. Does one machine hold this?
6. When the failure happens, where does this component live?

## Estimation cheat sheet

Units (powers of ten are fine for interviews):

| Unit | Bytes |
|---|---|
| KB | 10^3 |
| MB | 10^6 |
| GB | 10^9 |
| TB | 10^12 |

Quick facts and formulas:

- Seconds per day: 86,400 (about 100K for mental math). A month is about 2.5M seconds and a year about 31.5M.
- Average QPS = requests per day / 86,400. Peak is 2 to 5x the average, stated as an assumption.
- Storage = records x bytes per record x retention period.
- Node count = peak load / per-node capacity, then add headroom (about 2x) and replicas, and plan for next year's growth.
- Cache size = the working set you want resident (for example the hottest 20% under an 80/20 assumption) x bytes per entry. QPS x bytes is throughput, not cache size.
- Alphabet size comes from adding the character classes, not multiplying: 26 + 26 + 10 = 62 symbols. A code of length x has 62^x possibilities (62^6 is about 56.8 billion, 62^7 is about 3.5 trillion).
- Token bucket: refill rate = limit / window. One token every window / limit seconds. Worst case in a window is about capacity + refill x window. Retry-After = ceil((1 - tokens) / rate).
- Sanity checks after every number: "Does this seem plausible?" and "Does one machine hold it?" A 3-character code from 62 symbols gives only 238,000 combinations, so it couldn't cover billions of links.

## Study plan

- Mocks are a good way to learn, and they surface the right material. Add structured study right after each one: pick the 3 or 4 concepts that tripped you up and learn them properly.
- A rough split of 60% mocks and practice and 40% study fits a 4-6 week timeline.
- References: *Designing Data-Intensive Applications* by Martin Kleppmann (deep, read selectively) and *System Design Interview* by Alex Xu (lighter, interview-focused). You don't need to finish either before continuing mocks.
- Keep a running glossary of terms you stumbled on and review it before each mock.
- Your own experience helps: the queue-based pipeline in your artifact-update project, the polling overload on your package-manager service, and the release automation work are all real examples of patterns that come up here.
- Typing answers is different from speaking them under time pressure, so schedule at least one mock with a person and say your answers out loud.

---

# 13. Debrief

## The 5-minute version

A strong answer, using your decisions plus the fixes we covered. Practice saying it out loud.

> **Requirements.** Limit each authenticated client to 100 requests per minute across all endpoints. Rejections return 429 with a `Retry-After` header. Limits are fixed for now. Scale is about 1M requests per second at peak, with about 1M active clients per minute and the client base doubling next year. The check should add no more than a few milliseconds, and the limiter must not take the API down with it.
>
> **Estimates.** Peak is 1M req/sec, so ~2M next year. At about 50K checks/sec per node, that's roughly 20 nodes now and 40 next year, before headroom. State is tiny: about 100 bytes per client, so ~100 MB for the active set. The nodes exist for throughput, not memory.
>
> **Design.** The API gateway verifies a signed token locally to get the client ID, hashes it with consistent hashing (about 100 virtual nodes per node) to pick a counter node, and sends one atomic check. The node runs a token bucket with lazy refill as a single Redis Lua script: refill from elapsed time, compare, decrement, and return allow or reject along with the retry time. Allowed requests continue to the API service. Rejected requests get a 429 and don't consume tokens. Counter nodes have replicas, and adding a node moves about 1/N of the clients, who simply start with a fresh bucket.
>
> **Trade-offs and failure.** I chose token bucket because it permits normal bursts and needs only two values per client. The cost is no hard cap in any rolling minute, and a sliding window counter would fix that. If the shared store is unreachable, gateways fail open with a conservative local limit and page the team, because availability matters more than strictness. I'd also put per-IP limits at the edge.

## What went well

- **Your instincts were good.** You picked token bucket, decided on fail-open and defended it, wanted the limiter near the gateway for latency, and identified that a 429 shouldn't consume tokens.
- **You caught the consistency check on traffic.** About 1 request per second per client, under the limit, so most clients are well-behaved.
- **Node sizing was sound.** Per-node rate, divide into peak, adjust for the extra operation, with the reasoning shown.
- **Your lock idea was legitimate.** It's the first answer many engineers give, and it was the right problem to be thinking about.
- **You asked useful questions.** "What does request path mean?" and "what does the budget refer to?" are the right moves in a mock, where naming a gap is how you close it. In a real interview, the same thing works as "let me restate the requirement to make sure I have it right."

## What to tighten

### 1. Precision (the same theme as last time)

- State size was off by 1,000x again (10M bytes is 10 MB, not 10 GB).
- Token bucket parameters didn't match the requirement. Capacity 200 allows a 200-request burst, and a refill of 1 token per second is 60 per minute. State the burst you intend, then derive the numbers.
- The retry interval is 1 / rate (0.6 seconds), not rate / 1.

### 2. Vocabulary that was close but not exact

- Idempotent versus atomic.
- A salt isn't encryption, and the routing hash doesn't need one.
- "Fail-open costs more money" doesn't describe the real risk. The risk is no protection while the limiter is down, and the mitigation is a local fallback limit plus alerting.

### 3. Design-level patterns

- The coordinator sat in the request path for every request. Ask of each box, "what does this add to latency, and what happens when it's down?" Removing it (gateways compute the routing) was the simpler answer.
- Calling a token database per request would not survive 1M/sec. Signed tokens verified locally avoid that.
- Place the limiter by naming what information you have at that point, which is how the placement discussion works.

### 4. Delivery

- Your first answers were long on description and short on commitments. A good habit: for each component, one sentence stating the choice and one stating the reason.

## Open items

- **Hot clients:** one client sending thousands of requests per second to one node.
- **Multi-region:** whether limits are global or per region.
- **Local counters with periodic sync:** the main alternative design to the shared store.
- **Layered limits** (IP, user, endpoint) and client-facing details such as backoff.

## Compared with the URL shortener

Your structure was stronger this time. You committed to choices faster, treated the problem as a whole system instead of one component, and brought up failure early. The recurring weakness is arithmetic and units, so that's the cheapest improvement left to make.

## Suggested next steps

1. **Redo this from a blank page** in 2-3 days, timed at about 35 minutes, without looking at any notes. Then compare against the 5-minute version above.
2. **Make a one-page unit cheat sheet:** KB, MB, GB, TB in powers of ten; seconds per day (86,400); and your standard "sanity check" questions ("does one machine hold this?", "is this throughput or capacity?").
3. **Add one more problem** that stresses different muscles. A notification system would use queues, fan-out, and retries, and it connects to your GFS Hub experience. A chat system or news feed would also be good.
4. **Schedule a mock with a person.** Typing answers is different from speaking them under time pressure, and your Week 3 plan already includes it.
