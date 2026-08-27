# System Design — Complete Notes (HLD + LLD)

A structured, in-depth reference covering High-Level Design (HLD) and Low-Level Design (LLD) for interviews and real-world architecture work.

---

# PART A — HIGH LEVEL DESIGN (HLD)

## Step 1: Fundamentals

### 1.1 Serverless vs Serverful

**Serverful (traditional servers)**
- You provision and manage machines (VMs, containers, bare metal) that run 24/7.
- You are responsible for scaling, patching, capacity planning, and paying for idle time.
- Predictable performance, no cold starts, full control over runtime/environment.
- Examples: EC2 instances running a Node/Java backend, a Kubernetes cluster.

**Serverless (FaaS — Function as a Service)**
- You write functions; the cloud provider handles provisioning, scaling, and runtime management.
- Billed per invocation/execution time — pay only when code runs (no idle cost).
- Auto-scales instantly (in theory) from 0 to thousands of concurrent executions.
- **Cold starts**: first invocation after idle period has extra latency because the provider must spin up a fresh execution environment.
- Stateless by design — no local disk/memory persistence between invocations; state must live externally (DB, cache, object storage).
- Execution time limits (e.g., AWS Lambda historically capped at 15 minutes).
- Examples: AWS Lambda, Google Cloud Functions, Azure Functions, Cloudflare Workers.

**When to use which**
| Use Serverless | Use Serverful |
|---|---|
| Spiky/unpredictable traffic | Steady, high, predictable traffic |
| Event-driven tasks (image resize on upload, webhook handlers) | Long-running processes, websockets, heavy stateful compute |
| Want zero infra management | Need fine-grained control over OS/runtime |
| Cost-sensitive with low/variable usage | Cost-efficient at sustained high scale (reserved instances cheaper than per-invocation billing) |

**Trade-off summary**: Serverless trades control and consistency (cold starts, execution limits) for operational simplicity and cost efficiency at variable load. Serverful trades operational overhead for predictability and control.

---

### 1.2 Horizontal vs Vertical Scaling

**Vertical Scaling (Scale Up)**
- Add more resources (CPU, RAM, disk, IOPS) to a single existing machine.
- Simple — no architecture changes needed (no distributed system complexity).
- Hard ceiling — a single machine has a max possible spec.
- Single point of failure — if that one beefed-up machine dies, everything goes down.
- Usually requires downtime to resize.
- Good fit early on, or for stateful systems that are hard to distribute (e.g., a single large relational DB before you need to shard).

**Horizontal Scaling (Scale Out)**
- Add more machines/nodes and distribute load across them (via a load balancer).
- Theoretically unlimited scaling — just add more nodes.
- Improves fault tolerance — one node dying doesn't take down the whole system.
- Adds complexity: needs load balancing, data partitioning/replication, distributed consistency, service discovery, network overhead between nodes.
- Better suited for stateless services (web/app servers) — easy to add/remove instances behind a load balancer.

