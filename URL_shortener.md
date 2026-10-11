# URL Shortener

## *“Design a URL shortening service like bit.ly. Users give it a long URL and get back a short one. When someone visits the short URL, they’re redirected to the original.”*

## Notes
First introduction to a system designs interview using Claude. I'll give a brief overview of the experience and highlight what I should work on moving forward.

On another note, this is a very challenging/overwhelming process so it is important to take my time and to make sure I digest each step.
I do see what the end goal is and so the proper steps to get to that end goal must be made.
This is a first attempt at system designs and I gave answers based off of current knowledge.

Firstly, there is a proper flow on how to navigate through the interview.

### Steps
1. Clarify requirements (3-5 min): ask me questions about features, scale, and constraints. 
2. Estimate (3-5 min): rough QPS and storage numbers.
3. High-level design (10 min): the main components and how a request flows through them.
4. Deep dive (10-15 min): I’ll pick an area to probe, such as how you generate the short codes, or how you handle the read traffic.
5. Bottlenecks and trade-offs (5 min): what breaks at 10x scale?

The interview felt more of a conversation than if there were right or wrong answers, so being knowledgeable about different pieces of architecture are important.
It is also important to highlight the reason behind making design choices and to follow up with if there are any trade-offs or benefits based on previous decisions.
I will go through each step, highlight the questions and answers and what I did wrong in each step.


## Clarify requirements (3-5 min): ask me questions about features, scale, and constraints. 

- Question: How should a user generate/submit their link? Should it be a part of the URL or should there be a landing page?
- Answer: Good question. Let’s treat it as an API. A client sends a long URL, and the service returns a short one. There will be a simple landing page where people paste a link, but it just calls that same API, so I don’t want you spending time on the frontend. The short URL itself is only used for redirects: someone visits it and gets sent to the original. Don’t put the long URL inside the short link.
  - **Coach's note: Interviewers usually care more about the questions that change the architecture.**
  <br></br>
- Question: What would be the average number of URLs being generated a day?
- Answer: Let’s say about 100 million new URLs per month, which works out to roughly 3 million a day. Redirects are about 100x the creation volume, so this is a read-heavy system.
  - Coach's note: You now have enough to start estimating. Convert those numbers into QPS (average, then peak) for both writes and reads, and into storage over a few years. If you want, assume an average record size of around 500 bytes. Try the math yourself and say your assumptions out loud, such as how you’re converting days to seconds and what peak multiplier you’re picking. Then I’ll check your numbers.
  <br></br>
- Question: I would like to ask more clarifying questions. How long do links last? Do we want to gather metrics for "hot" links?
- Answer: Link lifetime: By default, links stay valid for 5 years. Users can optionally set a shorter expiration when they create a link. Once a link expires, visiting it should return an error (not a redirect), and we’d like the system to eventually reclaim the space.
  - Answer pt2.: Hot links: Basic analytics, like click counts per link, are a nice-to-have. Don’t build them into the core redirect flow, but show me where they’d fit if we have time. What I really care about is that traffic is very uneven: a small set of links gets the vast majority of clicks, and I’d like to see your design account for that.
  <br></br>
- Side question: If I never asked if user can pick their own short code, would that be a mark against the interview? I understand that adding that feature changes the architecture, but those feature ideas may not come naturally.
- Answer: To make feature ideas come more naturally, run a quick mental checklist on any “design X” prompt:
- - Create: any options at creation time? (custom alias, expiration)
- - Read: what happens on the main path, and what happens when it fails (expired, not found)?
- - Update/delete: can users edit or remove what they created?
- - Users/auth: do accounts exist, or is it anonymous?
- - Analytics: does anyone need data about usage?
- - Abuse: spam, malicious links, rate limiting?
- You won’t ask about all of these every time. Scanning the list just helps you pick the two or three that would change the architecture, then state assumptions for the rest.
  <br></br>
