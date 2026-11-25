# Notification System Mock Interview: Full Notes

These notes cover the whole mock interview: the prompt, the requirements, the estimation, the idempotency discussion, the high-level design with corrections, how workers fit in, extra topics worth studying, scale and failure, practice questions with strong answers, a glossary, cheat sheets, and the 5-minute version.

The interview ran through requirements, estimation, and the high-level design. The deep dive and the scale and failure steps were not run as questions, so sections 7 to 9 cover them for you as study material.

## Contents

1. The prompt and how the interview ran
2. Step 1: Requirements
3. Step 2: Estimation
4. Idempotency and duplicates
5. Step 3: High-level design
6. What a worker is
7. Topics to study in depth
8. Scale and failure
9. Practice questions with strong answers
10. Glossary
11. Cheat sheets and habits
12. The 5-minute version
13. Additions from the redo
14. Record states and how each is handled
15. Additions from the third run

---

# 1. The prompt and how the interview ran

**Prompt:** "Design a notification system. It should let services in our company send notifications to users through push notifications, email, and SMS."

The five steps:

1. Clarify requirements
2. Estimate scale
3. High-level design
4. Deep dive
5. Bottlenecks, scale, and failure

A real round is 45 to 60 minutes, which leaves roughly 35 to 50 minutes of design time. Typical budget: requirements 5 min, estimation 3 to 5 min, high-level design 10 min, deep dive 15 min, bottlenecks and failure 5 to 10 min. This mock was far deeper and slower than a real round, and a real interviewer usually probes one or two areas instead of every gap.

---

# 2. Step 1: Requirements

## What you asked, and what the interviewer answered

| Question | Interviewer's answer |
|---|---|
| Daily traffic | About 50 million notifications per day on average across all channels. Split by count: 70% push, 25% email, 5% SMS. |
| Burstiness | Traffic isn't steady. A big campaign or an alert can push the rate to several times the average for a short stretch. Later clarified as roughly 5 to 10x. |
| Services and users | About 100 internal publishing services. About 100 million registered users, about 20 million active on a typical day. |
| Failed deliveries | Retry transient failures with backoff, up to a limited number of attempts, then move to a dead letter queue for inspection and replay. Don't retry forever, and don't silently lose important notifications like password resets. |
| Acknowledgments | Users don't confirm anything. Record the delivery status the provider reports (sent, delivered, failed) where available. Read receipts are out of scope. |
| Publisher onboarding | Publishers are internal and trusted, and identify themselves with a service credential. A self-service portal is out of scope. |
| Per-publisher limits | Each service has a configurable quota. Over quota means the submission is rejected with a clear error, and the publisher backs off and retries. |
| Isolation | One service going over quota must not delay others. A low-priority bulk send must not hold up something time-sensitive like a password reset. |
| Provider limits | Push, email, and SMS providers enforce their own rate limits on us, separate from publisher quotas. |
| Notification sizes | Push about 1 KB, SMS about 0.5 KB, email about 20 KB (rendered HTML, no attachments), plus about 0.5 KB of metadata per notification. |
| Retention | Metadata for every notification is kept 90 days. Content is kept only the first 7 days. After day 7 the content is deleted and only metadata remains until day 90. |
| Delivery | Final delivery goes through third-party providers for push, email, and SMS. We don't build those ourselves. |

## What each answer changed in the design

| Requirement | Design consequence |
|---|---|
| Per-publisher quotas | A quota check at the gateway, using the same token bucket idea as the rate limiter |
| Isolation between publishers and priorities | Separate queues by priority, with dedicated worker pools |
| Different providers and limits per channel | Separate queues and workers per channel |
| Failed deliveries retried, then DLQ | Retry with backoff, a retry cap, and a dead letter queue with alerting |
| Publisher retries are possible | Idempotency key at intake (section 4) |
| Two retention windows | Content and metadata stored separately, each with its own TTL |
| Provider-reported status | A callback path that updates the record's status |
| Bursts of 5 to 10x | Queues absorb spikes, and workers drain at the providers' pace |

## Questions worth asking next time

You covered traffic, services and users, failures, acknowledgments, onboarding, quotas, and sizes. These are others to run through:

- Is there a priority difference between notification types (critical versus marketing)?
- Do we need scheduled or delayed notifications?
- Can a notification be sent to several channels at once, or does it fall back from one to another?
- Do users have preferences, opt-outs, or quiet hours?
- Do we need ordering guarantees per user?
- Are templates and localization in scope, or do publishers send finished content?
- Single region or multi-region?

You don't need to ask all of them. Pick two or three that would change the architecture and state assumptions for the rest.

---

# 3. Step 2: Estimation

## Volumes by channel (per day, average)

| Channel | Share | Notifications/day |
|---|---|---|
| Push | 70% | 35 million |
| Email | 25% | 12.5 million |
| SMS | 5% | 2.5 million |

The 70/25/5 split divides the number of notifications, not bytes. Applying it to bytes was one of the slips in this session.

## Rates (per second)

Average: 50M / 86,400 is about 580 per second in total.

| Channel | Average per second | At 10x peak |
|---|---|---|
| Push | about 405 | about 4,050 |
| Email | about 145 | about 1,450 |
| SMS | about 29 | about 290 (rounded to about 300) |
| Total | about 580 | about 5,800 |

You chose 10x so the design covers the worst case. The trade-off to name: sizing for 10x costs more hardware that sits idle most of the day, and the queue is what lets you run lower than 10x, since a short spike just waits in the queue.

## Peak: rate versus volume

This tripped things up, so it's worth being precise:

- 50 million per day is a **volume**. Doubling it gives a bigger day (100 million), which is not a peak.
- A peak is a **rate** (per second). You take the average rate and multiply it by a peak factor.
- "Several times the average" meant roughly 5 to 10x for a short burst. A general daily peak is often 2x. State which one you used.

## Bytes per day

Using your sizes (content plus 0.5 KB metadata): push 1.5 KB, email 20.5 KB, SMS 1 KB.

| Channel | Calculation | Per day |
|---|---|---|
| Push | 35M x 1.5 KB | about 52.5 GB |
| Email | 12.5M x 20.5 KB | about 256 GB |
| SMS | 2.5M x 1 KB | about 2.5 GB |
| Total | | about 311 GB/day |

