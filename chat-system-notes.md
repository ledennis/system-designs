# Chat System Mock Interview: Full Notes

These notes cover the whole mock interview: the prompt, the requirements, the estimation (including the 10x math you ran out of time for), the high-level design with corrections, how Cassandra partitioning and clustering work, how real-time delivery and the user-to-server lookup work, ordering and catch-up, the chat server crash deep dive, scale and region failure, more topics to study, practice questions with strong answers, a glossary, cheat sheets, and the 5-minute version.

## Contents

1. The prompt and how the interview ran
2. Step 1: Requirements
3. Step 2: Estimation
4. Step 3: High-level design
5. Cassandra: partitioning and clustering
6. Real-time delivery: connections and the user-to-server lookup
7. Ordering, retries, and catch-up
8. Step 4: Deep dive, a chat server crash
9. Step 5: Scale and failure
10. More topics to study
11. Practice questions with strong answers
12. Glossary
13. Cheat sheets and slips to watch for
14. The 5-minute version

---

# 1. The prompt and how the interview ran

**Prompt:** "Design a chat system like WhatsApp or Slack. Users can send messages to each other one-on-one and in groups, and they see the messages in real time."

The five steps:

1. Clarify requirements
2. Estimate scale
3. High-level design
4. Deep dive
5. Bottlenecks, scale, and failure

This session had a 40-minute limit. In a real round with that much time, aim for about 5 minutes on requirements, about 5 on estimation (round aggressively), about 10 on the high-level design, about 10 on one deep dive, and about 5 on scale and failure.

---

# 2. Step 1: Requirements

## What you asked, and what the interviewer answered