**Rule of thumb**: Scale vertically first (it's cheap and simple) until you hit diminishing returns or a hard ceiling, then scale horizontally. Modern large-scale systems (Netflix, Uber, Amazon) are horizontally scaled by design because no single machine could ever handle their load, and horizontal scaling gives redundancy.

---

### 1.3 What are Threads?

- A **process** is an independent running instance of a program with its own memory space (heap, stack, code, data segments).
- A **thread** is the smallest unit of execution within a process. Multiple threads within the same process **share** the process's memory (heap, global variables) but each thread has its **own stack** and program counter.
- **Why threads exist**: to achieve concurrency (do multiple things "at once") without the overhead of spinning up entirely separate processes (which have isolated memory and are expensive to create/switch between).
- **Concurrency vs Parallelism**:
  - Concurrency = multiple tasks make progress in overlapping time periods (may or may not run at the exact same instant) — achieved via context switching on a single core.
  - Parallelism = multiple tasks literally execute at the same instant — requires multiple CPU cores.
- **Multithreading benefits**: better CPU utilization (while one thread waits on I/O, another can run), responsiveness (UI thread stays responsive while a background thread does work), faster execution of parallelizable tasks.
- **Multithreading dangers**: race conditions, deadlocks, need for synchronization (locks, semaphores, mutexes) — because threads share memory, uncoordinated access to shared data corrupts it. (Covered in-depth in LLD → Concurrency section.)
- **Thread lifecycle**: New → Runnable → Running → Blocked/Waiting → Terminated.
- **Context switching**: the CPU saving one thread's state and loading another's so it can switch execution — has overhead, so too many threads can hurt performance ("thread thrashing").
- **User threads vs Kernel threads**: user-level threads are managed by a runtime/library without OS awareness (cheap, but can't leverage multiple cores well); kernel threads are managed by the OS scheduler (heavier, but true parallel execution across cores).
- **Green threads / coroutines / goroutines**: lightweight, user-space threads (or cooperatively scheduled units) used by languages like Go and Kotlin to support massive concurrency (thousands to millions) without the overhead of OS threads.

---

### 1.4 How does the Internet work?

1. **Physical layer**: Data travels as electrical signals, light pulses (fiber optics), or radio waves through physical media (cables, satellites, cell towers) connecting devices to Internet Service Providers (ISPs), which connect to regional/global backbone networks.
2. **IP Addressing**: Every device on the internet gets an IP address (IPv4: `192.168.1.1` style, ~4.3B addresses, now scarce; IPv6: much larger address space) that identifies it uniquely for routing.
3. **DNS (Domain Name System)**: Translates human-readable domain names (`google.com`) into IP addresses. Works like a distributed phonebook:
   - Browser checks local cache → OS cache → Router cache.
   - If not found, queries a **DNS Resolver** (usually your ISP's or a public one like 8.8.8.8).
   - Resolver queries a **Root DNS server** → directs to the right **TLD server** (.com, .org) → directs to the **Authoritative Nameserver** for that specific domain, which returns the IP.
4. **Routing**: Data is broken into **packets**. Routers examine each packet's destination IP and forward it hop-by-hop toward the destination, often via different paths (packet switching), using routing protocols (BGP between networks, OSPF within a network) to find efficient paths.
5. **TCP/IP Protocol Suite**: Governs how data is packaged, addressed, transmitted, and reassembled.
   - **IP** handles addressing and routing of packets.
   - **TCP** (on top of IP) handles reliable, ordered delivery — breaks data into segments, ensures they arrive in order, retransmits lost packets (handshake, acknowledgments).
6. **Client-Server model**: Your browser (client) opens a TCP connection to a web server's IP on a port (usually 443 for HTTPS), performs a **TLS handshake** (encryption setup), then sends an **HTTP request**; the server processes it and sends back an **HTTP response** (HTML/JSON/etc.), which the browser renders.
7. **CDNs & Edge servers**: Static content is often served from geographically nearby edge servers rather than the origin server to reduce latency (see CDN section).

**End-to-end flow for `https://example.com`**:
`Browser → DNS lookup → TCP handshake with server IP → TLS handshake → HTTP GET request → Server processes (queries DB, etc.) → HTTP response → Browser renders page.`


---

## Step 2: Database

### 2.1 SQL vs NoSQL Databases

**SQL (Relational)**
- Data stored in structured tables with rows and columns; schema is fixed and defined upfront.
- Relationships enforced via foreign keys; supports JOINs across tables.
- ACID compliant (Atomicity, Consistency, Isolation, Durability) — strong transactional guarantees.
- Vertically scales more naturally; horizontal scaling (sharding) is possible but harder and requires careful design.
- Best for: structured data with complex relationships, systems needing strong consistency and transactions (banking, order management, inventory).
- Examples: PostgreSQL, MySQL, Oracle, SQL Server.

**NoSQL (Non-relational)**
- Flexible/dynamic schema — good for unstructured or rapidly evolving data.
- Built from the ground up for horizontal scaling and distribution.
- Usually trade strict consistency for availability/partition tolerance (see CAP theorem).
- Four main types:
  - **Document stores** (MongoDB, CouchDB): JSON-like documents; good for nested/hierarchical data.
  - **Key-Value stores** (Redis, DynamoDB): simplest model — key maps to a value blob; extremely fast lookups.
  - **Column-family stores** (Cassandra, HBase): data stored by column rather than row; great for write-heavy, large-scale analytical workloads.
  - **Graph databases** (Neo4j): optimized for traversing relationships (social networks, recommendation engines).
- Best for: massive scale, flexible/evolving schemas, high write throughput, use cases where eventual consistency is acceptable.

**Decision framework**: Ask — do I need multi-row transactions and complex relational queries (→ SQL)? Or do I need to scale horizontally to huge volumes with simple access patterns and can tolerate eventual consistency (→ NoSQL)? Many real systems use both (polyglot persistence) — e.g., PostgreSQL for orders, Redis for caching/sessions, Elasticsearch for search.

---

### 2.2 In-Memory Databases

- Store data primarily in RAM instead of on disk → drastically faster reads/writes (microseconds vs milliseconds).
- Used for caching, session storage, real-time leaderboards, rate limiting, pub/sub messaging.
- Trade-off: RAM is volatile and expensive — data can be lost on crash/restart unless persistence is configured.
- Persistence strategies some in-memory DBs offer:
  - **Snapshotting**: periodically dump the full dataset to disk (e.g., Redis RDB).
  - **Append-only log (AOF)**: log every write operation to disk so it can be replayed on restart (durability closer to disk DBs, but slower than pure snapshotting).
- Examples: **Redis** (supports rich data structures: strings, hashes, lists, sets, sorted sets, pub/sub), **Memcached** (simpler, pure key-value cache, multi-threaded), Amazon ElastiCache.
- Common use: sit in front of a slower disk-based DB as a cache layer to absorb read traffic.

---

### 2.3 Data Replication & Migration

**Replication** — keeping copies of the same data on multiple nodes/servers.

Why: fault tolerance (if one node dies, others serve data), read scalability (spread read load across replicas), lower latency (place replicas geographically close to users).

**Types**:
- **Leader-Follower (Master-Slave)**: one leader handles all writes; followers replicate from the leader and typically serve reads. Simple, avoids write conflicts, but leader is a bottleneck/single point of failure for writes (until failover).
- **Multi-Leader (Master-Master)**: multiple nodes accept writes, then sync with each other. Better write availability across regions, but introduces write conflict resolution complexity.
- **Leaderless (e.g., Dynamo-style)**: any replica can accept a write; uses quorum reads/writes (e.g., write to W nodes, read from R nodes, with W + R > N) to maintain consistency guarantees. Used by Cassandra, DynamoDB.

**Synchronous vs Asynchronous replication**:
- Synchronous: leader waits for follower(s) to confirm write before acknowledging to client → strong consistency, but higher write latency, and availability suffers if a follower is slow/down.
- Asynchronous: leader acknowledges immediately, replicates in the background → lower latency, higher availability, but risk of data loss/staleness if leader crashes before replicating (replication lag).

**Data Migration** — moving data between systems/schemas/versions, typically with near-zero downtime:
- **Big-bang migration**: cut over all at once (simplest, riskiest — hard to roll back).
- **Dual-write / expand-contract pattern**: write to both old and new systems during a transition window, backfill historical data, verify consistency, then cut reads over to the new system, then stop writing to the old one.
- **Change Data Capture (CDC)**: stream changes from the source DB's write-ahead log (e.g., via Debezium) into the target system continuously.
- Key concerns: backward compatibility during transition, data validation/reconciliation, rollback plan, minimizing downtime.

---

### 2.4 Data Partitioning

- Splitting a large dataset into smaller, more manageable chunks called **partitions**, distributed across multiple nodes — necessary because a single machine can't hold or serve unlimited data/traffic.
- **Horizontal partitioning (a.k.a. sharding at the row level)**: split rows across partitions (e.g., users A–M on partition 1, N–Z on partition 2).
- **Vertical partitioning**: split columns/tables across different stores (e.g., user profile data in one DB, user activity logs in another).
- **Functional partitioning**: split by feature/service (e.g., orders DB, payments DB, inventory DB) — natural outcome of microservices.
- Goals: distribute load evenly (avoid "hot" partitions), keep related data together to minimize cross-partition queries/joins, allow the system to scale by adding more partitions.

---

### 2.5 Sharding

Sharding is the specific technique of horizontally partitioning data across multiple database instances ("shards"), each an independent database holding a subset of the data.

**Sharding strategies**:
- **Range-based sharding**: partition by a key range (e.g., user IDs 1–1M on shard 1, 1M–2M on shard 2). Simple and supports range queries efficiently, but can create hot spots if data/traffic isn't uniform (e.g., all new users hashing to the newest, busiest shard).
- **Hash-based sharding**: apply a hash function to the shard key, then mod by number of shards, to decide placement. Distributes load evenly, but range queries become inefficient (data scattered across shards), and **adding/removing shards requires re-hashing/moving large amounts of data** — this is exactly the problem **consistent hashing** solves (see Load Balancer section).
- **Directory-based sharding**: a lookup service/table maps each key to its shard. Flexible (can rebalance by just updating the mapping) but the lookup service itself becomes a critical dependency/bottleneck if not made highly available.
- **Geo-based sharding**: partition by user's geographic region — reduces latency and can help with data residency/compliance requirements.

**Challenges with sharding**:
- **Cross-shard joins/transactions** are expensive or impossible without extra coordination (2-phase commit, saga pattern).
- **Rebalancing**: as data grows, shards need to be split/redistributed without downtime.
- **Choosing the shard key** is critical — a bad key choice causes hot spots (e.g., sharding by "signup date" concentrates all current traffic on the latest shard).
- Increases operational complexity significantly (more databases to manage, monitor, back up).

---

## Step 3: Consistency vs Availability

### 3.1 Data Consistency & Its Levels

Consistency = whether all nodes/readers see the same data at the same time.

**Levels (strongest → weakest)**:
- **Strong consistency**: every read receives the most recent write, no matter which node serves it. Simple to reason about, but expensive — often requires synchronous coordination and hurts availability/latency under partition.
- **Sequential consistency**: all nodes see operations in the same order, though not necessarily instantly reflecting the latest write.
- **Causal consistency**: operations that are causally related (B depends on A) are seen by everyone in that order; unrelated operations can be seen in different orders on different nodes.
- **Eventual consistency**: if no new writes happen, all replicas will *eventually* converge to the same value — but there's a window where reads may return stale data. Common in highly available distributed systems (DNS, DynamoDB, Cassandra).
- **Read-your-writes consistency**: a user always sees their own writes immediately, even if other users might briefly see stale data (common practical compromise — e.g., after you post something, you see it, though others near you might briefly not).

### 3.2 Isolation & Its Levels (ACID's "I")

Isolation defines how/whether concurrent transactions affect each other. Problems isolation prevents:
- **Dirty read**: reading uncommitted data from another transaction that might later be rolled back.
- **Non-repeatable read**: re-reading the same row within a transaction gives a different value because another transaction committed a change in between.
- **Phantom read**: re-running the same query within a transaction returns a different *set* of rows because another transaction inserted/deleted matching rows.

**Isolation levels (weakest → strongest)**:
1. **Read Uncommitted**: transactions can see other transactions' uncommitted changes → allows dirty reads. Fastest, least safe.
2. **Read Committed**: only committed data is visible → prevents dirty reads, but non-repeatable reads can still happen.
3. **Repeatable Read**: guarantees the same row read twice within a transaction returns the same value → prevents dirty + non-repeatable reads, but phantom reads can still occur.
4. **Serializable**: transactions behave as if executed one at a time, sequentially → prevents all three anomalies. Strongest, but most expensive (heavy locking or conflict detection reduces throughput).

Trade-off: higher isolation = stronger correctness guarantees but lower concurrency/throughput.

### 3.3 CAP Theorem

In a **distributed system**, when a **network partition** occurs (nodes can't talk to each other), you must choose between:
- **Consistency (C)**: every node returns the same, most up-to-date data — but might have to refuse requests until it's sure the data is current.
- **Availability (A)**: every request gets a response — but that response might not reflect the latest write (from an unreachable node).
- **Partition tolerance (P)**: the system continues to function despite network partitions.

**Key insight**: Partition tolerance is not really optional in a real distributed system (networks *will* fail sometimes), so in practice the real choice is **CP vs AP** during a partition:
- **CP systems**: sacrifice availability to guarantee consistency (e.g., some configurations of MongoDB, HBase, ZooKeeper, traditional RDBMS in distributed setups). Will reject/delay requests rather than serve stale data.
- **AP systems**: sacrifice strict consistency to stay available (e.g., Cassandra, DynamoDB, CouchDB). Will serve possibly-stale data rather than fail the request, relying on eventual consistency to catch up.
- Note: CAP only applies *during a partition* — the rest of the time, you can have both C and A. Also, CAP is a simplification; in practice teams often reason using **PACELC** (if Partition, choose A or C; Else, choose Latency or Consistency) which captures the latency/consistency trade-off even *without* a partition.

**Practical example**: A banking ledger prioritizes Consistency (better to reject a transaction than show wrong balances). A social media "like count" or "who's online" indicator prioritizes Availability (better to show a slightly stale count than fail to load).

---

## Step 4: Cache

### 4.1 What is a Cache? (Redis, Memcached)

- A cache is a fast-access storage layer (usually in-memory) that sits between the application and a slower backing store (DB, disk, external API) to reduce latency and load on the backing store.
- Works on the principle of **locality of reference**: recently/frequently accessed data is likely to be accessed again soon.
- **Cache hit**: requested data found in cache (fast). **Cache miss**: data not in cache → fetch from source, then (usually) populate the cache for next time.
- **Redis**: in-memory data structure store; single-threaded event loop (per core), supports rich types (strings, lists, sets, sorted sets, hashes, streams), pub/sub, persistence (RDB/AOF), Lua scripting, and built-in replication/clustering. Widely used as cache, session store, rate limiter, leaderboard, message broker.
- **Memcached**: simpler, pure key-value cache; multi-threaded (can use multiple cores natively, unlike single-threaded Redis per instance), no persistence, no complex data types. Good when you need a dead-simple, very high-throughput cache and don't need Redis's extra features.
- **Cache-Aside (Lazy Loading) pattern**: application checks cache first; on miss, reads from DB and writes result into cache. Most common pattern — cache only ever contains what's actually been requested.

### 4.2 Cache Write Policies

- **Write-Through**: data is written to the cache **and** the backing store synchronously, as a single logical operation. Cache is always consistent with the DB; but write latency = cache write + DB write (slower writes). Good when read-after-write consistency matters.
- **Write-Back (Write-Behind)**: data is written to the cache only; the write to the backing store is deferred/batched and done asynchronously later. Very fast writes, reduces DB load (can batch/coalesce writes), but risk of **data loss** if the cache crashes before flushing to the DB — needs careful durability design.
- **Write-Around**: data is written directly to the backing store, **bypassing** the cache. The cache is only populated on a subsequent read (cache-aside style). Avoids filling the cache with data that might not be read again soon (e.g., bulk writes/imports), but the first read after a write is always a cache miss (slower).

**Choosing**: Write-through for consistency-sensitive systems with moderate write volume; write-back for write-heavy systems that can tolerate small risk of loss (or have durability elsewhere, e.g., a WAL); write-around when writes are rarely re-read immediately (logging, bulk data ingestion).

### 4.3 Cache Replacement (Eviction) Policies

Since cache memory is limited, when it's full, an eviction policy decides what to remove to make room:

- **LRU (Least Recently Used)**: evicts the item that hasn't been accessed for the longest time. Assumes recently used items will likely be used again. Implemented via a hash map + doubly linked list (O(1) get/put). Most common general-purpose policy.
- **LFU (Least Frequently Used)**: evicts the item with the lowest access count. Better than LRU when popularity is stable over time (some items are just "hot" and stay hot), but can wrongly keep an item that was popular long ago but no longer is, unless combined with aging/decay.
- **FIFO (First In First Out)**: evicts the oldest inserted item regardless of usage — simple but ignores access patterns, rarely optimal.
- **Segmented LRU (SLRU)**: splits cache into two segments — a "probationary" segment (newly added items) and a "protected" segment (items accessed more than once). An item is promoted to protected on a second access; eviction happens from probationary first. This prevents one-time-access items ("cache pollution" / scan resistance) from evicting genuinely hot data — a known weakness of plain LRU.
- **ARC (Adaptive Replacement Cache)**: dynamically balances between recency (LRU) and frequency (LFU) based on observed workload; used in some enterprise storage systems (e.g., ZFS).
- **Random Replacement**: evicts a random item — surprisingly not terrible in some workloads, extremely cheap to implement, no bookkeeping overhead.

**Trade-off framing**: LRU is simple and good for temporal-locality workloads; LFU is better for stable popularity distributions; SLRU/ARC guard against pathological access patterns (e.g., a one-off full table scan wiping out a useful cache) at the cost of extra bookkeeping complexity.

### 4.4 Content Delivery Networks (CDNs)

- A CDN is a geographically distributed network of proxy/edge servers that cache and serve content (static assets: images, video, JS/CSS, or even dynamic content via edge compute) closer to end users, reducing latency and offloading the origin server.
- **How it works**: user requests a resource → DNS routes them to the nearest edge/PoP (Point of Presence) → if the edge has the content cached, it serves it directly (cache hit); if not, it fetches from the origin server, caches it, and serves it (cache miss), so subsequent nearby users get a cache hit.
- **Push CDN**: origin proactively uploads/pushes content to CDN nodes ahead of time. Good for content that changes infrequently and you want guaranteed availability at edges from the start.
- **Pull CDN**: CDN fetches content from origin lazily, on first request (cache-aside style at the edge), then caches it for a TTL. Simpler to manage — you don't need to actively push updates.
- Benefits: lower latency (physical proximity), reduced origin load, better availability/resilience (can absorb traffic spikes and DDoS attacks), often includes caching, compression, and even security features (WAF, TLS termination) at the edge.
- Examples: Cloudflare, Akamai, Amazon CloudFront, Fastly.
- Cache invalidation on CDNs (purging stale content when origin changes) is a classic hard problem — commonly handled via TTLs, versioned URLs/cache-busting (e.g., `style.v2.css`), or explicit purge APIs.

---

## Step 5: Complete Networking

### 5.1 TCP vs UDP

**TCP (Transmission Control Protocol)**
- Connection-oriented: establishes a connection first via the **3-way handshake** (SYN → SYN-ACK → ACK) before any data is sent.
- Reliable: guarantees delivery, correct order, and error checking — lost packets are retransmitted, and out-of-order packets are reordered.
- Has flow control (don't overwhelm the receiver) and congestion control (don't overwhelm the network).
- Overhead from handshakes, acknowledgments, and retransmissions → higher latency than UDP.
- Used where correctness/completeness matters more than raw speed: HTTP/HTTPS (traditionally), file transfer (FTP), email (SMTP), database connections.

**UDP (User Datagram Protocol)**
- Connectionless: just sends packets ("datagrams") without establishing a connection or guaranteeing delivery/order.
- No retransmission, no ordering guarantee, minimal overhead → much lower latency.
- "Fire and forget" — the application layer must handle any reliability it needs itself.
- Used where speed matters more than perfect reliability, or where the application has its own reliability logic: video/audio streaming, VoIP, online gaming, DNS lookups, live broadcasts. A dropped video frame is less bad than the delay caused by waiting for a TCP retransmission.

**Rule of thumb**: TCP = correctness & completeness. UDP = speed & real-time-ness, tolerate some loss.

### 5.2 HTTP(1/2/3) & HTTPS

**HTTP/1.1**
- Text-based, request-response protocol over TCP.
- One request per TCP connection at a time by default (though "keep-alive" reuses the connection for sequential requests) → **head-of-line blocking**: a slow request blocks subsequent ones on the same connection. Browsers work around this by opening multiple parallel TCP connections per domain (usually 6).

**HTTP/2**
- Introduces **multiplexing**: multiple requests/responses can be in flight simultaneously over a *single* TCP connection, each broken into interleaved "streams" of binary frames — removes application-layer head-of-line blocking.
- **Header compression** (HPACK) reduces overhead of repetitive headers.
- **Server push**: server can proactively send resources it knows the client will need (though this feature saw limited real-world adoption and is being deprecated in some implementations).
- Still runs over a single TCP connection, so it still suffers TCP-level head-of-line blocking if a packet is lost (a single dropped packet stalls all multiplexed streams until retransmission, since TCP guarantees strict byte ordering).

**HTTP/3**
- Runs over **QUIC** (built on UDP) instead of TCP.
- QUIC implements its own reliability, ordering, and congestion control *per-stream*, so a lost packet only stalls the one stream it belongs to — solving TCP-level head-of-line blocking.
- Built-in TLS 1.3 encryption (security is not optional/separate).
- Faster connection establishment (combines transport + TLS handshake into fewer round trips, and supports 0-RTT resumption for repeat connections).
- Used increasingly by major sites (Google, Facebook, CDNs) for improved performance especially on lossy/mobile networks.

**HTTPS**
- HTTP layered over **TLS (Transport Layer Security)** for encryption, integrity, and authentication.
- TLS handshake: client and server agree on a cipher suite, server presents a certificate (verified against a trusted Certificate Authority) proving its identity, then they establish a shared symmetric session key (used for fast encryption of the actual data, since asymmetric crypto is slow) via key exchange (e.g., Diffie-Hellman).
- Protects against eavesdropping, tampering, and man-in-the-middle attacks; also required for many modern browser features and for SEO.

### 5.3 WebSockets

- A protocol providing a **persistent, full-duplex** (both directions simultaneously) connection between client and server over a single TCP connection.
- Starts as an HTTP request with an `Upgrade: websocket` header — the server agrees, and the connection "upgrades" from HTTP to the WebSocket protocol (this handshake is why WebSockets can work through existing HTTP infrastructure/proxies).
- Unlike regular HTTP request-response, either side can push a message at any time without the other having to ask first — ideal for real-time features.
- Use cases: chat applications, live notifications, collaborative editing, real-time dashboards, multiplayer games (though very latency-sensitive games often prefer UDP-based custom protocols or WebRTC).
- Contrast with polling alternatives:
  - **Short polling**: client repeatedly asks "anything new?" at intervals — wasteful, adds latency equal to poll interval.
  - **Long polling**: client asks and the server holds the request open until there's new data (or a timeout) — better than short polling but still has request overhead per cycle.
  - **WebSockets/SSE** are more efficient for frequent real-time updates since the connection stays open.
- **Server-Sent Events (SSE)** is a related, simpler alternative for **one-directional** (server → client only) real-time updates over plain HTTP.

### 5.4 WebRTC & Video Streaming

**WebRTC (Web Real-Time Communication)**
- Enables direct **peer-to-peer** audio, video, and data communication between browsers/devices, without necessarily routing through a central server for the actual media stream.
- Uses UDP under the hood (via SRTP for encrypted media) for low latency, since real-time voice/video tolerates some packet loss far better than delay.
- Needs help establishing a direct connection because most devices sit behind NAT/firewalls:
  - **STUN** server: helps a device discover its own public-facing IP/port so peers can reach it directly.
  - **TURN** server: relays traffic between peers when a direct P2P connection isn't possible (e.g., restrictive NAT/firewall) — acts as a fallback relay, at the cost of extra latency/bandwidth cost for the operator.
  - **ICE (Interactive Connectivity Establishment)** framework coordinates trying multiple candidate connection paths (direct, STUN-assisted, TURN-relayed) to find the best working one.
- Great for video calls (Zoom-like use cases at small scale/P2P), but for many-to-many (group calls, live streaming to many viewers) systems typically route through a central **SFU (Selective Forwarding Unit)** — a server that receives each participant's stream once and forwards it to others, avoiding the exponential bandwidth cost of full mesh P2P.

**Large-scale video streaming (e.g., Netflix/YouTube style, not real-time calls)**
- Uses **adaptive bitrate streaming** (e.g., HLS or DASH): video is encoded at multiple quality levels and split into small chunks (a few seconds each); the client player continuously measures available bandwidth and requests the appropriate quality chunk, adjusting up/down dynamically as network conditions change.
- Video chunks are typically served over HTTP(S) from a CDN, not over WebRTC — since it's not truly real-time (a few seconds of buffering is fine) and HTTP/CDN caching is far more scalable and cost-effective for one-to-many delivery than P2P/relayed streams.

---

## Step 6: Load Balancers

### 6.1 Load Balancing Algorithms (Stateless & Stateful)

A load balancer distributes incoming traffic across multiple backend servers to avoid overload on any single one, improve availability, and enable horizontal scaling.

**Stateless algorithms** (don't need to remember anything about past requests):
- **Round Robin**: requests distributed sequentially, one after another, across servers in a cycle. Simple, works well when all servers have similar capacity and requests are similar in cost.
- **Weighted Round Robin**: like round robin, but servers with more capacity get proportionally more requests.
- **Random**: pick a server at random — surprisingly effective at scale, low overhead.
- **Least Connections**: send the new request to whichever server currently has the fewest active connections — better than round robin when requests vary a lot in processing time.
- **Least Response Time**: sends traffic to the server with the fastest recent response time (and/or fewest active connections) — adapts to real server performance.
- **IP Hash**: hash the client's IP to consistently map them to the same server — a simple way to get "stickiness" without storing session state centrally.

**Stateful (session-aware) approaches**:
- **Sticky sessions**: the load balancer remembers (via a cookie or IP hash) which server handled a client's first request and routes all subsequent requests from that client to the same server. Needed when a server holds session state in local memory. Downside: uneven load if some users are much heavier than others, and it complicates scaling/failover (if that server dies, that user's session state is lost unless replicated).
- **Best practice**: prefer **stateless servers** with session data stored externally (e.g., in Redis or a DB) so *any* server can handle any request — this avoids needing sticky sessions at all and makes horizontal scaling much simpler.

### 6.2 Consistent Hashing

**The problem it solves**: With simple hash-based partitioning (`hash(key) % N`), adding or removing a single server (changing N) causes almost *all* keys to remap to different servers — causing massive, unnecessary data movement/cache invalidation.

**How consistent hashing works**:
- Imagine a hash ring (0 to 2^32-1 arranged in a circle). Both servers (nodes) and keys are hashed onto this same ring.
- A key is assigned to the first node found by moving clockwise from the key's position on the ring.
- When a node is added or removed, only the keys between that node and its predecessor on the ring need to move — everything else stays put. This means adding/removing a node only affects roughly `1/N` of the keys, not all of them.
- **Virtual nodes**: to avoid uneven load distribution (since raw hashing might cluster nodes unevenly on the ring), each physical server is mapped to many virtual points on the ring, spreading its "territory" out more evenly and making rebalancing on node changes smoother.
- Used in: distributed caches (Memcached client-side sharding), DynamoDB, Cassandra, CDN routing, load balancers for sharded backends.

### 6.3 Proxy & Reverse Proxy

**Forward Proxy**: sits in front of **clients**, forwarding their requests to the internet on their behalf. The server sees the proxy's IP, not the client's. Use cases: bypassing geo-restrictions, corporate content filtering/monitoring, anonymizing client identity, caching for a group of internal users.

**Reverse Proxy**: sits in front of **servers**, receiving client requests and forwarding them to the appropriate backend server(s), then returning the response to the client. The client only ever sees the reverse proxy's address. Use cases:
- Load balancing across multiple backend servers.
- SSL/TLS termination (handle encryption/decryption centrally, so backend servers don't each need to manage certs).
- Caching common responses.
- Security — hides internal server topology/IPs, can add a WAF layer.
- Compression, request routing (e.g., path-based routing to different microservices).
- Examples: Nginx, HAProxy, Envoy, AWS ALB/ELB.

**Key distinction**: Forward proxy protects/represents the *client*; reverse proxy protects/represents the *server*.

### 6.4 Rate Limiting

Rate limiting restricts how many requests a client (per user, IP, or API key) can make in a given time window — protects against abuse, DoS attacks, and ensures fair resource usage.

**Common algorithms**:
- **Fixed Window Counter**: count requests in fixed time buckets (e.g., per minute); reset the counter each new window. Simple, but allows bursts at window boundaries (e.g., a client could send max requests at the end of one window and max again immediately at the start of the next, doubling the effective rate briefly).
- **Sliding Window Log**: keep a timestamped log of every request in the last window; count log entries to decide if a new request is allowed. Very accurate, but memory-heavy (must store every timestamp).
- **Sliding Window Counter**: an approximation blending the current and previous fixed windows proportionally — much cheaper than a full log while smoothing out the boundary-burst problem of fixed windows.
- **Token Bucket**: a bucket holds tokens, refilled at a fixed rate up to a max capacity; each request consumes a token, and requests are rejected/queued if no tokens remain. Naturally allows short bursts (up to bucket size) while enforcing a long-term average rate. Very widely used (e.g., AWS API Gateway).
- **Leaky Bucket**: requests enter a queue (the "bucket") and are processed ("leak out") at a fixed constant rate, regardless of burstiness of arrival; excess requests when the bucket is full are dropped. Smooths bursty traffic into a steady output rate — good for protecting downstream systems that need consistent load.

**Where to implement**: at the edge/API gateway/reverse proxy (before hitting application servers) is most efficient; distributed rate limiting across multiple servers typically uses a shared fast store like Redis (with atomic increment operations) to keep counters consistent across nodes.

---

## Step 7: Message Queues

### 7.1 Asynchronous Processing (Kafka, RabbitMQ)

**Why async processing / message queues**:
- Decouple producers (who create work/events) from consumers (who process them) — they don't need to be online/available at the same time or know about each other directly.
- Absorb traffic spikes: producers can keep enqueueing even if consumers are temporarily slow or down (buffering).
- Enable retries, and allow scaling producers/consumers independently.
- Common uses: sending emails/notifications after signup, processing uploaded videos/images, order processing pipelines, event-driven microservice communication, log aggregation.

**RabbitMQ**
- A traditional **message broker** implementing AMQP; supports complex routing (exchanges: direct, topic, fanout, headers) to route messages to the right queue(s).
- Once a message is consumed and acknowledged, it's typically removed from the queue — designed for **task/work distribution** (each message processed once, by one consumer, e.g. a job queue).
- Supports message priorities, dead-letter queues (for failed messages), delayed messages.
- Good fit: complex routing logic, traditional task queues, RPC-style patterns.

**Kafka**
- A **distributed event streaming log/platform**, not just a queue. Messages ("events") are appended to a **topic**, which is split into **partitions** for parallelism and stored durably on disk for a configurable retention period (not deleted immediately on consumption).
- Multiple consumer groups can independently read the same topic from their own offset/position — enabling one event to be processed by many different downstream systems (e.g., an "order placed" event consumed separately by billing, inventory, and analytics).
- Because messages persist, consumers can replay history (e.g., reprocess an event stream from an earlier offset) — great for event sourcing, analytics pipelines, and audit trails.
- Extremely high throughput, built for horizontal scale (partitions distributed across a cluster of brokers).
- Good fit: event streaming, log aggregation, activity tracking, systems needing replay/durability, high-throughput pipelines feeding multiple independent consumers.

**RabbitMQ vs Kafka in one line**: RabbitMQ = smart broker doing flexible message routing for task distribution (message usually consumed once, then gone); Kafka = durable, replayable, partitioned event log for high-throughput streaming to potentially many independent consumers.

### 7.2 Publisher-Subscriber (Pub/Sub) Model

- Producers ("publishers") send messages to a **topic/channel** without knowing who (if anyone) will consume them.
- Consumers ("subscribers") express interest in a topic and receive all messages published to it, without the publisher needing to know about them.
- Decouples producers and consumers completely — enables **fan-out** (one event broadcast to many independent subscribers), unlike a simple work queue where each message typically goes to only one consumer.
- Difference from a simple queue: a **queue** (point-to-point) typically delivers each message to exactly one consumer (for load distribution of tasks); **pub/sub** delivers each message to *every* subscriber of that topic (for broadcasting events).
- Examples: Kafka topics with multiple consumer groups, Redis Pub/Sub (fire-and-forget, no persistence), Google Cloud Pub/Sub, AWS SNS (often paired with SQS: SNS fans out to multiple SQS queues, combining pub/sub fan-out with durable per-subscriber queuing).
- Common pattern in event-driven microservices: a service publishes a domain event ("UserSignedUp"); multiple independent services (email service, analytics service, recommendation service) subscribe and react — without the publisher needing to know or care who's listening.

---

## Step 8: Monoliths vs Microservices

### 8.1 Why Microservices?

**Monolith**: entire application built and deployed as a single unit — one codebase, one deployment, typically one database.
- Pros: simple to develop/test/deploy initially, easy to reason about (no network calls between components), no distributed system complexity, atomic transactions across the whole app are trivial.
- Cons: as it grows, becomes hard to maintain (large codebase, tightly coupled), a bug in one part can crash the whole app, scaling requires scaling the *entire* app even if only one part needs it, deployments become slow/risky (every change redeploys everything), hard for large teams to work independently without stepping on each other.

**Microservices**: application split into small, independently deployable services, each owning its own data and responsible for a specific business capability, communicating over the network (HTTP/gRPC/message queues).
- Pros: independent scaling (scale only the hot service), independent deployment (teams ship without waiting on each other), fault isolation (one service crashing doesn't necessarily crash others), technology flexibility (each service can use the best-fit language/DB), easier for large organizations to own and evolve services independently (matches Conway's Law — team structure mirrors service structure).
- Cons: significant added complexity — network latency/failures between services, distributed transactions are hard, need for service discovery, more complex monitoring/debugging (a single user request may span many services), operational overhead (many services to deploy/monitor/version), data consistency across services requires careful design (sagas, eventual consistency).

**When to choose which**: Start with a monolith for new/small products (simplicity, speed) and move to microservices once scale, team size, or independent-deployment needs justify the added complexity ("monolith first" is common wisdom). Premature microservices for a small team/product often creates unnecessary distributed-systems pain without a real payoff.

### 8.2 Single Point of Failure (SPOF)

- Any component whose failure alone can bring down the entire system.
- Examples: a single database instance with no replica, a single load balancer instance, a single server handling all traffic, a hardcoded dependency on one third-party service with no fallback.
- **Mitigation**: redundancy at every layer — multiple instances behind load balancers, database replication with automatic failover, multi-AZ/multi-region deployments, no single server/service that *must* be up for the whole system to function.
- Goal: design so that the failure of any *one* component degrades the system gracefully (or not at all) rather than causing total outage.

### 8.3 Avoiding Cascading Failures

Cascading failure: one component's failure/slowness causes overload/failure in dependent components, which then cascades further, potentially bringing down the whole system (a chain reaction).

**Common causes**: a slow downstream service causes callers to pile up open connections/threads waiting on it, which exhausts the caller's own resources (thread pool, connection pool), making the caller *itself* slow/unavailable to *its* callers — repeating up the chain.

**Mitigation patterns**:
- **Circuit Breaker**: after a number of failures/timeouts calling a downstream service, "trip" the circuit and immediately fail fast (or return a fallback) for subsequent calls for a cooldown period, instead of continuing to call a service that's clearly struggling — gives the downstream service room to recover and stops the caller from wasting resources waiting on doomed calls. (States: Closed → normal calls; Open → fail fast; Half-Open → try a few test calls to see if it's recovered.)
- **Timeouts**: never wait indefinitely for a downstream call — set aggressive timeouts so a slow dependency can't tie up your resources forever.
- **Bulkheads**: isolate resources (e.g., separate thread pools/connection pools per downstream dependency) so that if one dependency saturates its pool, it doesn't starve resources needed to call *other* dependencies (named after ship compartments that contain flooding to one section).
- **Retries with exponential backoff + jitter**: retry failed calls, but with increasing delay (and randomness to avoid synchronized "thundering herd" retries from many clients at once) rather than hammering an already-struggling service immediately and repeatedly.
- **Load shedding / graceful degradation**: under extreme load, deliberately reject or simplify some requests (e.g., skip a "recommended for you" section) to preserve capacity for core functionality, rather than trying to serve everything and failing entirely.
- **Rate limiting** (see Load Balancer section) also protects against one client/spike overwhelming shared resources.

### 8.4 Containerization (Docker)

- A **container** packages an application together with all its dependencies (libraries, runtime, config) into a single, portable, isolated unit that runs consistently across environments ("works on my machine" problem solved).
- Unlike a **VM** (which virtualizes an entire OS, including its own kernel — heavier, slower to start), containers share the host machine's OS kernel and isolate at the process level — much lighter weight, start in seconds (vs minutes for VMs), and pack more densely on a given machine.
- **Docker**: the dominant tool for building (via a `Dockerfile` describing the image), running, and sharing (via registries like Docker Hub) containers.
- **Docker image**: a read-only template/blueprint; a **container** is a running instance of an image.
- Enables consistent deployment across dev/staging/production, and is the foundational unit that orchestration systems (like **Kubernetes**) manage — Kubernetes handles scheduling containers across a cluster of machines, auto-scaling, self-healing (restarting crashed containers), rolling deployments, and service discovery between containers.
- Microservices and containers are a natural pairing: each microservice is packaged as its own container, deployed/scaled independently.

### 8.5 Migrating to Microservices

Migrating an existing monolith to microservices is typically done incrementally, not as a rewrite:

- **Strangler Fig Pattern**: gradually route specific pieces of functionality from the old monolith to new microservices (often via a routing layer/proxy in front), incrementally "strangling" the monolith until it's fully replaced — avoids the huge risk of a full rewrite, and the system stays functional and shippable throughout.
- **Identify service boundaries** using **Domain-Driven Design (DDD)** — find "bounded contexts" (cohesive business capabilities like "billing", "inventory", "user profile") that have low coupling to the rest of the system; extract these first.
- **Database decomposition**: often the hardest part — splitting a shared monolithic database into per-service databases, handling data that's needed by multiple services (via APIs, events, or careful duplication with sync via CDC), and managing transactions that used to be simple DB transactions but now span services (handled via the **Saga pattern**: a sequence of local transactions per service, each publishing an event that triggers the next step, with compensating transactions to undo prior steps if a later step fails).
- **Start with the least risky, most independent piece** (e.g., notifications, or a reporting service) to build confidence and tooling (CI/CD, monitoring, service discovery) before tackling core, highly-coupled domains.
- Invest early in observability (Step 9) — without good logging/tracing/monitoring, debugging a distributed system is far harder than debugging a monolith.

---

## Step 9: Monitoring & Logging

### 9.1 Logging Events & Monitoring Metrics

**Logging**: recording discrete events that happened in the system (e.g., "user 123 logged in", "payment failed with error X", a stack trace on exception). Logs are detailed, timestamped, and typically used for debugging specific incidents after the fact.
- **Structured logging** (JSON key-value format instead of free-text) makes logs machine-parseable/searchable at scale.
- **Log levels**: DEBUG (verbose, dev-only), INFO (general operational events), WARN (something unexpected but non-fatal), ERROR (a failure occurred), FATAL/CRITICAL (system-threatening).
- **Centralized logging**: in a distributed system with many services/instances, logs must be aggregated into a central searchable system rather than living on individual machines. Common stack: **ELK/EFK** (Elasticsearch + Logstash/Fluentd + Kibana), or managed solutions (Datadog, Splunk).
- **Correlation/Trace IDs**: attach a unique ID to a request at the entry point and propagate it through every service it touches, so all logs related to that one request can be tied together across a distributed system (foundational for **distributed tracing** — e.g., via OpenTelemetry, Jaeger, Zipkin).

**Monitoring (Metrics)**: numerical, aggregated measurements over time (e.g., requests per second, average latency, CPU usage, error rate, queue depth) — used to observe overall system health and trends, typically visualized on dashboards.
- **The Four Golden Signals** (Google SRE): Latency, Traffic, Errors, Saturation — a good starting point for what to monitor on any service.
- **RED method** (for services): Rate, Errors, Duration. **USE method** (for resources): Utilization, Saturation, Errors.
- Tools: Prometheus (metrics collection/storage, pull-based), Grafana (visualization/dashboards), Datadog, New Relic, CloudWatch.
- **Alerting**: set thresholds on key metrics (e.g., error rate > 5%, p99 latency > 2s) to automatically notify on-call engineers — should be tuned to avoid alert fatigue (too many false/noisy alerts causing people to ignore them).

**Logs vs Metrics vs Traces (the "three pillars of observability")**: Logs = detailed individual events (what exactly happened). Metrics = aggregated numeric trends (how is the system doing overall, cheap to store long-term). Traces = the path a single request took across multiple services (why was *this* request slow). They complement each other — metrics tell you *something* is wrong, traces/logs help you find *why*.

### 9.2 Anomaly Detection

- Automatically identifying patterns in metrics/logs that deviate significantly from expected/normal behavior, without necessarily having a pre-defined static threshold for every scenario.
- **Threshold-based**: simplest approach — alert if a metric crosses a fixed value (e.g., CPU > 90%). Easy but doesn't adapt to normal variations (e.g., naturally higher traffic during business hours).
- **Statistical methods**: flag values that are a certain number of standard deviations away from a rolling mean/moving average, or use seasonality-aware models (accounting for daily/weekly traffic patterns) to set dynamic expected ranges.
- **Machine-learning-based detection**: train models on historical "normal" behavior to flag deviations (useful for complex, multi-dimensional anomalies that simple thresholds miss) — used in fraud detection, unusual traffic pattern detection (potential DDoS), and predictive alerting before a full outage happens.
- Practical use cases: detecting a memory leak (steadily rising memory over time), catching a sudden spike in error rates right after a deployment (enabling automatic rollback), detecting fraudulent transaction patterns, spotting unusual login patterns (security).

---

## Step 10: Security

### 10.1 Tokens for Auth

- **Authentication (AuthN)**: verifying *who* you are. **Authorization (AuthZ)**: verifying *what* you're allowed to do. Tokens are typically used for both, once identity is established.
- **Session tokens (stateful)**: server generates a random session ID on login, stores session data server-side (in memory or a shared store like Redis), and gives the client the ID (usually via a cookie). Each request, the server looks up the ID to identify the user. Easy to revoke instantly (just delete server-side record), but requires server-side storage and doesn't scale as simply across many stateless servers (needs a shared session store).
- **JWT (JSON Web Token) (stateless)**: a self-contained token (Header.Payload.Signature, base64-encoded) that carries the user's identity/claims directly, cryptographically signed by the server. The server can verify authenticity by checking the signature, without needing to look anything up in a database — enabling truly stateless authentication that scales easily across many servers.
  - Downside: **hard to revoke early** — since it's self-contained and valid until its expiry, if a token is compromised, it remains valid until it expires unless you maintain a blocklist (which reintroduces state) or keep expiry very short combined with refresh tokens.
  - **Access token + Refresh token pattern**: short-lived access token (e.g., 15 min) used for actual requests, limiting exposure if leaked; a longer-lived refresh token (stored more securely, often httpOnly cookie) used to obtain new access tokens without forcing the user to log in again.
- **API keys**: simple long-lived secret strings identifying an application/service (not a specific end-user session) for server-to-server or third-party API access.

### 10.2 SSO & OAuth

**SSO (Single Sign-On)**: log in once and gain access to multiple related applications/services without re-authenticating for each — common in enterprises (log into your company account once, get access to email, docs, internal tools, etc.) via a central Identity Provider (IdP).

**OAuth 2.0**: an **authorization** framework (not primarily authentication, though often used as a building block for it) that lets a user grant a third-party application limited access to their data on another service, *without* sharing their password with that third party.
- Classic flow (Authorization Code flow): user clicks "Login with Google" on App X → redirected to Google, logs in and approves the requested permissions/scopes → Google redirects back to App X with a temporary authorization code → App X's backend exchanges that code (plus its own client secret) for an **access token** directly with Google's server → App X uses that access token to call Google's APIs on the user's behalf (e.g., to fetch profile info), without ever seeing the user's Google password.
- **OpenID Connect (OIDC)**: built on top of OAuth 2.0 to specifically add an **authentication** layer — introduces the **ID token** (a JWT containing verified identity claims: user ID, email, name), which is what actually enables "Login with Google/Facebook"-style SSO for third-party apps.
- **Scopes**: define exactly what access is being granted (e.g., `read:email`, `read:profile`) — principle of least privilege.

### 10.3 Access Control Lists & Rule Engines

- **ACL (Access Control List)**: a list attached to a resource specifying which users/roles have which permissions on it (e.g., a file's ACL might say "user A: read/write, user B: read-only, group C: no access"). Fine-grained, but can become unwieldy to manage at scale if permissions are assigned per-user everywhere.
- **RBAC (Role-Based Access Control)**: instead of assigning permissions directly to users, assign users to **roles** (admin, editor, viewer), and permissions to roles. Much easier to manage at scale — change a role's permissions once, and it applies to everyone with that role.
- **ABAC (Attribute-Based Access Control)**: access decisions based on attributes of the user, resource, and context (e.g., "allow if user.department == resource.department AND time is business hours") — more flexible/granular than RBAC, evaluated via policy/rule engines.
- **Rule engines / Policy engines**: centralized systems (e.g., Open Policy Agent - OPA) that evaluate authorization decisions against defined policies, decoupling "who can do what" logic from application code — lets you change access rules without redeploying services, and keeps authorization logic consistent across many services.

### 10.4 Encryption

- **Encryption in transit**: protecting data as it moves across a network — primarily via **TLS/HTTPS** (see Networking section). Prevents eavesdropping/tampering on the wire.
- **Encryption at rest**: protecting data stored on disk (databases, backups, object storage) so that even if the physical storage or a backup is stolen/leaked, the data is unreadable without the decryption key.
- **Symmetric encryption**: same key used to encrypt and decrypt (e.g., AES). Fast, good for bulk data encryption, but the key must be securely shared/stored (key management is the hard part).
- **Asymmetric encryption (public-key cryptography)**: a public key (shareable, used to encrypt or verify) and a private key (secret, used to decrypt or sign) — e.g., RSA, ECC. Slower than symmetric, so in practice it's often used just to securely exchange a symmetric session key (as in TLS handshakes), after which the fast symmetric cipher does the heavy lifting.
- **Hashing** (not technically encryption — one-way, not reversible): used for storing passwords (never store plaintext passwords) — combined with a unique **salt** per user (to defeat precomputed rainbow-table attacks) and a slow, deliberately expensive algorithm (bcrypt, scrypt, Argon2 — NOT fast general-purpose hashes like plain SHA-256, which are too fast and thus crackable via brute force at scale).
- **Key management**: rotating keys periodically, using dedicated key management services (AWS KMS, HashiCorp Vault) rather than hardcoding secrets, and following the principle of least privilege for who/what can access decryption keys.

---

## Step 11: System Design Trade-offs (Summary Cheat Sheet)

| Trade-off | One side | Other side | How to decide |
|---|---|---|---|
| **Push vs Pull** | Server proactively pushes updates to clients (WebSockets, webhooks, push notifications) — real-time, but server must track all subscribers and manage delivery/retries | Client periodically requests/pulls data (polling, cron jobs) — simpler, more scalable server-side, but not real-time and can waste requests when nothing changed | Push for real-time/low-latency needs (chat, live scores); pull for simplicity, when staleness of a few seconds/minutes is fine, or when the client count is huge and hard to track individually |
| **Consistency vs Availability** | Strong consistency — every read is up to date, but system may reject requests during a partition | High availability — system always responds, but might return stale data during a partition | CP for financial/inventory-critical correctness; AP for social feeds, likes/views counts, presence indicators |
| **SQL vs NoSQL** | Structured schema, ACID transactions, complex joins, vertical-first scaling | Flexible schema, built-in horizontal scale, usually eventual consistency | SQL for relational integrity & transactions; NoSQL for massive scale, flexible/evolving data, simple access patterns |
| **Memory vs Latency** | Keep more in memory (cache more aggressively) — very low latency, but higher infra cost and risk of staleness/eviction churn | Keep less in memory, hit disk/DB more — cheaper, always fresh, but higher latency | Cache hot/frequently-read data aggressively; don't cache rarely-accessed or highly volatile data (poor cache hit ratio isn't worth the memory cost) |
| **Throughput vs Latency** | Optimize for total requests processed per second (e.g., via batching requests together) — great overall capacity, but individual requests may wait longer to be batched/processed | Optimize for each individual request's response time — feels fast per-user, but may process fewer requests overall due to less efficient per-request overhead | Batch-oriented/analytics systems favor throughput; interactive user-facing systems (search-as-you-type, trading) favor latency |
| **Accuracy vs Latency** | Return a precise, fully computed answer — correct, but may take longer to compute (e.g., exact "unique visitor count") | Return an approximate answer computed fast (e.g., HyperLogLog cardinality estimate, cached/slightly stale aggregate) | Use approximation algorithms and sampling when "close enough, fast" beats "exact, slow" — common in analytics dashboards, recommendation scoring, large-scale counting |

**General framing for interviews**: There is rarely a universally "correct" choice — the right answer always depends on the specific requirements (read-heavy vs write-heavy, latency sensitivity, consistency needs, expected scale, cost constraints, team size/expertise). Always state the trade-off explicitly and justify your choice based on the stated (or reasonably assumed) requirements of the system being designed.

---

## Step 12: Practice — How to Approach Any HLD Problem

Use a consistent framework for every practice problem below:

1. **Clarify requirements** — Functional (what must it do?) and Non-functional (scale, latency, availability, consistency needs). Ask: how many users? read/write ratio? global or single-region?
2. **Back-of-envelope estimation** — estimate QPS (queries per second), storage needs, bandwidth, based on the scale given/assumed.
3. **High-level design / API design** — define core APIs, then draw the major components (clients, load balancer, services, DB, cache, queue, CDN) and how they connect.
4. **Deep dive** — pick 1–2 of the hardest parts (usually data model + the specific scaling bottleneck for that system) and go deep: sharding strategy, caching strategy, consistency approach.
5. **Identify bottlenecks & trade-offs** — single points of failure, hot spots, and explicitly state the trade-offs you chose (Step 11 table) and why, given the stated requirements.
6. **Scale/iterate** — discuss how the design evolves as scale grows 10x/100x.

### Practice Problems & the Core Concepts Each One Tests

- **YouTube** → video upload/transcoding pipeline (async processing, queues), adaptive bitrate streaming, CDN for video delivery, metadata DB design, recommendation system at a high level.
- **Twitter** → fan-out on write vs fan-out on read for timelines (push vs pull trade-off), handling celebrity accounts with millions of followers (hybrid fan-out), search indexing, rate limiting.
- **WhatsApp** → WebSockets for real-time messaging, message delivery guarantees (at-least-once, idempotency, ack/read receipts), end-to-end encryption, offline message queuing, group chat fan-out, presence (online/last-seen).
- **Uber** → real-time location tracking (geo-indexing: geohashing/quad-trees), matching riders to nearby drivers efficiently, handling surge pricing, ETA computation, consistency needs for trip state machine.
- **Amazon** → product catalog & search (search indexing, e.g., Elasticsearch), inventory management (consistency-critical), order processing (Saga pattern for distributed transactions), recommendation engine, handling flash-sale traffic spikes.
- **Dropbox / Google Drive** → file chunking & deduplication, sync conflict resolution, metadata vs blob storage separation, delta sync (only upload changed chunks), handling large file uploads (resumable uploads), sharing/permissions (ACL/RBAC).
- **Netflix** → video encoding pipeline, CDN strategy (Netflix's own Open Connect CDN), personalized recommendations at scale, adaptive streaming, handling globally distributed traffic and regional content licensing.
- **Instagram** → image storage & CDN delivery, feed generation (push vs pull, hybrid), follower graph storage at scale, like/comment counters (approximate counting at huge scale), story expiration (TTL data).
- **Zoom** → WebRTC/SFU architecture for group video calls, adaptive bitrate per participant based on bandwidth, handling NAT traversal (STUN/TURN), recording pipeline, large-scale webinar (one-to-many) vs small meeting (many-to-many) architecture differences.
- **Booking.com / Airbnb** → search & filtering at scale (geo + availability + price filters), inventory/availability consistency (avoiding double-booking — a classic strong-consistency requirement), pricing engine, review system, handling read-heavy search traffic with a write-consistent booking core.

**Tip**: For each, explicitly identify (a) the read/write pattern (read-heavy? write-heavy? both?), (b) the single hardest consistency-vs-availability decision in that system, and (c) where you'd put a cache, a queue, and a CDN. Interviewers care more about your reasoning and trade-off articulation than a "perfect" diagram.

---
---

# PART B — LOW LEVEL DESIGN (LLD)

## Step 1: Object-Oriented Programming Fundamentals

### 1.1 Encapsulation
- Bundling data (fields) and the methods that operate on that data into a single unit (a class), while restricting direct outside access to internal state.
- Achieved via access modifiers (`private`, `protected`, `public`) — expose behavior through public methods (getters/setters, or better, meaningful behavior methods) while hiding internal representation.
- Benefit: internal implementation can change freely without breaking code that depends on the class's public interface; protects invariants (e.g., a `BankAccount` class can ensure balance never goes negative by controlling all mutation through a `withdraw()` method that validates first, rather than letting external code set `balance` directly).

### 1.2 Abstraction
- Exposing only essential behavior/interface to the user while hiding complex implementation details.
- Achieved via abstract classes and interfaces — the caller knows *what* a method does, not *how*.
- Example: a `PaymentProcessor` interface exposes `processPayment(amount)`; the caller doesn't need to know whether it's calling Stripe, PayPal, or a bank API underneath.
- Abstraction is about *design/interface simplicity*; encapsulation is about *information hiding/access control* — related but distinct concepts, often used together.

### 1.3 Inheritance
- A class (subclass/child) can derive fields and methods from another class (superclass/parent), enabling code reuse and establishing an "is-a" relationship (e.g., `Car is-a Vehicle`).
- Supports method overriding — a subclass can provide its own specific implementation of a parent's method.
- **Caution**: overuse of deep inheritance hierarchies creates tight coupling and fragility (changes to a parent can break many children unexpectedly) — this is why the principle **"favor composition over inheritance"** is widely recommended: build behavior by composing smaller objects/interfaces together rather than through deep "is-a" chains.

### 1.4 Polymorphism
- "Many forms" — the ability to treat objects of different classes through a common interface, with each object responding to the same method call in its own way.
- **Compile-time (static) polymorphism**: method overloading — same method name, different parameter lists, resolved at compile time.
- **Run-time (dynamic) polymorphism**: method overriding — a subclass redefines a parent's method, and the correct version is chosen at runtime based on the actual object type, even when accessed through a parent-type reference. This is the mechanism behind most flexible, extensible LLD designs (e.g., a `List<Shape> shapes` where each shape's `.area()` call invokes the correct subclass implementation, letting you add new `Shape` subtypes without changing calling code).

### 1.5 SOLID Principles

- **S — Single Responsibility Principle**: a class should have only one reason to change — i.e., it should do one thing/represent one responsibility. Prevents "God classes" that do everything and become unmaintainable.
- **O — Open/Closed Principle**: classes should be open for extension but closed for modification — you should be able to add new behavior (e.g., a new payment method) by adding new code (a new class implementing an interface), not by editing existing, already-tested code.
- **L — Liskov Substitution Principle**: subclasses must be substitutable for their base class without breaking correctness — if `Bird` has a `fly()` method, and `Penguin extends Bird` but can't fly, that's an LSP violation; it would break code that assumes any `Bird` can fly. (Classic fix: don't force an inheritance relationship where the "is-a" doesn't fully hold behaviorally — restructure the hierarchy, e.g., separate `FlyingBird` from `Bird`.)
- **I — Interface Segregation Principle**: don't force a class to implement methods it doesn't need — prefer several small, specific interfaces over one large, general-purpose interface (e.g., don't make a `Printer` interface with `print()`, `scan()`, and `fax()` if some printers only print — split into separate interfaces).
- **D — Dependency Inversion Principle**: high-level modules shouldn't depend on low-level modules directly — both should depend on abstractions (interfaces). This is what enables swapping implementations easily and is the foundation of dependency injection (e.g., a `NotificationService` should depend on an abstract `MessageSender` interface, not directly on a concrete `EmailSender` class, so you can swap in `SmsSender` later without changing `NotificationService`).

**Why SOLID matters in LLD interviews**: These principles are exactly what interviewers look for when they say "clean, extensible design" — most LLD problems below are really testing whether you naturally apply SRP, OCP, and DIP as you design classes.

---

## Step 2: Design Patterns

Design patterns are reusable, proven solutions to common design problems. Grouped into three categories:

### 2.1 Creational Patterns (object creation mechanisms)

- **Singleton**: ensures a class has only one instance, globally accessible (e.g., a single `Logger`, a single DB connection pool manager). Implemented via a private constructor + a static instance accessor. Caution: overuse creates hidden global state and makes unit testing harder — use sparingly, and consider dependency injection instead where possible.
- **Factory Method**: defines an interface for creating an object, but lets subclasses decide which concrete class to instantiate — decouples client code from concrete class construction (e.g., a `ShapeFactory.createShape(type)` returns `Circle`, `Square`, etc. without the caller needing to know the concrete classes).
- **Abstract Factory**: a factory of factories — provides an interface for creating *families* of related objects without specifying their concrete classes (e.g., a `UIFactory` that produces matching `Button` + `Checkbox` + `Scrollbar` for either a "Windows" or "Mac" look-and-feel family, ensuring consistency across the family).
- **Builder**: separates the construction of a complex object (with many optional parameters) from its representation, building it step-by-step via a fluent interface (e.g., `PizzaBuilder().addCheese().addTopping("pepperoni").build()`) — avoids "telescoping constructors" with too many parameters.
- **Prototype**: creates new objects by cloning an existing instance (a "prototype") rather than instantiating from scratch — useful when object creation is expensive and a similar pre-configured object already exists to copy.

### 2.2 Structural Patterns (composing classes/objects into larger structures)

- **Adapter**: converts one interface into another that a client expects, letting incompatible interfaces work together (e.g., wrapping a third-party `XmlParser` behind your own `DataParser` interface so your app code doesn't depend on the third-party's specific API).
- **Bridge**: decouples an abstraction from its implementation so the two can vary independently (e.g., a `Shape` abstraction and a `Renderer` implementation, where you can mix any shape with any renderer without an exploding number of subclasses for every shape-renderer combination).
- **Composite**: composes objects into tree structures to represent part-whole hierarchies, letting clients treat individual objects and compositions of objects uniformly (e.g., a filesystem where a `File` and a `Folder` (containing files/folders) both implement the same `FileSystemItem` interface, so operations like `getSize()` work recursively without special-casing).
- **Decorator**: attaches additional responsibilities/behavior to an object dynamically, without altering its class — an alternative to subclassing for extending behavior (e.g., wrapping a basic `Coffee` object with `MilkDecorator`, then `SugarDecorator`, each adding cost/description, rather than needing a separate subclass for every combination).
- **Facade**: provides a simplified, unified interface to a complex subsystem of classes, hiding the complexity from the client (e.g., a single `OrderFacade.placeOrder()` method that internally coordinates inventory checks, payment processing, and shipping — the client doesn't need to orchestrate all three itself).
- **Proxy**: provides a stand-in/surrogate object that controls access to another object — used for lazy loading (only create the expensive real object when actually needed), access control (check permissions before delegating), caching, or logging, all transparently to the client which just interacts with the proxy as if it were the real object.

### 2.3 Behavioral Patterns (communication/responsibility between objects)

- **Strategy**: defines a family of interchangeable algorithms, encapsulates each one, and makes them swappable at runtime via a common interface (e.g., a `SortStrategy` interface with `BubbleSort`, `QuickSort` implementations that a `Sorter` class can swap without changing its own code) — directly supports the Open/Closed Principle.
- **Observer**: defines a one-to-many dependency so that when one object (the "subject") changes state, all its dependents ("observers") are automatically notified — the foundation of event-driven/pub-sub systems, UI event handling, and the basis for real notification-system designs (e.g., a `YouTubeChannel` (subject) notifying all `Subscriber` (observer) objects when a new video is uploaded).
- **Command**: encapsulates a request/action as a standalone object, allowing you to parameterize clients with different requests, queue/log requests, and support undo/redo (e.g., a `Command` interface with `execute()` and `undo()`, used to implement an editor's undo stack, or to queue remote-control button actions).
- **State**: lets an object alter its behavior when its internal state changes, appearing as if it changed class — encapsulates state-specific behavior into separate state classes rather than large conditional (`if/switch`) blocks (e.g., a `TrafficLight` with `RedState`, `YellowState`, `GreenState` classes each defining what happens on `next()`; directly useful for state-machine-heavy LLD problems like elevators or a vending machine).
- **Chain of Responsibility**: passes a request along a chain of potential handlers until one handles it — decouples sender from receiver, and lets you add/reorder handlers flexibly (e.g., a support ticket escalating from L1 → L2 → L3 support, or a middleware/filter chain processing an HTTP request).
- **Template Method**: defines the skeleton of an algorithm in a base class method, deferring some specific steps to subclasses — the overall structure/order is fixed, but individual steps are customizable (e.g., an abstract `DataProcessor.process()` that calls `readData()` → `transform()` → `writeData()` in a fixed sequence, where subclasses override individual steps for CSV vs JSON processing).
- **Visitor**: lets you add new operations to a group of related classes without modifying those classes themselves, by having each class "accept" a visitor object that performs the operation — useful when you have a stable set of classes but frequently add new operations across all of them (e.g., different node types in an AST/document structure, each accepting a `Visitor` for operations like "render", "validate", "export").
- **Mediator**: centralizes complex communication/coordination between a set of objects into a single mediator object, so objects don't need to reference each other directly (reduces many-to-many coupling into many-to-one) — e.g., an air traffic control tower coordinating planes, rather than every plane communicating directly with every other plane.

---

## Step 3: Concurrency & Thread Safety

### 3.1 Thread-Safe Injection / Design
- A class/method is **thread-safe** if it behaves correctly when accessed by multiple threads simultaneously, without needing extra synchronization from the caller.
- Techniques: make objects **immutable** where possible (immutable objects are inherently thread-safe — no mutable shared state means nothing to corrupt), confine mutable state to a single thread (thread confinement), use thread-safe data structures (e.g., `ConcurrentHashMap` instead of a plain `HashMap`), or explicitly synchronize access to shared mutable state.
- "Thread-safe injection" in dependency-injection contexts typically means ensuring injected shared dependencies (e.g., a singleton service, a shared connection pool) are themselves safe for concurrent use by all the threads that will receive/share that injected instance.

### 3.2 Locking Mechanisms
- **Mutex (Mutual Exclusion lock)**: ensures only one thread can access a critical section (shared resource) at a time — a thread must acquire the lock before entering, and release it after, blocking other threads attempting to acquire it in the meantime.
- **Semaphore**: a more general form — maintains a count and allows up to N threads to access a resource concurrently (a mutex is essentially a semaphore with count = 1). Useful for limiting concurrent access to a pool of N identical resources (e.g., a connection pool with 10 connections).
- **Read-Write Lock**: allows multiple concurrent readers (since reads don't conflict with each other) but requires exclusive access for a writer (no other readers or writers) — improves throughput for read-heavy workloads compared to a plain mutex that would serialize even reads.
- **Deadlock**: two or more threads are stuck waiting on each other's locks forever (e.g., Thread A holds Lock 1 and waits for Lock 2, while Thread B holds Lock 2 and waits for Lock 1). Requires all four of: mutual exclusion, hold-and-wait, no preemption, and circular wait to occur simultaneously.
  - **Prevention**: always acquire multiple locks in a **consistent global order** across all threads (breaks circular wait), use timeouts when trying to acquire locks (a thread gives up and retries/backs off instead of waiting forever), or minimize the scope/number of locks held simultaneously.
- **Livelock**: threads aren't blocked, but keep changing state in response to each other without making actual progress (e.g., two people repeatedly stepping aside for each other in a hallway, in sync, never passing).
- **Optimistic locking (vs pessimistic locking)**: instead of locking a resource up front (pessimistic — blocks others immediately), proceed assuming no conflict will occur, then check for a conflict (e.g., via a version number) at commit time, and retry if a conflict is detected. Good for low-contention scenarios (fewer unnecessary blocks); pessimistic locking is better for high-contention scenarios (avoids wasted retries).

### 3.3 Producer-Consumer Pattern
- A classic concurrency pattern: one or more **producer** threads generate data/tasks and place them into a shared **bounded buffer/queue**; one or more **consumer** threads take items from the queue and process them — decoupling the rate of production from the rate of consumption.
- Requires coordination so that: producers block/wait when the buffer is full (rather than overflowing), and consumers block/wait when the buffer is empty (rather than erroring on nothing to process).
- Implemented via a thread-safe blocking queue (e.g., Java's `BlockingQueue`) which internally handles the necessary locking/waiting/notifying, or manually via a mutex + condition variables (`wait()`/`notify()`) around a regular queue.
- This pattern is conceptually the foundation of message queues (Step 7 of HLD) at a single-machine/in-process level, and appears constantly in real systems: a web server's request queue feeding worker threads, a logging system buffering log lines for a background writer thread, a thread pool's task queue.

### 3.4 Race Conditions & Synchronization
- A **race condition** occurs when the correctness of a program depends on the relative timing/interleaving of multiple threads accessing shared data — if that timing is unlucky, the result is incorrect (e.g., two threads both reading a counter's value as 5, both incrementing to 6, and both writing 6 back — one increment is lost, even though two increments were intended).
- **Critical section**: the part of code that accesses shared resources and must not be executed by more than one thread at a time.
- **Synchronization**: coordinating access to shared resources (via locks, semaphores, atomic operations, or higher-level constructs) to prevent race conditions.
- **Atomic operations**: operations guaranteed to complete as a single, indivisible step from the perspective of other threads (e.g., `AtomicInteger.incrementAndGet()` in Java) — for simple counters/flags, atomics are much cheaper than a full lock since they're typically implemented via low-level CPU instructions (compare-and-swap) rather than OS-level blocking.
- **Practical rule**: identify all shared mutable state first; then either eliminate it (make it immutable/thread-local) or protect every access path to it consistently — a single un-synchronized access path is enough to reintroduce a race condition even if every other access is properly locked.

---

## Step 4: UML Diagrams

UML (Unified Modeling Language) is the standard visual notation for communicating LLD before/while coding.

**Most relevant diagrams for LLD interviews/design docs**:

- **Class Diagram** (the most important one for LLD): shows classes, their attributes, methods, and the relationships between them.
  - **Association**: a general relationship between two classes (e.g., `Teacher` teaches `Student`) — drawn as a plain line.
  - **Aggregation**: a "has-a" whole-part relationship where the part can exist independently of the whole (e.g., a `Department` has `Professors`, but a `Professor` can exist without that `Department`) — drawn as a line with an open/hollow diamond at the "whole" end.
  - **Composition**: a stronger "has-a" relationship where the part *cannot* exist without the whole (e.g., a `House` has `Rooms` — destroy the house, the rooms cease to exist as part of it) — drawn as a line with a filled/solid diamond.
  - **Inheritance/Generalization**: "is-a" relationship — drawn as a line with a hollow triangle arrow pointing to the parent class.
  - **Realization/Implementation**: a class implementing an interface — drawn as a dashed line with a hollow triangle arrow.
  - **Multiplicity**: notation like `1`, `0..1`, `1..*`, `0..*` on association ends, indicating how many instances relate to each other (e.g., one `Order` has `1..*` `OrderItems`).
- **Sequence Diagram**: shows the order of interactions/method calls between objects over time for a specific scenario/flow (e.g., the exact sequence of calls when a user places an order: `Controller → OrderService → PaymentService → InventoryService`) — great for clarifying *runtime behavior* and call order, complementing the *static structure* shown by a class diagram.
- **Use Case Diagram**: shows actors (users/external systems) and the use cases (functionalities) they interact with — useful early on for capturing functional requirements at a high level, less about implementation detail.
- **Activity Diagram**: similar to a flowchart — models workflows/business logic with decision points, useful for visualizing an algorithm or process flow (e.g., an order-fulfillment process with branching logic).
- **State Diagram**: models the states an object can be in and the transitions between them (e.g., an `Order`: Created → Paid → Shipped → Delivered, or → Cancelled) — directly maps to the **State design pattern** and is extremely useful for LLD problems that are fundamentally state machines (traffic light, elevator, vending machine, order/booking systems).

**Practical interview tip**: You don't need to draw pixel-perfect UML with exact notation in a live interview — but you should be able to verbally/visually convey classes, their key attributes/methods, the relationships (especially composition vs aggregation vs inheritance), and the sequence of key interactions. A class diagram + a state diagram (if relevant) covers most LLD interview needs.

---

## Step 5: APIs

### 5.1 API Design
- An API defines the contract through which different components/services/clients interact — good API design is central to good LLD, since it's the boundary that determines how easily a system can evolve.
- **REST (Representational State Transfer)** principles: resources identified by URLs (nouns, not verbs — `/orders/123`, not `/getOrder`), standard HTTP verbs carry the action (`GET` read, `POST` create, `PUT`/`PATCH` update, `DELETE` remove), stateless (each request contains all info needed; server holds no client session state between requests), use proper HTTP status codes (`200` OK, `201` Created, `400` Bad Request, `401` Unauthorized, `403` Forbidden, `404` Not Found, `409` Conflict, `500` Internal Server Error).
- **RPC-style / gRPC**: action-oriented calls (`createOrder()`, `getUserProfile()`) rather than resource-oriented — often more natural for internal service-to-service communication; gRPC additionally uses Protocol Buffers (compact binary serialization) and HTTP/2, giving strong typing and better performance than typical JSON-over-REST for internal microservice communication.
- **GraphQL**: client specifies exactly which fields/nested data it needs in a single query, avoiding both **over-fetching** (REST returning more fields than needed) and **under-fetching** (needing multiple REST round trips to assemble related data) — trades off some caching simplicity and adds server-side query complexity/cost-control concerns (a malicious/inefficient query can request deeply nested, expensive data).
- **Idempotency**: an operation is idempotent if calling it multiple times has the same effect as calling it once — critical for retry safety over unreliable networks. `GET`, `PUT`, `DELETE` are expected to be idempotent by convention; `POST` typically is not, so create-type APIs (e.g., "place order") often require an **idempotency key** supplied by the client so a retried request (e.g., due to a timeout where the first request actually succeeded) doesn't create a duplicate resource.
- **Pagination**: for endpoints returning large collections, use limit/offset or (better, for large/changing datasets) cursor-based pagination (return an opaque cursor pointing to the last item seen, avoiding the consistency issues of offset pagination when items are inserted/deleted between page requests).

### 5.2 Request/Response Object Modeling
- Design dedicated **DTOs (Data Transfer Objects)** for API requests/responses, separate from internal domain/database models — this decouples your public API contract from internal implementation details, so internal refactors don't necessarily break API consumers, and lets you control exactly what's exposed (e.g., never expose internal DB IDs, password hashes, or internal-only fields).
- Validate all input at the API boundary (required fields, types, ranges) before it reaches business logic — fail fast with a clear `400 Bad Request` and helpful error details, rather than letting invalid data propagate deeper into the system.
- Design consistent, predictable response envelopes/error formats across all endpoints (e.g., a consistent `{ "error": { "code": ..., "message": ... } }` shape for all errors) so clients can handle responses uniformly.

### 5.3 Versioning & Extensibility
- APIs evolve, but breaking existing clients is costly — versioning strategies let you evolve safely:
  - **URI versioning**: `/v1/orders`, `/v2/orders` — simple, explicit, easy to route, but can lead to duplicated logic across versions if not managed carefully.
  - **Header versioning**: version specified in a custom header (e.g., `Accept: application/vnd.myapi.v2+json`) — keeps URLs clean, but less discoverable/harder to test casually (e.g., in a browser).
  - **Backward-compatible evolution (preferred where possible)**: design changes to avoid needing a new version at all — e.g., only ever **add** new optional fields (never remove or repurpose existing ones), give new fields sensible defaults so old clients unaware of them keep working.
- **Extensibility principles**: design request/response schemas so new optional fields can be added without breaking existing clients (avoid strict/closed schemas that reject unknown fields), use enums/strings that can gain new values gracefully (clients should be built to tolerate/ignore unknown enum values rather than crash), and favor additive changes over destructive ones.
- Deprecation strategy: mark old versions/fields as deprecated with a clear sunset timeline and communicate it, rather than removing support abruptly.

### 5.4 Clean Code Principles: DRY, SRP, etc.
- **DRY (Don't Repeat Yourself)**: avoid duplicating logic/knowledge across the codebase — duplicated logic means duplicated bugs and inconsistent fixes when requirements change. (Balance against over-abstracting too early — premature, overly clever DRY-ing of code that isn't *actually* the same concept just because it looks similar can create the wrong coupling; a little duplication is sometimes better than the wrong abstraction.)
- **SRP** (see SOLID above) — applies at the API/method level too: an endpoint/method should do one clear thing.
- **KISS (Keep It Simple, Stupid)**: prefer the simplest design that satisfies requirements — avoid unnecessary complexity/cleverness that makes code harder to understand and maintain.
- **YAGNI (You Aren't Gonna Need It)**: don't build speculative flexibility/features for hypothetical future requirements that may never materialize — build for today's actual known requirements, and refactor when new requirements genuinely arrive.
- **Meaningful naming**: classes/methods/variables should clearly express intent — this is a major, easy-to-evaluate signal in LLD interviews.
- **Law of Demeter ("don't talk to strangers")**: a method should only call methods on its immediate collaborators, not reach through chains of objects (`a.getB().getC().getD().doSomething()`) — such chains tightly couple you to the internal structure of unrelated objects; expose the needed behavior directly instead.

### 5.5 Avoiding "God Classes"
- A **God Class** is a class that has taken on too many responsibilities — it knows about/does far too much, becomes a dumping ground for unrelated logic, is hard to test, hard to understand, and violates SRP badly (a classic smell: a huge `OrderManager` class that validates input, computes pricing, talks to the DB, sends emails, and handles payment logic, all in one place).
- **How to avoid/fix**: identify the distinct responsibilities mixed together and extract each into its own focused class/service (e.g., split into `OrderValidator`, `PricingCalculator`, `OrderRepository`, `NotificationService`, `PaymentService`, with the original class reduced to orchestrating calls between them).
- Watch for the warning signs during design: a class with a huge number of methods/fields, a class name that's vague ("Manager", "Handler", "Processor" used too broadly without a specific scope), or a class that needs to change for many unrelated reasons.
- This is one of the most commonly directly-tested things in LLD interviews — interviewers frequently probe "what if I also need X?" specifically to see whether your class design already anticipates separation of concerns or needs a rewrite.

---

## Step 6: Common LLD Problems — What Each One Tests

For every LLD problem, follow this approach: (1) clarify functional requirements/scope, (2) identify core entities/nouns → classes, (3) identify relationships (composition/aggregation/inheritance) and draw a class diagram, (4) identify behavior that varies → apply a design pattern (Strategy/State/Observer/Factory are the most common fits), (5) walk through the main flows with method calls (sequence diagram in your head), (6) discuss extensibility ("what if we add X?").

- **Tic-Tac-Toe / Chess**: `Board`, `Piece`/`Cell`, `Player`, `Game` (orchestrator) classes. Chess needs a `Piece` hierarchy (abstract `Piece` with subclasses `King`, `Queen`, `Rook`, etc., each implementing its own `getValidMoves()` — polymorphism in action) and a `Move` validation system. Both benefit from a `GameState`/Strategy pattern for win-condition checking, and the **State pattern** for game phases (InProgress, Check, Checkmate, Draw). Tests: inheritance/polymorphism, encapsulating rules per piece type, avoiding a God `Game` class that does everything.

- **Splitwise (expense sharing)**: `User`, `Expense`, `Group`, `Split` (abstract, with `EqualSplit`, `ExactSplit`, `PercentSplit` subclasses — classic **Strategy pattern**), and a `Balance`/`Ledger` service that tracks who-owes-whom and simplifies debts (a graph/greedy algorithm to minimize the number of settling transactions). Tests: Strategy pattern for split types, SRP (separating `ExpenseService` from `BalanceSheet` computation), extensibility for new split types.

- **Parking Lot**: `ParkingLot`, `Level`, `ParkingSpot` (abstract, sized for `Motorcycle`/`Compact`/`Large` — or a `VehicleType` enum-driven spot-matching strategy), `Vehicle` hierarchy, `Ticket`, and a `ParkingSpotAssignmentStrategy` (Strategy pattern — e.g., nearest-available-spot vs any-available-spot) plus a `PaymentProcessor`. Tests: Factory pattern for creating the right spot/vehicle type, Strategy for spot allocation, handling concurrency (multiple cars arriving simultaneously needing atomic spot assignment — ties back to LLD Step 3 locking).

- **Elevator System (multiple lifts)**: `Elevator` (with a `State`: Idle, MovingUp, MovingDown, DoorOpen — **State pattern**), `ElevatorController` (decides which elevator responds to a request — **Strategy pattern** for the dispatch algorithm, e.g., nearest-elevator vs least-busy), `Request` (internal floor button vs external hall call with direction), and a `Scheduler`. Tests: State pattern for elevator lifecycle, Strategy for the dispatch/scheduling algorithm, concurrency (multiple elevators + multiple simultaneous requests), and a real-time queue of pending requests per elevator.

- **Notification System**: `Notification` (abstract, with `EmailNotification`, `SmsNotification`, `PushNotification` subclasses), `NotificationService` (**Observer pattern** — users subscribe to event types; or a simple publish step to a queue), `NotificationFactory` to construct the right type, and often a **Decorator** for adding features like retry-logging or rate-limiting per notification. Tests: Factory + Strategy/Observer combination, Open/Closed Principle (adding a new channel like WhatsApp shouldn't touch existing channel code), and how this maps to the HLD-level async processing/pub-sub concepts (Step 7 of HLD).

- **Food Delivery App**: `Customer`, `Restaurant`, `MenuItem`, `Order` (a **State machine**: Placed → Confirmed → Preparing → OutForDelivery → Delivered/Cancelled), `DeliveryPartner`, and a `DeliveryAssignmentStrategy` (Strategy pattern — nearest partner, least-busy partner). Tests: State pattern for order lifecycle, Strategy for assignment, Observer for notifying customer/restaurant/partner on state changes, and separating concerns across `OrderService`, `PaymentService`, `DeliveryService` (avoiding a God `OrderManager`).

- **Movie Ticket Booking System**: `Movie`, `Show`(time+screen), `Seat` (with state: Available/Locked/Booked), `Booking`, `Theatre`/`Screen`. The hardest part is **concurrency**: preventing two users from booking the same seat simultaneously — typically solved with a short-lived **seat lock/hold** (pessimistic lock or a TTL-based reservation in a fast store like Redis) while the user completes payment, then converting the hold to a confirmed booking, or releasing it on timeout/failure. Tests: State pattern for seat/booking status, concurrency control (direct application of LLD Step 3 concepts), and the connection to HLD-level strong-consistency requirements (this is a "no double booking" problem, same family as Airbnb/Booking.com from the HLD practice list).

- **URL Shortener**: `UrlMapping` (long ↔ short), `UrlShortenerService` with an ID-generation strategy — options: hash the long URL (risk of collisions, needs collision handling) vs. a counter/auto-increment ID converted to base62 (guarantees uniqueness, simpler) vs. a pre-generated pool of unique keys handed out to servers in batches (avoids a shared counter becoming a bottleneck at high scale — ties to HLD-level distributed ID generation). Tests: Strategy pattern for the ID-generation approach, plus (at HLD scale) caching hot redirects, choosing a DB (simple key-value access pattern → NoSQL or Redis is a natural fit), and analytics tracking as a separate concern (SRP).

- **Logging Framework**: `Logger`, `LogLevel` (enum: DEBUG/INFO/WARN/ERROR), `LogMessage`, and `LogAppender` (abstract, with `ConsoleAppender`, `FileAppender`, `DatabaseAppender` subclasses — **Strategy/Decorator pattern** for output destinations, and a logger can have multiple appenders simultaneously). Often built as a **Singleton** (one logger instance/config per application) with a **Chain of Responsibility** for level filtering (does this log level even need to be processed by this appender?) and a **Builder** for configuring the logger. Tests: Singleton (with thread-safety! — the classic "double-checked locking" singleton implementation question), Strategy for appenders, and asynchronous logging design (producer-consumer: application threads produce log messages, a background thread consumes and writes them, ties to LLD Step 3.3).

- **Rate Limiter**: `RateLimiter` interface with concrete strategies matching the HLD algorithms directly — `TokenBucketRateLimiter`, `SlidingWindowRateLimiter`, `FixedWindowRateLimiter` (**Strategy pattern**), keyed per-user/per-IP (a `Map<Key, RateLimiterState>`). At LLD scale (single machine), this is mostly about correct, thread-safe implementation of the chosen algorithm (atomic operations/locks on the shared counter/bucket state per key); at HLD scale, the same logic needs a shared distributed store (Redis) so multiple servers agree on the count. Tests: Strategy pattern (cleanly swappable algorithms behind one interface), thread-safety of the counter/bucket updates, and explicitly bridging this LLD component to the HLD Rate Limiting concepts from Step 6.4.

**Cross-cutting pattern recognition across LLD problems**: Notice how often the same patterns recur — **State** (game/order/elevator/booking lifecycles), **Strategy** (splits/spot-assignment/dispatch/rate-limit algorithms/ID generation), **Factory** (creating the right subtype of vehicle/notification/piece), **Observer** (notifying interested parties on state changes). Getting comfortable spotting *which* pattern fits *which* kind of variability (varying algorithm → Strategy; varying lifecycle/behavior-by-state → State; varying object creation → Factory; one-to-many change propagation → Observer) is the single highest-leverage LLD interview skill.