Average throughput: 311 GB / 86,400 seconds is about 3.6 MB per second. At 10x that is about 36 MB per second.

An insight worth saying out loud: **push dominates by count, but email dominates by bytes** (about 80% of the daily bytes from 25% of the notifications).

## Storage with two retention windows

Keep content and metadata separate, since they have different lifetimes.

**Content (7 days)**

| Channel | Calculation | Per day |
|---|---|---|
| Push | 35M x 1 KB | 35 GB |
| Email | 12.5M x 20 KB | 250 GB |
| SMS | 2.5M x 0.5 KB | 1.25 GB |
| Total | | about 286 GB/day |

286 GB x 7 days is about 2 TB.

**Metadata (90 days)**

50M x 0.5 KB is 25 GB per day, and 25 GB x 90 days is about 2.25 TB.

**Total:** about 2 TB + 2.25 TB = about 4.25 TB.

Slips from this session to avoid:
- Multiplying the combined 311 GB/day by 7 counts the metadata for 7 days only, and mixes the two retention windows. Compute the pieces separately.
- Metadata at 50M x 0.5 KB x 90 is 2.25 TB, not 2,250 TB. After any storage number, ask whether it looks plausible next to the other figures.

## Replication and nodes

- A replication factor (RF) of 3 means three copies in total, each on a different node. The multiplier is 3, not 3 x 2.
- On disk: 4.25 TB x 3 is about 12.75 TB.
- At about 2 TB per node, 12.75 / 2 is about 6.4, so about 7 nodes.
- The 2 TB per node figure is an assumption. Why keep nodes modest: compaction needs spare disk space to work in, rebuilding a failed node streams all its data and takes longer the more it holds, and you want room for growth. Say what you assume and why.
- Cassandra has no master with replicas hanging off it. Any of the three replicas of a record can serve reads and writes.

## Why Cassandra, not just "big data"

4.25 TB would fit on one machine, so size alone isn't the reason. The reasons are write throughput (50M writes per day with spikes), availability (replicas across nodes), and built-in TTL for the two retention windows.

---

# 4. Idempotency and duplicates

## The problem

1. A publisher sends "send this password-reset email to user 123."
2. We accept it, store it, and queue it.
3. Our response is lost or arrives too late, so the publisher sees a timeout.
4. The publisher can't tell whether we received it, so the safe move is to retry.
5. If our service generates a fresh random ID for every request, the retry looks like a brand-new notification, and user 123 gets two emails.

Retries are normal, so we can't ask publishers never to retry. We must handle them.

## The solution: an idempotency key

The publisher generates a unique key for each logical notification and sends the same key on every retry. The service:

1. Looks up the key. If it was seen before, create and send nothing, and return the result of the first request.
2. If not seen, store the key and create the notification.

Details that matter:

- **Check and store must be one atomic step.** Two retries arriving at nearly the same moment would both pass a lookup-then-write. Use a conditional insert ("create only if this key doesn't exist"), such as a unique key on the record, or Redis `SET ... NX` with an expiry. Same pattern as the conditional write in the URL shortener and the atomic check in the rate limiter.
- **Keep keys for a limited time**, such as about 24 hours, longer than any realistic retry window.
- **The key has to come from the publisher.** Only the sender knows whether two requests are retries of one notification or two separate notifications. Guessing from content is wrong both ways: two legitimate identical messages get merged, and a retry with a different timestamp slips through. A random ID created by our service after the request arrives can't help.

## Two stages, two checks

| Stage | Question | Key | Prevents |
|---|---|---|---|
| Intake | Have I already accepted this request? | The publisher's idempotency key | Creating the same notification twice |
| Delivery | Have I already sent this to the provider? | The notification's own ID (UUID), plus the channel if one notification goes to several channels | Sending the same notification twice |

They form a chain. On the first request, the service stores the publisher's key, creates the notification, and assigns its own UUID. On a retry, the key is found and the service returns the same notification ID without creating anything. From then on, the pipeline works with the notification ID.

Why the worker uses the notification ID instead of the publisher's key: the publisher's key is only unique within that publisher and may be missing, one request can fan out to several channels that each need their own identity, and inside the pipeline you want an ID you generated yourself.

## Delivery guarantees

- Queues deliver at least once. If a worker crashes before acknowledging, the message is delivered again.
- At-least-once delivery plus idempotent handling gives an effect close to "exactly once."
- **The remaining gap:** a worker sends to the provider and crashes before recording that it did. On redelivery it sends again, unless the provider dedupes on an ID you pass. Some providers support that, and some don't. Where one doesn't, a rare duplicate is possible. Say that out loud, and for critical messages prefer providers that dedupe.

## Choosing the notification ID

- A timestamp alone is not unique. Two notifications created at the same instant collide.
- Hashing content plus timestamp fails for campaigns: the same message sent to five million users in the same second would produce the same hash. Adding the user ID fixes that case, but the same user can still get two identical notifications in the same second (a duplicate by mistake).
- A random **UUID v4** doesn't depend on content, and collisions are negligible at this scale. That's the choice you made. It then needs the idempotency key (above) to handle retries, because a random ID can't recognize a repeat.
- Use the same ID to link the content record and the metadata record.

---

# 5. Step 3: High-level design

## Your design

1. Publisher sends the notification to the service with a publisher key.
2. Service checks whether the message has been queued, and if not, sends an ack back to the publisher.
3. Service generates the message and a UUID, and stores metadata and content in Cassandra.
4. Service queues the message into Kafka or RabbitMQ, so it flows separately to the push, SMS, and email providers.
5. If a message is lost, it goes to a dead letter queue.
6. The service processes the dead letter queue and retries.

What's right: the publisher's idempotency key, a UUID per notification, a queue between intake and the providers, a Cassandra store, and a dead letter queue.

## What to fix

1. **Intake checks are missing.** Before the idempotency check, the gateway should authenticate the publisher's service credential and enforce its quota (the per-publisher token bucket from your rate limiter design). Over-quota requests are rejected there.
2. **The ack comes after the durable work.** Acknowledge the publisher only after the record is stored and the message is queued, so an ack means it won't be lost.
3. **The dual-write gap.** Writing to Cassandra and then to the queue are two separate steps. If the service crashes between them, the notification is stored but never sent. Fix: save the record as `PENDING` and have a small relay publish pending records to the queue, re-publishing any that get stuck (the outbox idea in section 7). The other fix is to write to the queue first and have a consumer do the persisting.
4. **What the DLQ is for.** A queue that is durable and acknowledged doesn't lose messages. A failed delivery is retried with exponential backoff, and only after the maximum number of attempts does it go to the DLQ. The DLQ is for inspection, alerting, and manual or tooled replay, not automatic endless retry. Also don't retry permanent failures (an unsubscribed user, an invalid device token).