- Question: How fast should a redirect be and is it okay if the result is stale? Do we want to prioritize latency or consistency?
- Answer: Redirects need to be fast. Aim for under ~100 ms at the 99th percentile on the server side, since a redirect sits in front of every page load for the user. The service should be highly available, with something like 99.99% uptime as the goal, because a down redirect service breaks every link anyone has shared. On consistency, prioritize latency and availability over strict consistency. A few seconds of staleness is fine in these cases:
- - A newly created link may take a moment to be redirectable from every region.
- - An expired or deleted link may keep redirecting briefly from a cache.
- There are two things I won’t compromise on:
- - A short code must never redirect to the wrong URL, and two different long URLs must never get the same short code.
- - Once you return a short link to the user, it must be durably stored. We can’t lose it.
  <br></br>
- Question: I am going to assume that anyone can generate links and that to prevent spam, there will be rate limiting based on IP.
- Answer: That works. Anonymous creation keeps the scope simple, so we won’t need accounts or authentication for now. Rate limiting on the create endpoint is a sensible abuse control. Just keep it off the redirect path, since redirects need to stay fast and are meant to be high-volume.

### Notes

- This covers about 60% of the conversation that was had. I'm going to highlight some key points.
- Interviewers usually care more about the questions that change the architecture
-  - This means that the focus should be on things that relate to the underlying system. QPS, database type, storage size, load balancing, reads/writes, CAP theorem, caching, analytics, asynchronous calls, design implementation
- Thinking about features flow
- - Create: any options at creation time? (custom alias, expiration)
- - Read: what happens on the main path, and what happens when it fails (expired, not found)?
- - Update/delete: can users edit or remove what they created?
- - Users/auth: do accounts exist, or is it anonymous?
- - Analytics: does anyone need data about usage?
- - Abuse: spam, malicious links, rate limiting?
- **Need to create another document with things related to branches of possibilities based on requirements**

## Estimate (3-5 min): rough QPS and storage numbers.