| Question | Interviewer's answer |
|---|---|
| Users | About 100 million registered, about 20 million active on a typical day, about 5 million connected at once at the busiest time |
| Daily traffic | About 1 billion messages per day. About 80% one-on-one and 20% in groups. The busiest hours run several times the daily average. |
| Offline | Yes. A message to an offline user is stored and delivered when they reconnect. A push notification service exists for alerting offline users (call it, don't design it). History is kept 1 year, and a user can load it on a new device. |
| Accounts | Every user has an account and is authenticated. Authentication is out of scope, and each connection arrives with a verified user ID. |
| Pictures and video | Out of scope. Messages are text. If time allows, show where attachments would fit. |
| A message that fails to send | The sender sees its state: sending, sent, or failed. A failed message shows an error and a retry option. A retry must not create duplicates, and once the sender sees "sent," the message must not be lost. Delivered and read receipts are out of scope. |
| Limits per user | A per-user send-rate limit to stop spam. The rate limiter from the earlier problem handles it (say where it sits). Messages are capped at 4,000 characters. |
| Preferences | Friends lists and alert sounds are out of scope. An online or offline indicator is a nice-to-have. |
| Group size | Up to 500 members |
| Groups of groups | No. A group contains individual users only. |
| Friendship | Not required. Any user can message any other user. Blocking and contacts are out of scope. |

Two non-negotiables: within a conversation everyone sees the same order, and delivery to an online user feels instant (under about half a second).

## What each answer changed in the design

| Requirement | Design consequence |
|---|---|
| Real-time, under half a second | Persistent connections (WebSockets), not polling |
| Offline users, 1-year history | Store every message durably first. Offline delivery becomes a read of the history. |
| Same order for everyone | A server-assigned sequence number per conversation |
| Sender sees sent or failed, no duplicates | A client-generated message ID that the server dedupes on |
| Groups up to 500 | Store once, fan out to online members, with batching |
| Send-rate limit | A limiter keyed by user, enforced where messages arrive (the socket) |
| 1 billion a day, 1 year | A write-heavy, large store: Cassandra |

## Questions worth asking next time

You covered a lot. Others to run through for a chat system: are messages end-to-end encrypted, do we need typing indicators or presence, can users have several devices at once, should deleted or edited messages be supported, and is there a message size or retention requirement per conversation.

---

# 3. Step 2: Estimation

## Round aggressively

In a real round with limited time: 1B a day is about 12K per second, and a 5x peak is about 60K. Your exact figures were fine, but spend minutes on them only if the interviewer asks.

## Messages per second

- 1 billion / 86,400 is about 11,575 per second, or about 11,600 on average.
- At a 5x peak: about 57,875, or about 58,000 per second.
- Storage writes follow messages: one stored row per message.

## Deliveries, not just messages

A group message is delivered to many people, so deliveries are far higher than messages. Assume an average group of 10:

- Group messages: 20% of 1B is 200 million, and 200M x 10 is 2 billion deliveries.
- One-on-one: 800 million deliveries.
- Total: 2.8 billion deliveries per day, about 32,400 per second on average and about 162,000 per second at a 5x peak.
- Spread over about 110 servers that's about 1,500 deliveries per second per server, which is light.
- You store each message once (about 11,600 per second) and deliver it many times. Deliveries are not stored writes.

## Message size

- Use an **average** message size for storage, not the 4,000-character cap. The cap is a ceiling for limits and buffers. Most chat messages are a sentence or two. The average of about 100 bytes of text is a reasonable assumption. Using 4,000 bytes would mean buying 40 times the storage you need.
- Add the stored fields: sender, conversation (or receiver), timestamp, message ID, sequence number, TTL. About 100 bytes. So about 200 bytes per message.

## Storage

- 1 billion x 200 bytes is 200 GB per day.
- 200 GB x 365 is 73,000 GB, which is **73 TB** per year.
- With 3 copies (RF 3): about 219 TB.
- At about 2 TB per node (an assumption): about 110 nodes. Text compresses well, so the real number could be lower.

The slip to avoid: 73,000 GB is 73 TB, not 73,000 TB. After any storage figure, convert one step at a time up the ladder (KB, MB, GB, TB), and check that it's plausible.

## Connections and servers

- Each connected user holds **one** connection, from their device to a chat server. The person they chat with is on their own connection, possibly to another server. So 5 million connected users is about 5 million connections, not 2.5 million.
- You must assume how many connections one server holds. Idle connections are cheap, and tens of thousands per server is commonly used. Your assumption was 50,000.
- 5,000,000 / 50,000 is 100 servers. Adding 10% gives 110.
- 110 servers give about 500,000 connections of spare room in total, which easily absorbs one crashed server's 45,000.
- A single server failing is easy. Losing a whole availability zone is harder: with 110 servers across 3 zones, losing one leaves about 73 servers, or about 3.7 million connections of capacity, short of the 5 million needed at peak. Surviving that needs about 150 servers (5M divided by two thirds, then divided by 50,000). Whether to pay for that is a business decision, and saying so is a good answer.
- Connection **count** and connection **surge** are different. When a server dies, 45,000 users reconnect in a burst, and each needs authenticating, registering in the lookup, and catching up on missed messages. See section 8.

---

# 4. Step 3: High-level design

## The design

```
Client <--WebSocket--> Load balancer -> Chat servers (about 110)
                                           |-- Redis: user -> server (heartbeat expiry)
                                           |-- Cassandra: messages, by conversation + sequence
                                           |-- forward to the recipient's chat server
                                           '-- push notification service (offline users)
Catch-up on reconnect: "messages after sequence N" for each conversation
```

## What happens to a message

1. The sender's app creates a **client message ID** and sends the message over its WebSocket. It shows "sending."
2. The chat server checks the per-user send limit (section 7).
3. The server dedupes on the client message ID, assigns the conversation's next **sequence number**, and writes the message to Cassandra at QUORUM.
4. Only after it's stored does the server acknowledge the sender, who now sees "sent."
5. The server looks up which servers the recipients are on, and forwards the message to them. Those servers push it down the recipients' sockets.
6. A recipient with no entry in the lookup is offline: the message is already stored, and the server calls the push notification service.
7. When an offline user reconnects, their app asks for each conversation's messages after the last sequence number it has.

The order matters: **store first, then acknowledge, then deliver.** Storage is the source of truth, and delivery is a best-effort push on top of it. That's why crashes, offline users, and reconnects all reduce to "fetch after my last sequence number."

## The three cases

- **One-on-one, recipient online:** steps 1 to 5.
- **Recipient offline:** steps 1 to 4, then 6 and 7. Nothing extra is stored. The offline user reads the history.
- **Group of 500:** store the message **once** in the conversation's log (not 500 copies). Look up all the members' servers in one batch, send one forward per server, and each server pushes to its members who are online. Offline members read from the log later.

## Groups: fan-out on write versus on read

- **Fan-out on write (what we chose):** do the deliveries when the message is sent. Sending costs more (up to 500 deliveries), and members don't have to fetch anything. Right for real-time chat.
- **Fan-out on read:** do almost nothing at send time, and each reader pulls the log. Sending is cheap, but every reader does work and delivery is slower.
- Either way, the message is stored once. The trade-off is send-time work versus read-time work, and it isn't "throughput versus a lazy approach."

## Where the send limit sits

Per-user limits with a 429 and `Retry-After` is the right pattern for HTTP requests. But once a WebSocket is open, messages travel over the socket and don't pass through the gateway as separate requests. So enforce the send limit in the **chat server**, per user, using the shared rate limiter, and return an error message over the socket with a retry time. The gateway can still limit new connections per user or IP.

## What to fix from your first pass

1. Don't rely on polling for real-time delivery. At 5 million users polling once a second, that's 5 million requests per second to learn about 12,000 messages per second, and the delay would miss half a second. Use WebSockets.
2. Say how a message gets from the sender's server to the recipient's server (section 6).
3. Say how a sequence number is assigned without a race (section 7).
4. Enforce the send limit on the socket, not only at the gateway.

---

# 5. Cassandra: partitioning and clustering

## The big picture

Cassandra is a cluster of equal nodes. There's no master and no single coordinator to fail. Data is divided by a **partition key**, and each division is copied to several nodes. It's built for very high write volume and for reading data by key. It's not built for joins or ad-hoc queries.

## How a cluster is set up

- **Nodes** are arranged in a ring. Each owns ranges of a hash space (the same idea as the consistent hashing ring from the rate limiter, including virtual nodes).
- **Replication factor (RF)** is how many copies of each piece of data exist. RF 3 puts each partition on three different nodes, and any of them can serve reads and writes.
- **Keyspace** is like a database, and the replication is set there:

```sql
CREATE KEYSPACE chat WITH replication =
  {'class': 'NetworkTopologyStrategy', 'dc1': 3};
```

- **Topology awareness:** the cluster knows which rack or availability zone each node is in, so the three copies land in different zones.
- **Gossip:** nodes tell each other who's alive. A new node contacts a few seed nodes to join, takes over some ranges of the ring, and receives that data from its neighbors.

## Partition key: which nodes hold the data

The primary key has two parts. The first is the **partition key**. Cassandra hashes it to a position on the ring, and that decides which three nodes own the data. All rows with the same partition key are stored together on those nodes.

The rule to design around: **a query should name a partition key.** It then goes to one set of nodes and reads one contiguous chunk. A query without the partition key has to ask the whole cluster, which is slow, and Cassandra refuses it unless you add `ALLOW FILTERING`.

## Clustering columns: sort order inside a partition

After the partition key come the **clustering columns**. They decide the order of rows within the partition on disk. Reading a range of rows in that order is a fast sequential read.

```sql
CREATE TABLE messages (
  conversation_id uuid,
  bucket          int,        -- for example a month number
  seq             bigint,     -- order number within the conversation
  message_id      uuid,
  sender_id       uuid,
  body            text,
  created_at      timestamp,
  PRIMARY KEY ((conversation_id, bucket), seq)
) WITH CLUSTERING ORDER BY (seq DESC);
```

Read `PRIMARY KEY ((conversation_id, bucket), seq)` like this: the double parentheses are the **partition key** (conversation plus time bucket), and `seq` is the **clustering column**.

```
Partition (conversation 42, bucket 2026-11)    -> nodes C, D, E
   seq 1019 | sender A | "see you at 5"
   seq 1018 | sender B | "ok"
   seq 1017 | sender A | "where are you?"
   ...
Partition (conversation 42, bucket 2026-10)    -> nodes A, B, F
   seq 987  | ...
Partition (conversation 77, bucket 2026-11)    -> nodes B, D, G
```

Two queries you'd run constantly:

```sql
-- latest 50 messages
SELECT * FROM messages
 WHERE conversation_id = ? AND bucket = ? LIMIT 50;

-- catch-up after reconnecting
SELECT * FROM messages
 WHERE conversation_id = ? AND bucket = ? AND seq > ?;
```

Each one hits a single partition and reads a contiguous range, which is why it's fast. Cassandra keeps rows sorted by `seq`, but it doesn't assign it. Your chat service numbers the messages.

## Why the bucket is there

Without `bucket`, all of a conversation's messages would sit in one partition that grows forever. Oversized partitions slow down reads, compaction, and repairs, and a common rule of thumb is to keep a partition under about 100 MB. Bucketing by month (or by a fixed number of messages) caps the size. The cost is that reading further back means querying the previous bucket too.

## Design tables around your queries

List the questions the app asks, then build one table per question, even if that duplicates data.

| Query | Table | Partition key | Clustering |
|---|---|---|---|
| Messages in a conversation | `messages` | `(conversation_id, bucket)` | `seq` |
| A user's conversation list | `conversations_by_user` | `user_id` | `last_activity DESC` |
| Has this send already been stored? | `sent_by_client_id` | `(sender_id, client_message_id)` | none |

The last table is how you dedupe retries: insert with `IF NOT EXISTS`, and if the row is already there, it's a repeat.

## How a write works (and why writes are fast)

1. The client connects to any node, which becomes the **coordinator** for that request.
2. The coordinator hashes the partition key and sends the write to the three replica nodes.
3. Each replica appends to a **commit log** (for durability) and updates an in-memory **memtable**, and doesn't edit data in place.
4. The memtable is later flushed to immutable sorted files (SSTables), and a background **compaction** merges them.

The write is an append, which is why Cassandra handles huge write volumes.

## Consistency levels

You choose how many replicas must respond to each read or write:

- `ONE`: fastest, and may read stale data
- `QUORUM`: 2 of 3, which tolerates one node being down
- `ALL`: all 3, slowest, and fails if any node is down

If reads plus writes exceed the replication factor (QUORUM and QUORUM with RF 3, so 2 + 2 > 3), a read sees the latest write. For chat, QUORUM is a sensible default.

## Things that go wrong

- **No joins and no ad-hoc queries.** If you need a new query, you add a table.
- **Unbounded partitions.** That's what bucketing prevents.
- **Hot partitions.** One extremely busy partition loads the same three nodes. A 500-member group is fine, but a key like `date` alone would be a disaster.
- **Deletes and TTLs** leave markers called tombstones that slow reads if too many accumulate.
- **Conditional writes (`IF NOT EXISTS`)** are lightweight transactions. They run a consensus protocol, so they're slower than normal writes. Use them sparingly.

## The 30-second version

> "I'd use Cassandra because writes are heavy and the data is large. Messages are partitioned by conversation and a time bucket, so a conversation's messages live together on three replica nodes, and the bucket keeps partitions bounded. Within a partition, rows are clustered by sequence number, so loading recent messages or catching up after a reconnect is one sequential read. I'd read and write at QUORUM for consistency, and keep separate tables for the other queries, like a user's conversation list."

---

# 6. Real-time delivery: connections and the user-to-server lookup

## Why not ordinary requests

An ordinary request is the normal HTTP pattern: the client asks and the server answers. The server can't start a message on its own. Polling would work, but it's wasteful and slow.

- 5 million users polling once a second is 5 million requests per second, to learn about 12,000 messages per second.
- The delay between a message being sent and the next poll would miss the half-second target.

A **WebSocket** is a persistent two-way connection that stays open, and the server pushes messages down it as they arrive. The open socket is the session, so you authenticate once when it opens and don't need a separate session ID. Alternatives: long polling (a held-open request, less efficient) and server-sent events (one direction only).

## How a message gets from the sender's server to the recipient's

Your 110 servers each hold a share of the connections. If the sender is on server A and the recipient is on server B, A has to know where the recipient is.

- **A shared user-to-server store** (a small replicated Redis cluster) maps each connected user to their server.
- The chat server writes the entry when a user connects, and removes it when they disconnect.
- The entry has a **short expiry** that the server keeps refreshing with heartbeats. If a server dies, its entries expire by themselves, so nothing points at a dead server forever. When the user reconnects elsewhere, the new connection overwrites the old entry right away.
- Server A looks up the recipient, forwards the message to B, and B pushes it down the socket.
- **No entry, or a failed forward:** treat the user as offline. The message is already stored, so send a push notification.
- **Groups:** look up all the members in one batch, group them by server, and send one forward per server instead of 500 separate messages.

## Sizing the lookup

- 5 million connected users at about 100 bytes each is about 500 MB, so memory isn't a concern.
- The load is higher: lookups follow deliveries (about 162,000 per second at peak). Batch the lookups (one per group, not per member) and shard if needed.

## Why not the alternatives

- **A copy of the table in every server** goes stale when a user moves from one server to another, and keeping 110 copies in step is its own problem.
- **The load balancer as the router:** its job is to spread new connections. It doesn't track which user is on which server, and it isn't in the path of server-to-server messages. Consistent hashing for connections would also strand a user whenever a server is added or removed.
- **A separate node with replicas** (the Redis cluster above) is the right shape: one place every server can trust.

---

# 7. Ordering, retries, and catch-up

## One order for everyone

- The server assigns each message a **sequence number** within its conversation. Clients sort by it.
- Don't use client timestamps, because device clocks disagree.
- To assign numbers without a race, route every message of a conversation to the same place (hash the conversation ID to a chat server or partition), so a single component numbers them one at a time. Or use an atomic counter per conversation.
- A client that sees a gap in the numbers knows a message is missing and can fetch it.

## Failed sends and retries

- The sender's app creates a **client message ID** and keeps retrying with the same ID until it gets an acknowledgment, automatically, without the user doing anything.
- The server dedupes on that ID (the `sent_by_client_id` table with `IF NOT EXISTS`), so a retry never creates a second message.
- "Sent" means the server stored it durably and acknowledged. That's the point where it must never be lost.
- The client creates the message ID, and the server assigns the sequence number separately.
- Same pattern as the notification system's idempotency keys.

## Catch-up after a reconnect

The client sends the conversation ID and the last sequence number it has. The server reads the messages after that number from Cassandra and returns them, in pages. Because the same mechanism serves offline users, reconnecting users, and a new device (which asks from the beginning), there's no separate "offline storage."

## At-least-once plus de-duplication

A message may be delivered live and then fetched again during catch-up. The client ignores any sequence number it already has. At-least-once delivery plus de-duplication on the client gives "exactly once, in order."

## The send limit

Enforce it per user in the chat server (section 4), using the shared limiter from the rate limiter design. A rejected message returns an error over the socket with a retry time.

---

# 8. Step 4: Deep dive, a chat server crash

**Scenario:** at the busiest hour, one chat server crashes. It held about 45,000 connections and was in the middle of delivering messages.

## The timeline (Alice, Bob, Carol)

- **Bob** is connected to chat server B. **Alice** and **Carol** are connected to other, healthy servers.

1. Alice sends Bob message 1. Her server stores it, looks up Bob, forwards it to server B, and B pushes it down Bob's socket.
2. Server B crashes. Bob's app loses its connection and starts reconnecting.
3. Carol sends Bob message 2. Her server stores it, then tries to deliver it. The lookup for Bob points at the dead server, or finds nothing.
4. Bob reconnects to server D and asks for what he missed.

Both messages are sent **to** Bob, by other people. The retry-until-acknowledged behavior is for a message a user is **sending**, in the other direction.

## Why nothing is lost

- Message 2 was **stored first**, so the failed delivery loses nothing. "Sent" meant stored, and delivered and read receipts are out of scope, so Carol doesn't need to know Bob was offline. The server can fire a push notification, since Bob appears to be offline.
- Message 1: either Bob's app got it before the crash, so his last sequence number includes it, or it didn't, so it falls after his last number and he fetches it again. If they overlap, the client ignores a sequence number it already has.

## The reconnection surge

45,000 clients reconnect at nearly the same moment, and each reconnection needs authenticating, registering in the lookup, and a catch-up read.

- **Client:** wait a random, growing delay before reconnecting (exponential backoff with jitter).
- **Servers:** limit how many new connections they accept per second.
- **Catch-up reads:** paged.
- Spread over about 30 seconds, 45,000 reconnects become about 1,500 per second.
- The clients find a working server through the load balancer, which only sends connections to healthy servers. The lookup entries for the dead server expire by themselves, and each new connection overwrites the old entry.

---

# 9. Step 5: Scale and failure

## 10x growth

If everything grows 10x:

| Item | Today | At 10x |
|---|---|---|
| Connected at peak | 5 million | 50 million |
| Chat servers (at 50,000 each) | about 100 (110 with headroom) | about 1,000 |
| Messages per day | 1 billion | 10 billion |
| Messages per second (average, 5x peak) | about 11,600 (about 58,000) | about 116,000 (about 580,000) |
| Deliveries per day | 2.8 billion | 28 billion |
| Deliveries per second (average, 5x peak) | about 32,400 (about 162,000) | about 324,000 (about 1.6 million) |
| Storage per year | 73 TB | about 730 TB |
| With 3 copies | about 219 TB | about 2.2 PB |
| Cassandra nodes (at 2 TB each) | about 110 | about 1,100 |

Dynamic scaling of servers and more Cassandra nodes are the right changes. In a real round, do this by multiplying by 10 instead of recomputing from scratch. Things to watch first:

- The connection layer (and zone-level headroom).
- The user-to-server lookup, at 1.6 million deliveries per second at peak. Batch the lookups and shard the store.
- Hot Cassandra partitions.
- Cost: compression and moving old messages to cheaper storage would reduce the storage.

## Region failure

- Replicate Cassandra across regions, route traffic to a healthy region, and clients retry and reconnect there.
- A conversation's messages should be numbered in **one region at a time**. If two regions both assign sequence numbers, the "same order for everyone" requirement breaks.
- The user-to-server lookup is rebuilt as clients reconnect.
- Replication is asynchronous, so a message acknowledged just before the failure may not have copied over yet. Say whether you accept that, or pay for stronger replication with higher write latency.
- Messages that failed to send are retried by the clients with the same message ID, so nothing duplicates.
- The change from a single-region design: a multi-region deployment with automatic failover, global traffic routing, and cross-region replication.

---

# 10. More topics to study

- **WebSocket load balancing:** connections are long-lived, so after adding servers, existing connections don't move on their own. Plan to drain servers gracefully, and let new connections fill new servers.
- **Heartbeats:** the client and server ping each other so dead connections are noticed. The same heartbeat refreshes the lookup entry's expiry.
- **Presence (online indicator):** derived from the same lookup entries and heartbeats. It's a nice-to-have, and it gets expensive for big groups, so update it lazily.
- **Slow clients and backpressure:** buffer limits per connection. Drop a connection that falls too far behind, and let it catch up through the history.
- **Multi-device:** each device keeps its own last sequence number per conversation. A new device fetches the history from the start.
- **Attachments (if asked):** upload the file to object storage, and put a reference (URL, size, type) in the message. The chat path stays small.
- **Archiving:** keep recent messages on fast storage and move older ones to cheaper storage.
- **Security:** encrypt in transit and at rest, and note end-to-end encryption as an option where servers can't read message bodies (it complicates search and multi-device).
- **Monitoring:** connection counts per server, delivery latency, queue and lookup latency, error rates, reconnection rate.
- **Fan-out for very large groups:** at 500 members, per-server batching is enough. For channels with millions of members, you'd switch to fan-out on read.

---

# 11. Practice questions with strong answers

Try answering each out loud before reading.

**How do messages reach a user in real time?**
A WebSocket stays open from the user's device to a chat server, and the server pushes messages down it. Polling would be too slow and wasteful.

**How does a message get from one chat server to another?**
A shared Redis store maps each user to their server, with an expiry refreshed by heartbeats. The sender's server looks up the recipient and forwards the message to their server.

**How does everyone see the same order?**
The server assigns a sequence number per conversation, with one component numbering a conversation's messages (route by conversation ID). Clients sort by it, and don't use client timestamps.

**How do you avoid duplicates on a retry?**
The client creates a message ID and reuses it on every retry. The server dedupes with a conditional insert.

**What happens to a message sent to an offline user?**
It's already stored, with a sequence number. The server sends a push notification. When the user reconnects, the app asks for messages after its last sequence number.

**What does "sent" mean?**
The server stored it durably and acknowledged. Delivered and read receipts are out of scope.

**A group of 500: how do you deliver?**
Store the message once. Look up all the members' servers in one batch, send one forward per server, and each server pushes to its online members. Offline members read the log later.

**What happens when a chat server crashes?**
Its clients reconnect with randomized backoff to a healthy server, the lookup entries expire and are overwritten, and each client catches up by asking for messages after its last sequence number. Nothing is lost, because messages are stored before they are delivered.

**Where does the send limit sit?**
In the chat server, per user, because messages travel over an open socket. The gateway limits new connections.

**Why Cassandra, and what's the key?**
Heavy writes and a large dataset. Partition by (conversation, time bucket) and cluster by sequence number, so a conversation's recent messages are one sequential read.

**What breaks first at 10x?**
The connection layer and the lookup, then Cassandra capacity. Scale servers and nodes, batch lookups, and shard.

**A region goes down. What happens?**
Traffic goes to a healthy region with replicated data. Clients reconnect and catch up. A conversation's numbering stays in one region at a time, and replication lag means a very recent message might be missing, which you state and accept.

---

# 12. Glossary

| Term | Meaning |
|---|---|
| WebSocket | A persistent two-way connection between a client and a server, so the server can push messages |
| Long polling | A request the server holds open until it has something to send |
| Chat server | A server that holds users' WebSocket connections and routes messages |
| Sequence number | A per-conversation number the server assigns, defining the message order |
| Client message ID | An ID the sending app creates so retries can be deduplicated |
| Fan-out on write | Doing the deliveries when a message is sent |
| Fan-out on read | Doing little at send time and letting each reader pull |
| Presence | Whether a user is online |
| Heartbeat | A periodic signal showing a connection or server is alive |
| Backoff and jitter | Waiting longer after each failed attempt, with random variation so clients don't synchronize |
| Partition key (Cassandra) | The part of the primary key that decides which nodes hold the data |
| Clustering column (Cassandra) | The part that sorts rows inside a partition |
| Replication factor (RF) | The number of copies of each record |
| Quorum | A majority of replicas (2 of 3) |
| Coordinator (Cassandra) | The node that handles a request and talks to the replicas |
| Memtable, commit log, SSTable | In-memory buffer, durable append log, and immutable sorted file in Cassandra's write path |
| Compaction | Merging SSTables in the background |
| Tombstone | A deletion marker |
| Lightweight transaction | A conditional write (`IF NOT EXISTS`), slower than a normal write |
| At-least-once delivery | A message is never lost but may arrive more than once |
| Throughput vs capacity | How much flows per second versus how much is held |

---

# 13. Cheat sheets and slips to watch for

## Formulas

- Rate = volume per day / 86,400. Peak = rate x peak factor (state it).
- Deliveries per day = one-on-one messages + group messages x average group size.
- Storage = messages per day x bytes per message x retention days. Times the replication factor for disk.
- Connections: one per connected user. Servers = connections / connections per server, plus headroom (and zone-level headroom for a zone loss).
- Reconnect rate = clients to reconnect / seconds of spread.

## Units

KB = 10^3, MB = 10^6, GB = 10^9, TB = 10^12, PB = 10^15 bytes. Seconds per day is 86,400.

## Slips from this session

1. Using the 4,000-character cap as the average message size. Use an average, and mention the cap as the upper bound.
2. Unit slips on storage (26.645 TB, then 73,000 TB, instead of 73 TB). Convert one step at a time.
3. Counting connections as 2 per conversation. It's one connection per connected user.
4. Polling for real-time delivery. Use a WebSocket.
5. Keeping the user-to-server table in each node, or in the load balancer. Use one shared store with expiring entries.
6. Describing the message-send retry as a resend by the sender when the question was about a message sent to the user on the crashed server. Reread who the sender is in a scenario, and ask if it's unclear.
7. Enforcing the send limit only at the gateway when messages travel over an open socket.
8. Saying "I'm out of time" on the 10x math. Multiply by 10 and say it.
9. Not saying the order of operations: store first, then acknowledge, then deliver.

## What went well

- Broad requirements, including offline, failure behavior, limits, and group size.
- The deliveries calculation, the key insight for chat.
- Sound instincts: sequence numbers, catch-up by last sequence number, a client-generated idempotency ID, Cassandra, and fan-out on write.
- Asking when something was unclear, which led to a clearer scenario.

## Pace for a real 40-minute round

About 5 minutes on requirements, 5 on estimation (round aggressively: 1B a day is about 12K per second, and a 5x peak is about 60K), 10 on the high-level design, 10 on one deep dive, and 5 on scale and failure.

---

# 14. The 5-minute version

> **Requirements.** 100M users, 20M daily, 5M connected at peak. 1B messages a day (80% one-on-one, 20% group, up to 500 members). Text only, 4,000-character cap, 1-year history. Offline users get messages on reconnect, the sender sees sending, sent, or failed, and everyone in a conversation sees the same order.
>
> **Estimates.** About 11,600 messages per second on average and about 58,000 at a 5x peak. Groups of about 10 raise deliveries to about 2.8B a day, about 32,400 per second on average and about 162,000 at peak. At about 200 bytes per message, that's 200 GB a day and 73 TB a year, or about 219 TB with three copies. 5M connections at 50,000 per server is 100 servers, plus headroom.
>
> **Design.** Clients keep a WebSocket to a chat server. A shared Redis store maps each user to their server, with an expiry that heartbeats refresh. The server stores each message first (deduped on a client-generated ID, numbered with a per-conversation sequence number, written to Cassandra at QUORUM), then acknowledges the sender, then looks up the recipients and forwards the message to their servers. Offline users get a push notification and fetch messages after their last sequence number on reconnect. Groups store the message once and deliver it to online members. The send limit is enforced in the chat server.
>
> **Cassandra.** Messages are partitioned by (conversation, time bucket) and clustered by sequence number, so loading a conversation's recent messages or catching up is one sequential read.
>
> **Scale and failure.** A crashed server's clients reconnect with randomized backoff, and catch-up reads are paged. At 10x, about 1,000 servers and about 1,100 Cassandra nodes. A region failure fails over to a second region, with each conversation numbered in one region at a time.