## The reference design

```
Publisher
  -> API gateway: authenticate service credential, per-publisher quota
  -> Notification service: validate, idempotency check, assign UUID
       -> Cassandra: metadata (90-day TTL) and content (7-day TTL)
       -> queues, one set per channel and priority
  -> Channel workers (push / email / SMS)
       -> user preferences and contact details (cached)
       -> provider (push, email, SMS), throttled to its limits
       -> record status
  Provider callbacks -> update delivery status
  Failures -> retry with backoff -> after max attempts -> DLQ
```

Intake:

```
Publisher -> API gateway -> Notification service -> Queues
                                  |
                                  +--> Cassandra (metadata, content)
```

Delivery and failure:

```
Queues -> Channel workers -> Providers (push, email, SMS)
              |
              +--> Retry (exponential backoff) --> back to the Queues
                        |
                        +--> after max attempts --> Dead letter queue
```

## Request flow in words

1. The publisher sends a notification with its service credential and an idempotency key.
2. The gateway authenticates it and checks its quota. Over quota means rejection with a clear error.
3. The notification service validates the request and checks the idempotency key. A repeat returns the original result.
4. It assigns a UUID, writes the metadata and content to Cassandra, and puts the notification on the queue for its channel and priority.
5. It acknowledges the publisher.
6. A worker takes the message, checks it hasn't been sent and that the user hasn't opted out, looks up the contact details, and calls the provider within its rate limit.
7. The worker records the status. Provider callbacks update it later (delivered, bounced).
8. A failed send is retried with backoff. After the maximum attempts, it goes to the DLQ with an alert.

## Decisions worth saying out loud

- **Separate queues per channel.** Each channel has its own provider, limits, and failure modes, so an email provider outage doesn't slow down push.
- **Separate queues per priority.** Password resets and one-time codes sit in a critical queue, campaigns in a bulk queue. A five-million-user campaign can't delay a reset. This covers the isolation requirement.
- **Workers decide whether to send.** They check preferences (opt-outs, quiet hours) and look up the device token, email address, or phone number.
- **Provider limits.** Workers throttle to each provider's rate limit, with a circuit breaker when a provider is failing, and optionally a backup provider per channel.
- **Kafka or RabbitMQ.** At about 5,800 per second at peak, either works. Pick one and say why (section 7).
- **Idempotency at both stages.** The publisher's key stops duplicate creation, and the notification ID stops a duplicate send.

## Cassandra tables

Cassandra modeling is driven by the queries, so design one table per access pattern:

| Table | Key | Holds | TTL |
|---|---|---|---|
| Notifications by ID | `notification_id` | Metadata: publisher, user, channel, priority, status, attempts, timestamps | 90 days |
| Content by ID | `notification_id` | The content | 7 days |
| Notifications by user | `(user_id, time bucket)`, clustered by time descending | A user's history | 90 days |

Idempotency keys can live in a small table or in Redis with a 24-hour expiry.

---

# 6. What a worker is

A worker is a program that sits behind a queue, takes messages off it, and does the actual work of delivering each notification. The service that accepts a publisher's request doesn't send anything. It records the notification and puts a message on the queue. Workers are separate processes that do the sending.

## What one push worker does for each message

1. Takes a message from the push queue. The queue keeps it as "in progress" until the worker confirms it's done.
2. Checks whether it should send: has this notification ID already been sent, and has the user opted out or is it quiet hours?
3. Looks up where to send: the device token (or email address or phone number for the other channels), usually from a cache.
4. Calls the provider, staying within its rate limit.
5. Records the result by updating the notification's status.
6. Acknowledges the message so the queue removes it. If the send failed, it schedules a retry instead, and after too many attempts the message goes to the DLQ.

## Why it's a separate piece

- **Speed for publishers.** The service responds right after storing, because the slow part (talking to a provider) happens later.
- **Independent scaling.** Run as many workers as needed. When a campaign fills the queue, add workers without touching the API service.
- **Isolation.** Separate worker groups for push, email, and SMS, so one provider's problem only backs up that channel.
- **Safe failure.** If a worker crashes mid-message, it never acknowledged, so the queue gives the message to another worker. That's why the idempotency check matters.