Ok, so let's calculate the write and read QPS. \
100M new urls per month \
Average of 30 days in a month means that we have 100M/30 = ~3,333,333 writes per day.
3,333,333 / 24 * 60 * 60 = ~40 writes per second \
100x read volume means about 40,000 reads per second \
Since it's ~500 bytes per record, we'd expect 12 * 100M urls a year
1,200M * 500 = 60,000M = 60B bytes. \
60B bytes / 1024 (Can I just divide by 1000 to roughly convert to MB? I'm going to assume I can for now to make it easier mental math)
60M Megabytes = 60,000 GB = 60 TB \
60 TB over 5 years = 300TB worth of storage \
**Not sure how to calculate peak, so I'd need help with that**

Feedback: \
What’s right \
Writes: 100M / 30 ≈ 3.33M/day, divided by 86,400 seconds ≈ 40 writes/sec. One note on notation: write it as ÷ (24 × 60 × 60), or just say “86,400 seconds in a day.” As typed, / 24 * 60 * 60 evaluates differently. \
Dividing by 1000 instead of 1024: yes, that’s fine. Interviewers expect rough powers of ten in these estimates.

What needs fixing

Reads: 40 writes/sec × 100 = 4,000 reads/sec, not 40,000. That’s an easy slip under pressure.

Storage: there are two errors, and they partially cancel, which makes this hard to catch.

1.2B URLs/year × 500 bytes = 600B bytes (600 GB), not 60B. Check by multiplying 1,200M × 500 = 600,000M.
60B bytes is 60 GB, not 60 TB. Walking the unit ladder: ÷1000 → 60M KB, ÷1000 → 60K MB, ÷1000 → 60 GB.

Corrected: ~600 GB per year, ~3 TB over 5 years. Your 300 TB was off by 100x.

### Notes

- I forgot to convert Bytes to KB, so I was off by 1000x in size
- I multiplied 40 x 100 wrong lol. I need to make sure to double check the numbers when calculating, no sure how detrimental this is.
- I can use 1000 instead of 1024 to simplify the math and provide rough estimates faster
- 86,400 seconds in a day
- **Peak calculation is 2-5x the average**

## High-level design (10 min): the main components and how a request flows through them.

Answer: \
On creating a short link, we have a user that comes to the landing page that wants to shorten their url.
User submits the URL.
The API gateway would authorize based on the token bucket set for that IP.
Once authorized, the service will process the request, we can go into the code generation later.
The data should be saved within our database at this point.
A shortened URL will have been generated and will return to the user.

On following a short link
A user enters the short link into the browser.
The link hits our service.
A cache will determine if the link is cached and will return the URL if it is.
If not, the service will use a short URL to original URL algorithm to fetch the original URL.
This data will be written to the cache asynchronously why the original URL is returned. This asynchronous step can also gather metrics if needed.

Response for creation
1. “Short URL to original URL algorithm.” This is the most important thing to correct. On a cache miss, there’s no algorithm that turns a short code back into the original URL. If codes are random or counter-based, the original URL isn’t encoded in them at all. The miss path is a database lookup keyed by the short code. Saying “I’d look up the code in the DB” is the precise version. If you meant that, it’s a wording fix, but in an interview, wording like this can make the interviewer think you misunderstand the problem.
2. Authorization vs. rate limiting. The token bucket decides whether to admit a request, which isn’t authorization. That distinction matters because we said anonymous creation means there’s no authentication or authorization at all. Say “rate limit” or “throttle.”
3. No database named, and no data model. You said “our database,” but I’d want to hear what kind and what’s in a record. The access pattern is a simple lookup by short code, with no joins, ~3 TB over 5 years, and eventual consistency tolerated on reads. That fits a key-value or wide-column store naturally, though a relational DB would also work at this size. Pick one and justify it. A minimal record would be: short_code (key), long_url, created_at, expires_at.

### Notes

- Precision on wording is not great. Intention is correct.
- Wanted to create the database record later but got dinged by Claude :(

Response for redirecting
4. Expiration and not-found handling. What happens when the code doesn’t exist or has expired? Return an error (404 or 410), and make sure the cache respects expiry (a TTL, or checking expires_at).
The redirect status code. This is a classic follow-up. A 301 (permanent) lets browsers cache the redirect, which cuts your traffic but means you never see repeat clicks, so analytics undercount and expiration may not take effect. A 302 (temporary) sends every click through your service. Given we want analytics and expiring links, which would you pick and why?
A load balancer in front of the service instances, since you’ll run many replicas to handle 20K reads/sec.

5. Async cache write. It works, but the usual pattern is simply cache-aside: on a miss, read the DB, populate the cache, and return. You don’t need to make the cache write async to protect latency. For analytics, I’d keep it separate: publish a click event to a queue for consumers to aggregate. You’ve built that exact pattern before (GFS Hub’s queue-to-OMS flow), so you can say so.

### Notes
- Need to take account for 301/302 status code.
- Default for caching is cache-aside

Question: How many characters do you need to cover 100M links/month over 5 years, using a-z, A-Z, 0-9? Then compare at least two approaches and their trade-offs.

Answer: \
To determine link length, let's assume that the link can only contain alphanumeric characters.
We want to plan for 100M * 12 months * 5 years = 6B links.

(26 x 26 x 10) = ~6,250 unique characters for each char
6,000,000,000 = 6,250^x
log base 6250 (6B) = ~3
Length should be 3

Response: \
Your total is right: 100M × 12 × 5 = 6B links. The base of the calculation is the problem.

The error

You multiplied the character classes: 26 × 26 × 10 = 6,760 (not 6,250, but that’s a smaller issue). The number that matters is the alphabet size, the count of different symbols one position can hold, and you get it by adding the classes:

26 lowercase + 26 uppercase + 10 digits = 62 symbols per position

Each position picks one of 62 symbols, so a code of length x has 62^x possibilities.

Redo it
62^5 ≈ 916 million (not enough for 6B)
62^6 ≈ 56.8 billion (enough)
62^7 ≈ 3.5 trillion

So the minimum length is 6 characters, not 3.

A quick sanity check you can run in an interview: 3 characters from a 62-symbol alphabet gives only 62³ ≈ 238,000 combinations. Six billion links couldn’t possibly fit. If an answer feels small, test it that way before committing.

Six characters covers 6B links, but you’d be using about 10% of the space. If you generate codes randomly, you’ll hit collisions more often as it fills, so many designs pick 7 characters (about 0.2% fill) for headroom, fewer retries, and codes that are harder to guess. Either answer is defensible if you explain the trade-off. If someone asked for case-insensitive codes, the alphabet shrinks to 36 and you’d need 7 characters just to cover the volume.

### Notes
- I incorrectly counted the permutations of the character length, It should have been adding versus multiplying.
- Increasing space decreases collisions

## 4. Deep dive (10-15 min): I’ll pick an area to probe, such as how you generate the short codes, or how you handle the read traffic.

Question: \
Now compare at least two approaches to generating the codes. Constraints to keep in mind:

Codes must be unique, even with many servers creating links at the same moment
Short, and not easy to guess or enumerate
The write path must stay durable (no lost links)

For each approach, tell me how it avoids collisions, what it costs, and where it breaks down. Then pick one and defend it.

Answer: \
Code Generation Approaches

Creating a hash function and funneling the URL provided. This will generate a code randomly and avoids collisions given that all URLs are unique so we would expect a different code to be generated each time (although not completely guaranteed).
The service can give out a code based on the last code generated. The links will be provided in sequential order based on our 7 character length restriction. (aaaaaaa, aaaaaab, aaaaaac, etc). This will avoid collision since the service can track which code should be returned and will never provide a link that has been given.

Response: \
Approach 1: hash the URL
A hash isn’t random. It’s deterministic: the same URL always produces the same code. That gives you free deduplication, but two users shortening the same link get the same short link, which causes trouble if each should have its own expiry or click counts. Salting the input fixes that, but then you’ve lost the dedup.
Truncation is where collisions come from. A full hash (MD5, SHA-256) is far too long, so you keep the first 7 base-62 characters. Unique inputs don’t guarantee unique truncated outputs. With 6B links in a 3.5T keyspace, the last insert collides about 0.17% of the time (roughly 1 in 600). That’s rare per request, but over billions of writes it’s certain to happen.
So you need check-and-retry. Insert with a uniqueness constraint (or a conditional write), and on conflict, rehash with a salt or counter appended. The cost is an extra step on the write path and retry logic.
Upside: servers are stateless and need no coordination with each other.
Approach 2: counter
Upside: collisions are impossible by construction, with no retry logic.
Where does the counter live? “The service tracks the last code” works with one server. With many servers creating links at the same moment, either they share one counter (a single point of failure and a contention point; at ~200 writes/sec peak, throughput isn’t the issue, availability and correctness are) or each has its own and they hand out duplicates.
It’s predictable. Anyone can walk the sequence and scrape every URL in the system, and the codes reveal your volume. That conflicts with the “hard to guess” requirement.
A detail: aaaaaaa, aaaaaab... is only lowercase. The real version is an integer encoded in base 62 using all 62 symbols.
Known fixes
Coordination: each server leases a block of IDs (say 1M) from a central coordinator, then hands them out locally with no per-request coordination. If a server crashes, the unused part of its block is just a gap, which doesn’t matter in a 3.5T space.
Predictability: scramble the integer with a reversible permutation before encoding. Since it’s one-to-one, uniqueness is preserved, but the output looks random.

Coach’s note: You spotted the core trade-off: hashing avoids coordination but needs collision handling, and a counter avoids collisions but needs coordination. In an interview, that framing is worth saying out loud. One thing is missing, though. You compared the approaches but didn’t commit to one, and interviewers expect a choice with reasoning.

### Notes:
- Hashing is deterministic (duh)
- Focus on uniqueness and unpredictability to secure the system
- Reversible permutation is a thing. Idea is that if things are rearranged randomly, you can rearrange the randomness.
- Always mention a coordinator (maybe even global coordinator) when keeping counters as a solution
- Use retry logic with salts to prevent collision
- If “hard to guess” is a real requirement, say you’d use a secret key

Answer: \
I would pick the counter approach and create a central coordinator that will provide the servers a block of 1,000,000 IDs. The ID will be scrambled using a reversible permutation before encoding, this will preserve the uniqueness. If a server crashes, it can request a new block of IDs from the coordinator.

Response: \
That’s a solid, defensible choice. You committed, you used both fixes we discussed, and the crash handling is correct. Let me push on a few details.

What’s right
Block leasing keeps the coordinator off the hot path. At 1M IDs per block, a server contacts it once per million creations. Even at your 200/sec peak, that’s rare, and servers keep serving from their local block if the coordinator is briefly down.
The permutation preserves uniqueness because it’s one-to-one. Your claim holds.
Gaps are harmless. A crashed server wastes part of a block, but a 3.5T keyspace makes that irrelevant.
Where I’d push

1. The coordinator is a single point of failure. If it’s one box and it dies, servers can’t get new blocks once their current ones run out. You need it replicated and durable, either as a small consensus-backed service (ZooKeeper or etcd style) or an atomic increment on a replicated database. Also, it must persist the high-water mark before handing out a block, so a restart can never re-issue the same range. If it forgets, you get duplicate codes, which violates a hard requirement.

2. Crash recovery wording. “It can request a new block” is correct, and I’d make the reason explicit: a restarted server must never reuse its old block, because it may have issued IDs from it without recording its position. Always take a fresh block and write off the old one. A related improvement is to prefetch the next block before the current one runs out, so allocation never blocks a request.

3. How strong is the permutation? Not all reversible scrambles are equal. Multiplying by a constant modulo 62^7 is a valid one-to-one mapping (the multiplier must share no factors with 62, so odd and not a multiple of 31), but a motivated person can recover it from a few samples. A keyed scheme, such as a Feistel network (format-preserving encryption), is much harder to reverse. If “hard to guess” is a real requirement, say you’d use a secret key.

4. Keep a safety net. Even with collision-free generation, I’d still put a uniqueness constraint on short_code in the database. Bugs and misconfigured servers happen, and it’s cheap insurance.
   What would make me switch?

You didn’t answer that part, so here’s the strong version: I’d move to hashing if the business wanted deduplication (the same long URL always maps to the same short link) or if I wanted zero coordination, for example when running independent regions. In a multi-region setup, the counter approach still works by giving each region a disjoint range of IDs.

Coach’s note: The pattern to remember is that any design with a central allocator invites the follow-up “what if that dies?” Get ahead of it by saying “the coordinator is replicated and persists its state before issuing a block” in your first description. That one sentence heads off the whole line of questioning.

Question: \
Database: name your choice and justify it from the access pattern we established (lookup by code, ~3 TB over 5 years, eventual consistency OK on reads, 5-year default expiry). Then sketch the record.
Reads: you need ~20K reads/sec at peak, and a few links get most of the traffic. Explain how the cache is sized and evicted, and what happens to the database if the cache goes down.

Answer: \
I would use a DynamoDB to store the data. This is a non relational database that has fast look ups. A Redis cache can be put in front of the database. At 20K reads/sec at peak, we would want the size to be at least 500bytes * 20,000 = 10,000,000 bytes = ~10 MB. The eviction policy will be set to LRU, which will keep all the hotlinks in the cache while removing the ones that are not used. If the cache goes down, the database will receive more traffic, meaning we need to make sure that the database is powerful enough to handle 20k requests/per second.

Response: \
DynamoDB is a good fit, and you named the right cache policy (LRU) and the right failure concern. The cache sizing has an error, and the record design is missing.

What’s right
DynamoDB: the access pattern is a lookup by short code with no joins, and it scales horizontally without us managing shards. It also gives us a native TTL feature for expiry and conditional writes (insert only if the code doesn’t exist), which covers the uniqueness safety net we discussed.
Redis with LRU: hot links stay resident and cold ones age out, which matches the skewed traffic.
You recognized the cache-down risk. You’ve lived it: your CSRMS outage was a database that couldn’t absorb its read load.
What needs fixing

1. Cache sizing. 20,000 reads/sec × 500 bytes = 10 MB is a throughput figure (about 10 MB/s over the wire), not a cache size. Cache size depends on the working set, the hot data you want resident. Use the 80/20 rule:

Reads/day: 4,000 avg × 86,400 ≈ 345M
Cache the hottest ~20%: ≈ 69M entries
69M × 500 bytes ≈ ~35 GB

That’s a small Redis cluster, and a cache entry only needs the code, long URL, and expiry, so it could be smaller than a full record. Stating the assumption (80/20) out loud matters more than the exact number.

2. No record sketch. Something like this:

short_code (partition key), long_url, created_at, expires_at

One catch: DynamoDB’s TTL deletion is lazy, so expired items can linger for a while. The read path must still check expires_at and return an error for expired links. The cache TTL should also never outlive the link’s expiry.

3. Why DynamoDB over SQL. Your justification was “non-relational with fast lookups.” A sharper version: the access pattern is pure key-value, no relationships, and the data grows to terabytes, so a store that scales horizontally by key is a natural fit. A relational DB would work at 3 TB too, so acknowledge the trade-off instead of presenting it as the only option.

The cache-down answer

“Make the database powerful enough” is a valid part of it, but it’s only one layer. At 20K reads/sec, DynamoDB is comfortable (eventually consistent reads cost half a capacity unit each for items this size, so roughly 10K read units). The real risks are different:

Cold-cache stampede: if the cache restarts empty, every request misses at once and floods the database. Mitigations: run Redis replicated (a failed node doesn’t empty everything), keep DB headroom, and coalesce duplicate misses so one fetch serves many waiting requests.
Hot key: DynamoDB limits throughput per partition, so a single viral link can overload one partition even when total load looks fine. The cache absorbs that traffic. For extreme cases, add a small in-process cache on each app server.

Coach’s note: The pattern in your answer was strong on naming components and weaker on sizing them from first principles. When you size something, ask what the number measures: bytes per second (throughput) or bytes held (capacity). Mixing those up is a common estimation slip, and it’s easy to catch once you check the units.

### Notes
- At this point, my brain is fried and I'm just chugging along
- Another emphasis on making sure I know the pros and cons between DB choices
- **Formula for cache size is AVG reads per day x 20% x size per record**

## Bottlenecks and trade-offs (5 min): what breaks at 10x scale?

Questions: \
10x growth: traffic goes to ~200K peak reads/sec. What breaks first, and what do you change?
Failure: an entire region goes down. How do users still get redirected, and what happens to link creation?

Answer: \
At 10x growth, the cache would fill up and requests to the database will increase.  This means that we would want to increase the cache size and determine if the growth will persist and look into increasing our database as well.

If an entire region goes down, I'd want the traffic to be redirected from a load balancer to a working region. Link creation should still work since we used a counter approach to generating links, meaning that the working servers will still have unique ids to provide.

Response: \
You’re heading in the right direction on both. Neither answer is specific enough yet, so let me push on each.

10x growth

What’s right: you tied the cache to database load, which is the correct causal chain. And “determine if the growth will persist” is good judgment, because it separates a permanent capacity change from a temporary spike (autoscaling or on-demand capacity vs. committing to bigger provisioned infrastructure).

What’s missing: “increase the cache and the database” is true of nearly any system. I want to hear what actually breaks first and why. A full cache isn’t a failure by itself, because LRU just evicts cold entries. The real problems:

The cache tier’s throughput and memory. At 200K reads/sec, a single Redis node is out of its depth (a node typically tops out around the order of 100K ops/sec), and the working set under the same 80/20 assumption grows to about 350 GB. So you shard the cache by key (Redis Cluster or consistent hashing), which spreads both load and memory.
The app tier. Stateless servers scale horizontally behind the load balancer, so this is the easy one, but you should name it.
The hit ratio is the metric to watch. At a 95% hit ratio, 200K reads/sec leaves only ~10K/sec reaching DynamoDB, which it handles comfortably. If the hit ratio drops, database load balloons. So the database is not necessarily the first thing to scale.
Hot keys. One viral link can overload a single cache shard or database partition regardless of total capacity. A small in-process cache on each app server absorbs that.

Things that hold up at 10x, which would be strong to point out:

Writes: ~2,000/sec at peak, still easy, and block leasing keeps the coordinator off the hot path.
Storage: ~30 TB over 5 years, still fine for DynamoDB.
Keyspace: 60B links in a 3.5T space is under 2% full, so 7 characters still works. (Six wouldn’t have been comfortable, so that decision pays off.)
Region failure

What’s right: redirecting traffic to a healthy region is the right goal, and you connected link creation to the ID design.

What needs work:

A load balancer lives inside a region. If the region dies, so does its load balancer. You need global routing above it, such as DNS failover with health checks (Route 53 style) or anycast, so traffic shifts automatically.
Is the data there? The healthy region can only serve redirects if the links exist in it. That means cross-region replication, for example DynamoDB Global Tables. Replication is asynchronous, which fits our “eventual consistency is fine” requirement, but it also means links created seconds before the failure might not have replicated yet. That’s a real tension with “never lose a link we returned to the user,” so name it as a trade-off.
Capacity. If two regions each run at 60%, one can’t absorb both. Plan for N+1 headroom. The survivor’s cache will also be cold for the new traffic, so expect a stampede on the database. This ties back to the cold-cache mitigations from last time.
Creation: you’re mostly right, but the reason matters. Servers in the surviving region can keep issuing codes because they hold leased ID blocks, and they need a coordinator that isn’t stuck in the failed region. Give each region a disjoint range of IDs and a replicated coordinator, so the regions never issue the same code.

Coach’s note: The recurring habit to build is asking “where does this component live when the failure happens?” Your region answer was right in spirit but assumed the load balancer, the data, and the coordinator would all survive somehow. Walking each component through the failure scenario catches those gaps.

### Notes
- Be more descriptive with answers even when your brain is fried
- Any time load increase is mentioned, should think about horizontal scaling plus regional scaling + failover

## Debrief

This is the compressed answer you’d give if an interviewer said “walk me through your design.” Practice saying it out loud.

Requirements. Users submit a long URL and get a short one back, and visiting the short link redirects to the original. Creation is anonymous with per-IP rate limiting. Links default to a 5-year lifetime with optional shorter expiry. Redirects must be fast (p99 under ~100 ms) and highly available. Short codes must be unique and durable, and slightly stale reads are acceptable. Click analytics are a nice-to-have, kept off the critical path.

Estimates. 100M new links a month is about 40 writes/sec on average, or ~200 at 5x peak. Reads are 100x that: ~4,000/sec on average and ~20K at peak. At ~500 bytes per record, that’s ~600 GB a year and ~3 TB over five years. Over five years that’s 6B links, so a 7-character base-62 code (3.5T possibilities, ~0.2% used) leaves plenty of room.

Design. Clients hit a load balancer, then stateless app servers. The gateway throttles creation with a token bucket per IP. To create a link, a server takes the next ID from its leased block of 1M, scrambles it with a keyed reversible permutation, encodes it in base 62, and writes {short_code, long_url, created_at, expires_at} to DynamoDB with a conditional write as a uniqueness safety net. A replicated coordinator hands out blocks and persists its high-water mark first. For redirects, we check Redis (cache-aside, LRU, ~35 GB under an 80/20 assumption). On a miss we look up the code in DynamoDB, populate the cache, check expiry, and return a 302 (so every click reaches us for analytics and expiry). Click events go onto a queue and are aggregated asynchronously.

Scale and failure. At 10x, I’d shard the cache, scale the app tier horizontally, and watch the hit ratio, since that determines database load. For hot keys, I’d add a small in-process cache. For a region failure, global DNS failover routes to a healthy region, Global Tables replicate the data (asynchronously, so very recent links are a known risk), each region has a disjoint ID range, and capacity is planned with N+1 headroom.

What went well
You committed to a design and defended it, including the fixes for its weaknesses.
You kept analytics and rate limiting off the redirect path.
You tied the design to your own experience (the CSRMS polling outage, the GFS Hub queue pattern), which makes your answers credible.
You asked good clarifying questions and followed up on why they mattered.
What to tighten
Arithmetic and units: reads came out 10x too high, storage 100x too high (TB vs GB), and base-62 was computed by multiplying instead of adding. Writing the unit ladder at the top of your scratch space helps. So does the habit of asking “does this number seem plausible?” before moving on.
Throughput vs capacity: the 10 MB cache sizing was bytes per second, not bytes held.
Precise vocabulary: “database lookup by short code” instead of “an algorithm,” and “rate limit” instead of “authorize.”
Failure walkthroughs: for each component (load balancer, data, coordinator), ask where it lives when the failure happens.
Open item: you never answered the 301 vs 302 question. Give it a try: which would you choose, and what do you give up?