Other names for the same thing: a **consumer** (in Kafka, a group of consumers shares a topic's partitions), a background worker, or a job processor. In practice it's one running copy of a small service, such as a container, and you run many copies at once.

---

# 7. Topics to study in depth

## Kafka versus RabbitMQ

| | Kafka | RabbitMQ |
|---|---|---|
| Model | A partitioned, durable log. Consumers track their position (offset). | Queues with per-message acknowledgment |
| Throughput | Very high | High enough for about 5,800/sec |
| Ordering | Guaranteed within a partition. Use `user_id` as the key for per-user order. | Per queue, weaker with several consumers |
| Replay | Native: rewind an offset to reprocess | Not native |
| Delays and retries | No per-message delay, so use retry topics with delays | TTL plus dead-letter exchanges give delays and retries easily |
| Priority | Use separate topics | Built-in priority queues |
| Parallelism | One consumer per partition within a group | Many consumers on one queue |

Either is defensible. Pick one, give a reason, and mention the cost. For example: "I'd use Kafka for durability and replay, and retry topics for backoff." Or: "RabbitMQ, because per-message acks and dead-letter exchanges map directly to our retry and DLQ needs."

## The outbox pattern (the dual-write problem)

Problem: saving to the database and publishing to the queue are two operations, and a crash between them loses the notification.

- **Outbox:** write the notification and an "outbox" entry in one atomic step, and a relay process reads the outbox, publishes to the queue, and marks it published.
- **In Cassandra**, which has no multi-table transactions, the same idea works with a `PENDING` status and a sweeper that republishes anything pending for too long. Republishing is safe because of the idempotency check downstream.
- **Alternative:** write to the durable queue first and have a consumer persist the record.

## Priority and isolation

- Separate queues (or topics) per priority, such as critical, normal, and bulk.
- Dedicated worker pools per priority, so bulk can't consume the capacity reserved for critical.
- If workers share a pool, use weighted consumption so bulk still makes progress but never starves critical.
- Per-publisher quotas at the gateway stop one service flooding the system, and a per-publisher rate in the bulk queue stops one campaign monopolizing it.
- Give time-sensitive notifications an expiry. A one-time code shouldn't be sent 30 minutes late.

## Provider details

- **Push:** goes through Apple's service for iOS and Google's for Android. Devices are identified by tokens that can expire. When a provider reports a token as invalid or unregistered, remove it and don't retry. Payloads are limited to a few KB.
- **Email:** bounces and spam complaints should feed a suppression list so you stop sending to those addresses. Sender authentication (SPF, DKIM, DMARC) and sending reputation matter for inbox placement.
- **SMS:** costs money per message, has length limits (long messages are split into segments), is subject to carrier regulations, and must honor opt-out keywords such as STOP. Best reserved for critical messages like one-time codes.
- **Provider limits and failures:** throttle to each provider's rate limit, honor their `Retry-After` responses, open a circuit breaker when error rates climb, and consider a second provider per channel to fail over to.

## User preferences, opt-outs, and templates

- Store per user: opted-in channels, quiet hours, frequency caps, and language. Workers cache these.
- Check preferences **at send time**, not just at enqueue time, since preferences can change while a notification waits.
- Unsubscribe requests must be honored. There are legal requirements around this.
- Templates: publishers can send a template ID plus parameters instead of finished content. A template service renders it (with localization). This keeps requests small and could reduce stored content, though in this exercise we stored the rendered content (about 20 KB per email).

## Cassandra specifics

- Model per query, one table per access pattern. Avoid secondary indexes.
- Avoid unbounded partitions. For per-user history, partition by user and a time bucket (such as the month).
- Write-heavy workloads suit Cassandra.
- With RF 3, reading and writing at QUORUM (2 of 3 replicas) is strongly consistent for that key (since 2 + 2 > 3) and tolerates one node being down.
- TTL expiry creates deletion markers (tombstones), so plan compaction and avoid designs that scan lots of expired data.
- Conditional writes (lightweight transactions) exist but are slower. Use them sparingly, or put idempotency keys in Redis.

## Retries and backoff

- Delay grows with each attempt: `delay = min(cap, base x 2^attempt)`, plus random jitter so retries don't all land together.
- Cap the attempts (about 5), then send to the DLQ and alert.
- Retry **transient** errors (timeouts, server errors, rate-limit responses). Don't retry **permanent** ones (invalid token, unsubscribed user, malformed request).
- Honor a provider's `Retry-After`.
- Messages can expire, so a late retry of a time-sensitive notification is dropped.

## Delivery status

- States: `PENDING`, `QUEUED`, `SENT`, `DELIVERED`, `FAILED`.
- Providers report status through callbacks (webhooks) or polling. Store the provider's message ID so a callback can be matched.
- Callbacks can arrive late, repeated, or out of order. Make updates idempotent and only move a status forward.

## Campaigns (large fan-out)

A request for five million users shouldn't become five million messages instantly.

- A campaign service expands the audience in batches and feeds them into the bulk queue at a controlled rate.
- Throttle to stay within provider limits and leave room for critical traffic.
- Support pause and cancel.
- Use (campaign, user) as the idempotency key so a restarted expansion doesn't resend.

## Ordering

Ordering isn't guaranteed across channels or workers by default. If per-user order matters, key the queue by `user_id` (in Kafka, so one user's messages stay in one partition). Many notifications don't need ordering, and saying so is fine.

## Monitoring

Watch queue depth and consumer lag, end-to-end delivery latency, send success rate per provider, retry rate, DLQ size, and quota rejections per publisher. Alert on a growing DLQ, a failing provider, and rising lag.

## Security and privacy

Encrypt data in transit and at rest, avoid logging sensitive content (one-time codes, personal data), restrict who can replay from the DLQ, and enforce the retention windows.

---

# 8. Scale and failure

- **At 10x:** add queue partitions, workers, and Cassandra nodes. The binding limit will probably be the providers' rate limits, not your own fleet, so build in backpressure. The queue absorbs bursts: bulk waits while critical goes through.
- **Autoscaling:** scale workers on queue lag.
- **Storage** (from the estimate): about 4.25 TB, about 12.75 TB with three copies, roughly 7 nodes at 2 TB each.
- **A worker dies:** its unacknowledged messages are redelivered to another worker. That's safe because of the idempotency check.
- **A Cassandra node dies:** the other replicas serve its data (with QUORUM, one node down is tolerated), and a new node is rebuilt from the replicas.
- **A provider goes down:** the circuit breaker opens, messages wait in the queue or switch to a backup provider, and retries use backoff. Alert someone.
- **The queue itself is unavailable:** the service can't hand off, so it keeps the record as `PENDING` and the relay republishes when the queue returns (or rejects new requests if it must, so publishers retry later).
- **A region goes down:** traffic fails over to a second region with a replicated queue and database. Some duplicates are possible after failover, and the idempotency keys limit them. Replication is asynchronous, so a very recent notification might not have replicated yet, and you state whether you accept that.

---

# 9. Practice questions with strong answers

Try answering each out loud before reading.

**How do you stop a publisher's retry from sending a duplicate?**
The publisher sends an idempotency key. We check and record it with one atomic conditional insert, and a repeat returns the original result. The key is kept about 24 hours.

**How do you stop a worker's retry from sending a duplicate?**
The notification ID is the key for the send. The worker checks the record's status before sending, or passes the ID to a provider that dedupes. There's a small residual window if a worker crashes after sending and before recording, which I'd accept or close with a provider that supports idempotency.

**What happens if the service crashes between saving and queueing?**
Save the record as `PENDING`, and a relay republishes anything pending too long. Republishing is safe because downstream is idempotent.

**How do you keep a bulk campaign from delaying a password reset?**
Separate queues and worker pools by priority, with reserved capacity for critical. A campaign service feeds the bulk queue at a controlled rate, and per-publisher quotas at the gateway limit each service.

**What happens when a provider is down?**
Sends fail and are retried with backoff and jitter. A circuit breaker stops hammering the provider, messages wait in the queue (which absorbs the backlog), and a backup provider can take over. The team gets an alert.

**What does the DLQ do?**
It holds messages that exhausted their retries, so people can inspect them and replay once the cause is fixed. It triggers an alert. It isn't an automatic infinite retry loop.

**How do you know a notification was delivered?**
We record the provider's callbacks (sent, delivered, bounced) and update the status, matching them with the provider's message ID. Callbacks can arrive late or out of order, so updates only move forward.

**How do you handle an unsubscribed user?**
Workers check preferences at send time and skip the send. Permanent failures like an unsubscribed user or invalid token are not retried.

**How would you store this and why Cassandra?**
Write-heavy volume, availability through replicas, and built-in TTLs for the 7-day and 90-day windows. Content and metadata live in separate tables, linked by notification ID, with a user-history table keyed by user and time bucket.

**What breaks first at 10x?**
Probably the providers' rate limits. Add queue partitions, workers, and nodes, apply backpressure, and let the queue absorb bursts.

**An entire region goes down. What happens?**
Traffic fails over to the second region, which has replicated data and queues. Recent notifications may not have replicated, duplicates are possible, and idempotency keys limit them. Retries come from outside the failed region, with backoff and jitter.

---

# 10. Glossary

| Term | Meaning |
|---|---|
| Idempotency key | A unique key the publisher sends so repeats of the same request can be recognized and not processed twice |
| UUID v4 | A randomly generated 128-bit identifier. Collisions are negligible at this scale. |
| At-least-once delivery | A queue guarantee that a message is never lost but may be delivered more than once |
| Dead letter queue (DLQ) | Where messages go after exhausting their retries, for inspection and replay |
| Worker / consumer | A process that takes messages from a queue and does the work |
| Fan-out | One request that becomes many messages (a campaign to millions of users) |
| Backpressure | Slowing or pausing intake when downstream can't keep up |
| Exponential backoff | Waiting longer after each failed attempt |
| Jitter | Random variation added to a delay so retries don't synchronize |
| Circuit breaker | Stops calling a failing dependency for a while, then tests it again |
| Outbox pattern | Recording intent to publish together with the data, so a relay can reliably publish later |
| Replication factor (RF) | The number of copies of each record (3 means three copies in total) |
| Quorum | A majority of replicas (2 of 3) that must acknowledge a read or write |
| TTL | Time to live: the record is deleted automatically after a set time |
| Tombstone | A deletion marker in Cassandra, created by deletes and TTL expiry |
| Compaction | Cassandra's background merge of data files, which needs spare disk space |
| Consumer lag | How far behind a consumer is on a queue. A key health metric. |
| Device token | A push provider's identifier for one app on one device |
| Suppression list | Addresses we've stopped sending to after bounces, complaints, or unsubscribes |
| Throughput vs capacity | Throughput is how much flows per second. Capacity is how much is stored or can be held. |

---

# 11. Cheat sheets and habits

## Formulas used in this problem

- Average rate = volume per day / 86,400.
- Peak rate = average rate x peak factor (state it: 2x for a typical daily peak, 5 to 10x for bursts).
- Percent splits apply to counts unless the numbers you were given say otherwise.
- Bytes per day = notifications per day x size per notification, by channel.
- Storage = bytes per day x retention days. Do each retention window separately.
- Disk with replication = storage x RF.
- Nodes = disk with replication / per-node size, then add headroom.

## Units

KB = 10^3, MB = 10^6, GB = 10^9, TB = 10^12 bytes. Seconds per day is 86,400 (about 100K for mental math). After any number, ask "does this look plausible next to the others?" and "does one machine hold it?"

## Habits that interviewers reward

- State assumptions out loud, and say what a multiplier is applied to.
- Commit to a choice, give the reason, and say what would make you switch.
- Ask of each component: what does it add to latency, and what happens when it's down?
- If you don't know a term, describe the mechanism and say you don't remember the name.
- Notice duplicates, retries, and failures early. They're what separate a sketch from a design.

## Slips from this session to watch for

1. Applying 2x to a daily volume instead of a rate.
2. Applying the 70/25/5 split to bytes instead of counts.
3. Combining content and metadata into one retention window.
4. Metadata arithmetic off by 1,000x (2.25 TB, not 2,250 TB).
5. Replication multiplied by 3 x 2 instead of 3.
6. Linking records by timestamp (not unique) or by a content hash (identical campaigns collide).
7. Missing the publisher's idempotency key as the answer to retries.
8. Treating the DLQ as an automatic retry loop instead of an inspection and replay queue.
9. Forgetting the gateway checks (authentication, quota) and acknowledging the publisher before the durable work.

## What went well

- Thorough requirements, including per-publisher limits and sizes.
- Splitting content and metadata, with different TTLs, and choosing Cassandra for the right reasons (throughput and availability, not size).
- Choosing a random UUID, and understanding that content-based IDs would collide.
- A clear first design with intake, queue, providers, and a DLQ.

---

# 12. The 5-minute version

> Publishers send notifications through a gateway that authenticates them and enforces per-publisher quotas. The notification service validates the request, checks the publisher's idempotency key, assigns a UUID, stores the record in Cassandra (metadata for 90 days, content for 7), and puts the notification on a queue chosen by channel and priority. It acknowledges the publisher after the durable work. Channel workers check the user's preferences, look up contact details, and send through the provider, throttled to its limits, and record the status from the provider's callbacks. Failures retry with exponential backoff and jitter, and after the maximum attempts they go to a dead letter queue for inspection and replay. The notification ID is the idempotency key for the send, so retries and redelivery don't duplicate. At 10x we add partitions, workers, and nodes, and provider limits become the constraint.
>
> **Numbers:** about 50M per day is about 580 per second on average and about 5,800 per second at a 10x peak. About 311 GB per day in total, about 2 TB of content for 7 days plus about 2.25 TB of metadata for 90 days, so about 4.25 TB, or about 12.75 TB with three copies (roughly 7 nodes).

---

# 13. Additions from the redo

This section adds what came up in the second run of this interview. Where it differs from an earlier section, treat this section as the more specific version.

## Requirements the interviewer gave in the redo

| Question | Interviewer's answer |
|---|---|
| Priority | Yes. Password resets and one-time codes are time-sensitive, marketing can wait, and a bulk send must not hold up a critical one. |
| User preferences | Users can opt in or out per channel and per category (marketing versus account alerts). Assume an existing preferences service to read from. |
| Unsubscribed users | Skip the send, record that it was skipped, and don't retry. Security and account messages aren't subject to marketing opt-outs. |
| Multi-region | Single region for the core design, with a region failure asked in the last step |
| Providers | External third-party APIs per channel, each with its own rate limit, able to be slow or fail, and reporting delivery status back |
| When to send | As soon as possible. Critical within a few seconds, bulk can take longer. Scheduled or delayed sends are out of scope. |
| The DLQ | A message lands there only after transient failures exhaust their retries. Permanent failures (invalid token, unsubscribed user) aren't retried and don't need the DLQ. An alert names the publisher, notification ID, channel, recipient, error, and attempt count, without personal data in the text, and critical alerts go out faster. A person fixes the cause and replays. |

## Estimation numbers from the redo

- Average rates by count: about 406 push, 145 email, and 29 SMS per second (about 580 total).
- At a 5x peak: about 2,030 push, 725 email, and 145 SMS per second (about 2,900 total). A campaign is a good reason for 5x. Say what you chose and why.
- Average throughput with metadata included (push 1.5 KB, email 20.5 KB, SMS 1 KB): 406 x 1.5 + 145 x 20.5 + 29 x 1 is about 3,610 KB/s, or about 3.6 MB/s.
- Storage: about 2.25 TB of metadata, plus about 2 TB of content, about 4.25 TB total, about 12.75 TB with RF 3, about 7 nodes at 2 TB each.
- Slips to avoid: attaching the wrong channel label to a rate (push versus SMS), and leaving the metadata off one channel's size. Check that every size in a sum is measured the same way.

## Intake crash points

Three writes happen at intake: claim the idempotency key, save the record, put the message on the queue.

| Crash after | State | What the publisher's retry finds |
|---|---|---|
| Claiming the key, before saving the record | Key exists with the notification's UUID, no record | The key. The service reads the UUID, then saves the record and queues it. Both steps are safe to repeat because they're keyed by that UUID. |
| Saving the record as `PENDING`, before queueing | Record exists, nothing queued | The key. A sweeper also finds the stuck `PENDING` record and queues it. |
| Queueing, before marking `QUEUED` | The record still says `PENDING`, but the message is queued | The key. The sweeper may queue it again, and a duplicate message is safe because the worker checks the notification ID before sending. |

The rule is that a key being present always means the notification exists or will exist.

## Where to keep the idempotency key

- Redis for the key and Cassandra for the record are two separate systems, so no single operation covers both and a crash between them leaves them out of step.
- Writing the record first and checking the key second fails for concurrent duplicates: two requests with the same key both create a record, and you end up with two notifications.
- Putting the key in the same store as the record and claiming it first avoids both problems.
- In Cassandra, the mechanism is a **lightweight transaction** (conditional write). Cassandra has no server-side scripts:

```
INSERT INTO idempotency_keys (publisher_id, idempotency_key, notification_id)
VALUES (?, ?, ?) IF NOT EXISTS;
```

It reports whether the insert was applied, and if the key already existed it returns the existing row, including the notification ID. The primary key is `(publisher_id, idempotency_key)`.

- Cost: it runs a consensus protocol among replicas, so it takes several round trips and contends on the same key. That's acceptable once per incoming notification (about 580 per second on average, about 2,900 at peak), but not for every step.
- If you don't remember the name in an interview, say "a conditional insert that only succeeds if the key doesn't exist, which costs more than a normal write."

## What causes a retry

- The worker sends, the send fails with a transient error, and the worker updates the retry count on the record and requeues the message with a **delay** (a delay queue, a retry topic, or a visibility timeout). The message can't be processed until the backoff time has passed.
- Delay grows exponentially with random jitter. Cap the attempts. After the cap, mark the record `FAILED` and send it to the DLQ with an alert.
- Don't retry permanent errors.
- Give time-sensitive messages a delivery TTL, so a one-time code that has waited too long is dropped instead of sent.

## A provider outage

- Don't retry each failed message against a dead provider. Use a **circuit breaker** with three states: closed (sending normally), open (the provider is failing, so stop sending and let messages wait in the queue), and half-open (after a pause, let a few messages through to test it).
- Waiting messages stay in the queue, and their TTLs still apply.
- On recovery, ramp up gradually: let a few through, open fully only when they succeed. Send critical messages first.
- Throttle to the provider's quota, and requeue anything over it.
- If the preferences service is down, decide by message type: for marketing don't send, and for security and account messages send anyway.

## The campaign expansion job

- **Definition:** a campaign request contains one message and an audience (a list or segment, such as 5 million users). The expansion job is a background process that turns it into one notification per user. The publisher gets its 202 right away and the job runs for minutes or hours. It's a separate component from the API service and the workers (called the campaign service in section 7).
- **Batching:** read the audience in batches (for example 1,000 users), create the notifications, mark them bulk, and feed them to the bulk queue at a controlled rate.
- **No duplicates:** the (campaign, user) idempotency key, claimed with a conditional insert.
- **Resuming:** store a checkpoint per campaign: a snapshot of the audience taken at the start, and the last fully completed batch (or the last user ID). Process a batch, then advance the checkpoint. A batch in progress at a crash is run again from its start, and it's safe because the key skips users already handled. Don't scan all records on restart.
- **Stable audience:** snapshot the audience at the start, split it into numbered batches, and page by a stable cursor. Re-querying a live list lets batch boundaries shift, which skips or doubles users. State your policy for users who join after the snapshot.
- Parallel expansion workers claim a batch with a lease, and the key keeps it safe even if two take the same batch.
- Check the campaign's status between batches so it can be paused or cancelled.

## The feed rate

- Messages divided by a per-second quota gives a **duration**, not a rate. 5,000,000 / 3,000 per second is about 1,667 seconds, the time at the full quota.
- A feed rate is a **share of the provider's quota** that you choose. If the provider allows 3,000/s and normal plus critical traffic peaks at about 2,000/s, about 1,000/s is left. Don't give a campaign all of it: retries also use quota, critical traffic can spike, and you need room to back off.
- At about 500/s, 5,000,000 notifications take 10,000 seconds, about 2.8 hours. At 1,000/s, about 1.4 hours.
- Better: make bulk adaptive, with a token bucket set to the provider quota minus live critical and normal usage, so the campaign speeds up when other traffic is quiet and slows when it's busy.

## Scale and failure with numbers

- **10x:** about 5,800/s on average and about 29,000/s at a 5x peak. Storage grows to about 42.5 TB, about 127.5 TB with three copies, about 64 nodes at 2 TB each.
- **What breaks first:** the providers' rate limits, since you can't add servers to a third party. Get higher limits, spread across more provider accounts or vendors, and let the queues absorb the spike. After that, watch the idempotency write, which is a conditional (slower) write performed once per incoming notification.
- **Region failure:** a global load balancer sends traffic to the second region. Cassandra replicates across regions, and the queues are mirrored or rebuilt by replaying records still `PENDING` or `QUEUED`. Publishers retry from outside the failed region with backoff and jitter. Replication is asynchronous, so the last few seconds of records and keys may be missing, which means a small chance of a duplicate send. Limit it with provider dedupe IDs and say you accept it, or pay for stronger replication with higher write latency. Replay critical notifications first.

## Slips from the redo to watch for

1. Putting the wrong channel label on a rate, and leaving metadata off one channel's size.
2. Defining feed rate as messages divided by quota (that's a duration), and giving a campaign the whole remaining quota.
3. "Make it atomic" without saying how. Name the mechanism: a conditional insert in the same store.
4. Writing the record before checking the key, which lets concurrent duplicates create two records.
5. "Queues halt" during a provider outage. The queue keeps the messages, and the circuit breaker stops the sending.
6. Saying "I don't know" instead of describing what you'd need. In an interview, say what you'd want ("a conditional insert, I don't remember what Cassandra calls it") and ask for a hint.

---

# 14. Record states and how each is handled

The state names are a design choice. In an interview, 5 or 6 states with clear transitions is enough.

## The states

| State | Meaning | Set by | How it's handled |
|---|---|---|---|
| `PENDING` | Record saved, not yet confirmed as queued | Notification service at intake | A sweeper finds records stuck here past a threshold (say a minute), republishes them, and marks them `QUEUED`. A duplicate message is safe because workers check the notification ID. |
| `QUEUED` | On a queue, waiting for a worker | Service (or the sweeper) | A worker takes it. If it waits too long, queue-lag alerts fire, and a delivery TTL drops stale messages. |
| `SENT` | The provider accepted the request (not the same as delivered) | Worker, after the provider call succeeds | Wait for the provider's callback. Email and SMS may never report back, so decide what a missing callback means. |
| `DELIVERED` | The provider confirmed delivery | Callback handler | Terminal. |
| `FAILED` | Retries exhausted, or a permanent failure | Worker | Exhausted retries go to the DLQ with an alert, and a person fixes the cause and replays. Permanent failures (invalid token, bounced address) stop here, with no retries and no DLQ. |
| `SKIPPED` | The user opted out or unsubscribed | Worker, after the preference check | Terminal. Record why, and don't retry. Security and account messages aren't subject to marketing opt-outs. |
| `EXPIRED` | The delivery TTL passed before it could be sent | Worker | Terminal. A one-time code that waited 30 minutes is dropped, not sent. |

A failed attempt that will be retried doesn't need its own state. The record stays `QUEUED` with the attempt count, the last error, and the time of the next attempt, and the message is requeued with a backoff delay. If you prefer, call it `RETRYING`.

## How they connect

```
claim key + save record
         |
      PENDING --(sweeper, if stuck)--> QUEUED
         |  queued                        ^
         v                                | replay from DLQ
      QUEUED --> worker takes it          |
         |------> SKIPPED  (opt-out)      |
         |------> EXPIRED  (TTL passed)   |
         |------> transient failure --> backoff --> QUEUED (attempts + 1)
         |                                  after max attempts --> FAILED --> DLQ
         |------> SENT --> callback --> DELIVERED
                              \--> FAILED (bounced)
```

## Rules that apply to every state

- **Status only moves forward.** Callbacks can arrive late, repeated, or out of order, so a late `SENT` update must never overwrite `DELIVERED`. Use a conditional update (only change the status if the current one is earlier), or store the timestamp of each event.
- **Every update is idempotent.** Applying the same transition twice does no harm, since everything is keyed by the notification ID.
- **Store details beside the status:** attempt count, next attempt time, last error, the provider's message ID (to match callbacks), and timestamps.
- **Terminal states are `DELIVERED`, `FAILED`, `SKIPPED`, and `EXPIRED`.** Whatever the state, the content is deleted at 7 days and the metadata at 90 days by TTL.

## What handles stuck records

| Stuck in | What catches it |
|---|---|
| `PENDING` | The sweeper republishes it |
| `QUEUED` for too long | Lag alerts, and the delivery TTL for time-sensitive messages |
| `SENT` with no callback | A timeout policy you choose (leave it, or mark it unknown) |
| `FAILED` | The DLQ alert, then manual replay, which sets it back to `QUEUED` |

---

# 15. Additions from the third run

This section adds what came up in the third run of this interview. Where it differs from an earlier section, treat this section as the more specific version.

## How campaigns and channels work

- A publisher's request names the **audience** (or a single user), the **message**, and the **channels it may use**. A campaign is typically sent on one channel (an email promotion) or two (push plus email), not on everything.
- Preferences are applied **per user, per requested channel**: the system sends only on channels the user has opted into for that category. A user who opted out of every requested channel gets nothing, and the skip is recorded.
- Each user and channel pair is its own notification, with its own record, provider call, and retries. The 50 million a day and the 70/25/5 split already include this fan-out.
- Automatic fallback from one channel to another (SMS if push fails) was out of scope. Each channel is independent.
- Say where the expansion of one campaign request into per-user notifications happens (the expansion job from section 13).

## Per-channel peaks, and why they matter

- At 5x: about 2,030 push, 725 email, and 145 SMS per second, adding up to about 2,900.
- If every channel is multiplied by the same factor, the total and the storage don't change. Per-channel numbers still matter because each channel has its own provider and rate limit, its own queue, and its own worker pool, and channels don't always peak together (a marketing email blast spikes email while SMS stays flat). A flat 5x per channel is a simplification, so say it's an assumption.

## Email dominates the storage

| | Share of notifications | Share of the 7-day content | Share of everything stored |
|---|---|---|---|
| Push | 70% | about 12% | about 43% |
| Email | 25% | about 87% | about 54% |
| SMS | 5% | about 0.4% | about 3% |

Email is a quarter of the traffic and about 87% of the content storage. Over everything stored it's about half, because the 90-day metadata is the same size for every notification, so push (the most notifications) carries most of that. At this size, one design for all three channels is workable. If you did handle email differently, it would be in the content store (for example keeping a template ID and parameters instead of rendered HTML), at the cost of rendering again when you need the content.

## Keys, records, and the 202

- The **idempotency keys** expire after **24 hours**, longer than any realistic retry window.
- The **notification records** follow their own retention: 90 days for metadata and 7 days for content. Keep the key inside the notification record so the two are linked.
- Create the **notification ID at intake**, not later in a worker. The 202 returns the stored metadata and that ID, so the publisher has something to identify its notification by, and a retry that finds the key can return the same thing. The same ID is used all the way to the provider as the dedupe key.
- A retry within 24 hours finds the key and gets the original record back. A retry after the key has expired would create a new notification, which is the trade-off of choosing a TTL.
- The 202 is returned only after the record is written, so "accepted" means it won't be lost.

## Finding records that never got queued

A query for `WHERE status = 'PENDING'` doesn't work well in Cassandra, because `status` isn't part of the key. It means scanning, or a secondary index on a low-cardinality column that changes constantly, which is an anti-pattern. Two fixes that work together:

1. **Keep the poller out of the main path.** After writing the record, the service publishes to the queue right away and marks it `QUEUED`. Critical notifications wait on nothing. Pending records are the exception.
2. **A sweeper table as a safety net.** When a record is created, also insert a row into a table partitioned by a time bucket (for example the minute), with the notification ID as the clustering column. When the message is queued, delete or expire the row. The sweeper reads buckets older than a short threshold and republishes whatever is left. That's a read of a known partition, not a scan. A one-minute bucket holds about 35,000 rows at the average rate (about 174,000 at peak), so add a shard suffix or split by priority.

Use a threshold of seconds for critical and longer for bulk. Republishing is safe because delivery is idempotent on the notification ID.

## The bulk lane and the provider's quota

- Critical and bulk have **separate queues and worker pools**, so a password reset doesn't wait behind a 5-million-user campaign.
- Both lanes call the same provider, which has one rate limit across everything we send. So cap the bulk lane at a share of the provider's quota (50% is a reasonable starting point) so critical traffic always has room. Better: make it adaptive, the provider quota minus live critical and normal usage.
- With 40 bulk workers that can't see each other, send them all through a **shared limiter** (a token bucket in front of the provider), the same idea as the rate limiter design, applied to the provider instead of a user.
- A "no, not now" from the limiter is **not a failed send**. Don't count it against the 5 attempts, or a campaign waiting for quota could push thousands of healthy messages into the DLQ. Have the limiter return a wait time, and have the worker hold the message until then.

## A worker crash after the provider accepts

1. A worker sends a push, the provider accepts, and the worker crashes before recording `SENT`.
2. The message was never acknowledged, so the queue redelivers it to another worker after a timeout.
3. The retry uses the same notification ID, so a provider that dedupes on an ID recognizes the repeat.
4. If the provider doesn't support dedupe, a rare duplicate is possible. Say so, and accept it for most notifications.
5. It matters more for things that are costly or confusing to send twice (an SMS is billed per message, or a "you've been charged" message). For those, prefer a provider that dedupes on an ID, or shrink the window by recording that you're about to send before you call the provider.

An idempotency key prevents duplicate **creation**. It doesn't make delivery "at least once." The queue gives at-least-once delivery, and the key plus the notification ID turn that into an effectively-once result.

## Scale at 10x

- About 5,800 per second on average and about 29,000 per second at a 5x peak (500 million a day).
- Storage: about 22.5 TB of metadata plus about 20 TB of content, about 42.5 TB, about 127.5 TB with three copies, about 64 nodes at 2 TB each.
- The **provider limits** break first, since you can't add servers to a third party. Scale workers and Cassandra, and ask the providers for higher limits.
- The conditional write for the idempotency key now runs about 29,000 times per second at peak and is slower than a normal write. Watch it.

## Region failure

- Replicate Cassandra to a second region, with service nodes standing by there, and switch traffic on failure.
- The idempotency keys live in Cassandra, so replication carries them too. The gap is **replication lag**: cross-region replication is asynchronous, so records and keys written in the last moments before the failure might not have arrived. A conditional write can only reject a duplicate for a key the second region already has, so for those few keys a publisher's retry could create a second notification. Say it as a known trade-off.
- Notifications that were in the failed region's queues are recovered from the database: records still saying `QUEUED` that never moved on. Use a sweeper with a threshold by priority (seconds to a minute for critical, longer for bulk, not an hour), and run a recovery pass over those records **as soon as traffic switches**.
- Some of them may have been sent just before the crash without the status being updated, so provider-side dedupe covers it, and a rare duplicate is the accepted cost.

## What a real round looks like

- It's **time-boxed**: usually 45 to 60 minutes in total, with the design itself about 35 to 50 minutes.
- The deep dive is short: the interviewer picks one or two areas and spends about 10 to 15 minutes on them, usually a handful of questions, not chain after chain.
- A reasonable answer gets a move to the next topic, and if you're stuck, they give a hint or move on.
- You can steer: "I've covered the main flow, so would you like me to go deeper on retries or on campaigns?"
- The same building blocks keep coming back. Across the four problems so far: idempotency keys, queue plus workers, consistent hashing, retries with backoff, conditional writes, caching, and a replicated key-value store. A new problem is mostly a recombination of those.
- A real round expects clear requirements, a sensible estimate, a clean design, and the main trade-offs in your own words. "A conditional insert that only succeeds if the key doesn't exist" is a full answer without the Cassandra name.

## Slips from the third run to watch for

1. Arithmetic under pressure (580 x 5). Write the multiplication out.
2. Saying an idempotency key ensures "at least once." It prevents duplicate creation, and the queue provides at-least-once delivery.
3. Creating the notification ID in a worker instead of at intake.
4. Polling on a column that isn't part of the Cassandra key.
5. Counting a limiter's "wait" against the retry limit.
6. Saying the new region "won't have the idempotency keys." They replicate, and the gap is replication lag.
7. A sweeper threshold of an hour for everything, instead of by priority.
