# System Design — Starter to Deep
*From "what is system design" to designing production systems*

---

# Part 1: What Is System Design?

## The Simplest Explanation

System design is the art of deciding HOW to build a large application. Not the code, but the architecture — what components exist, how they talk to each other, and what happens when things go wrong or get busy.

```
Coding interview:    "Write a function that does X"
                     → Tests: can you CODE?

System design:       "Design Instagram / URL shortener / chat app"
                     → Tests: can you ARCHITECT?
```

**Real-life analogy:** Coding is like building a single room. System design is planning an entire city — where do roads go, how does water reach every house, what happens during a flood, how do you handle 10x population growth?

## Why It Matters

Every production application needs to answer:
- How do I handle 1000 users? 1 million? 100 million?
- What happens when a server crashes?
- Where does the data live? How is it backed up?
- How fast does the user get a response?
- How do I deploy updates without downtime?

---

# Part 2: Core Building Blocks

## 1. Client-Server Model

```
┌──────────┐         HTTP Request          ┌──────────┐
│  Client  │ ───────────────────────────▶  │  Server  │
│ (Browser,│                               │ (Node.js,│
│  Mobile  │  ◀───────────────────────────  │  FastAPI)│
│  App)    │         HTTP Response          │          │
└──────────┘                               └──────────┘

Client: shows UI, sends requests
Server: processes requests, stores data, returns responses
```

## 2. DNS (Domain Name System)

```
User types: www.myapp.com

Step 1: Browser → DNS Server: "What's the IP for www.myapp.com?"
Step 2: DNS Server → Browser: "It's 54.23.100.77"
Step 3: Browser → 54.23.100.77: "GET /index.html"

DNS is like a phone book for the internet.
```

## 3. Load Balancer

**Problem:** One server can handle ~1000 concurrent requests. What if you have 100,000?

**Solution:** Put multiple servers behind a load balancer. It distributes requests across servers.

```
                    ┌──────────────┐
Requests ──────▶   │Load Balancer │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         ┌────────┐   ┌────────┐   ┌────────┐
         │Server 1│   │Server 2│   │Server 3│
         └────────┘   └────────┘   └────────┘

Strategies:
  Round Robin:   1→S1, 2→S2, 3→S3, 4→S1, 5→S2...
  Least Connections: send to the server with fewest active connections
  IP Hash:       same client always goes to same server (session affinity)
  Weighted:      more powerful servers get more traffic
```

**AWS equivalent:** Application Load Balancer (ALB)

## 4. Web Server vs Application Server

```
Web Server (Nginx):
  • Serves static files (HTML, CSS, JS, images)
  • Acts as reverse proxy (forwards dynamic requests to app server)
  • Handles SSL termination (HTTPS → HTTP internally)
  • Handles compression (gzip)

Application Server (Node.js / FastAPI):
  • Runs your business logic
  • Handles database queries
  • Processes API requests
  • Runs behind the web server/load balancer
```

## 5. Database

```
Relational (SQL):                     Non-Relational (NoSQL):
  PostgreSQL, MySQL                     MongoDB (documents)
  Tables with rows and columns          Redis (key-value)
  Fixed schema                          Elasticsearch (search)
  ACID transactions                     Cassandra (wide-column)
  JOINs between tables                  Neo4j (graph)
  Best for: structured data,            Best for: flexible schema,
    complex queries, financial data       high write volume, real-time

When to choose:
  "I need transactions and JOINs"        → SQL (PostgreSQL)
  "My data shape changes frequently"     → MongoDB
  "I need caching / sessions"            → Redis
  "I need full-text search"              → Elasticsearch
  "I need to store relationships"        → Neo4j (or SQL with JOINs)
```

## 6. Cache

**Problem:** Database queries take 50-100ms. For data that doesn't change often, that's wasteful.

**Solution:** Store frequently accessed data in a cache (Redis). Cache reads take <1ms.

```
Without cache:
  Client → Server → Database (50ms) → Server → Client
  Total: ~60ms

With cache:
  Client → Server → Cache (0.5ms) → Server → Client
  Total: ~5ms   (10x faster!)

  Cache miss: Server → Cache (miss) → Database → Store in cache → Client
```

```
Caching strategies:
  Cache-aside (lazy):
    Read: check cache → if miss, read DB, store in cache
    Write: write to DB, invalidate cache
    Most common pattern.

  Write-through:
    Write: write to cache AND DB simultaneously
    Read: always read from cache
    Data is always fresh but writes are slower.

  Write-behind:
    Write: write to cache, async write to DB later
    Fast writes but risk of data loss if cache crashes.
```

## 7. CDN (Content Delivery Network)

```
Without CDN:
  User in Tokyo → Server in Virginia (200ms latency each way)

With CDN:
  User in Tokyo → CDN edge in Tokyo (5ms)

CDN caches static content (images, JS, CSS) at 400+ locations worldwide.
User gets content from the nearest edge server.

Examples: CloudFront, Cloudflare, Akamai
```

## 8. Message Queue

**Problem:** User uploads a PDF. Processing takes 30 seconds. You can't make them wait.

**Solution:** Put the task in a queue. Return "processing" immediately. A worker picks it up later.

```
Synchronous (bad):
  User → Upload PDF → [30 seconds processing] → Response
  User waits 30 seconds!

Asynchronous with queue (good):
  User → Upload PDF → Put in queue → Response "Processing!"  (instant)
                            ↓
                      Worker picks up → processes PDF → notifies user when done

Queue examples: Redis (BullMQ), RabbitMQ, SQS, Kafka
```

```
When to use:
  ✓ Tasks that take more than a few seconds (video processing, ML inference)
  ✓ Tasks that can fail and need retry
  ✓ Decoupling services (service A doesn't need to wait for service B)
  ✓ Smoothing traffic spikes (queue absorbs bursts)
```

---

# Part 3: Scaling Concepts

## Vertical vs Horizontal Scaling

```
Vertical scaling (scale UP):
  Get a bigger machine. More CPU, more RAM.
  ┌──────────┐       ┌──────────────────┐
  │ 2 CPU    │  →    │ 32 CPU           │
  │ 4GB RAM  │       │ 128GB RAM        │
  └──────────┘       └──────────────────┘
  Simple but has a ceiling. One machine can only get so big.

Horizontal scaling (scale OUT):
  Get more machines. Distribute the load.
  ┌──────────┐       ┌────┐ ┌────┐ ┌────┐ ┌────┐
  │ 1 server │  →    │ S1 │ │ S2 │ │ S3 │ │ S4 │
  └──────────┘       └────┘ └────┘ └────┘ └────┘
  No ceiling but more complex (load balancing, stateless design, data sync).
```

## Stateless vs Stateful Servers

```
STATEFUL server (can't scale horizontally):
  Server stores user sessions in memory.
  User 1 → Server A (session stored in Server A's memory)
  If Load Balancer sends User 1 to Server B → session is lost!

STATELESS server (scales horizontally):
  Server stores NOTHING in memory. Everything is in Redis/DB.
  User 1 → Server A (reads session from Redis)
  User 1 → Server B (reads session from Redis — same data!)

RULE: For horizontal scaling, servers MUST be stateless.
  No in-memory sessions → use Redis
  No in-memory caches → use Redis
  No file uploads to local disk → use S3
```

## Database Scaling

```
Step 1: Read Replicas
  Problem: too many reads.
  Solution: primary DB handles writes, replicas handle reads.
  
  ┌──────────┐     replication     ┌──────────┐
  │ Primary  │ ──────────────────▶ │ Replica  │
  │ (writes) │                     │ (reads)  │
  └──────────┘                     └──────────┘

Step 2: Caching
  Problem: still too many reads.
  Solution: put Redis in front of the database. Cache popular queries.

Step 3: Sharding
  Problem: too much data for one database.
  Solution: split data across multiple databases.
  
  Users A-M → Database 1
  Users N-Z → Database 2
  
  Each shard holds a subset of the data.
  Complication: queries across shards are hard (JOIN across databases).
```

---

# Part 4: Key Concepts

## CAP Theorem

You can only guarantee 2 out of 3:
```
C — Consistency:    every read returns the most recent write
A — Availability:   every request gets a response (even if data is stale)
P — Partition Tolerance: system works even if network between nodes fails

Since network partitions WILL happen, you really choose between:
  CP: consistent but may be unavailable during partition (MongoDB, HBase)
  AP: available but may return stale data during partition (Cassandra, DynamoDB)
  
Most web apps choose AP (it's better to show slightly stale data than to show nothing).
```

## Latency Numbers Every Programmer Should Know

```
Operation                          Time
─────────                          ────
L1 cache reference                 1 ns
L2 cache reference                 4 ns
RAM reference                      100 ns
Read 1 MB from RAM                 3 μs
SSD read                           100 μs
Read 1 MB from SSD                 1 ms
Network round trip (same DC)       500 μs
Read 1 MB from network             10 ms
HDD seek                           10 ms
Network round trip (cross-country) 30-50 ms
Network round trip (cross-ocean)   100-200 ms

Takeaway:
  Memory > SSD > Network > Disk
  Cache everything you can.
  Minimize network round trips (batch requests, CDN).
  Keep data close to computation (same region, same data center).
```

## API Design

```
REST API:
  Resources are nouns: /users, /orders, /products
  Actions are HTTP methods: GET, POST, PUT, DELETE
  Stateless: each request contains all info needed
  
  GET    /api/users          → list users
  POST   /api/users          → create user
  GET    /api/users/:id      → get user
  PUT    /api/users/:id      → update user
  DELETE /api/users/:id      → delete user

GraphQL:
  Single endpoint: POST /graphql
  Client specifies exactly what data it wants
  No over-fetching or under-fetching
  
  query {
    user(id: "1") {
      name
      email
      posts { title }
    }
  }

gRPC:
  Binary protocol (Protocol Buffers) — much faster than JSON
  Strongly typed contracts (.proto files)
  Bidirectional streaming
  Use for: service-to-service communication (not client-facing)
```

---

# Part 5: System Design Interview Framework

## The 4-Step Process (Use This Every Time)

### Step 1: Clarify Requirements (2-3 min)

```
"Before I design, let me make sure I understand the requirements."

Functional requirements (what does it DO?):
  "Users can upload photos"
  "Users can follow other users"
  "Users see a feed of posts from people they follow"

Non-functional requirements (how well does it do it?):
  Scale:     How many users? How many requests/sec?
  Latency:   How fast should responses be?
  Availability: How much downtime is acceptable?
  Consistency: Is eventual consistency OK?
```

### Step 2: High-Level Design (10-15 min)

```
Draw the big boxes:
  Client → Load Balancer → API Servers → Database
  Add: CDN, Cache, Message Queue as needed

"Let me start with the core architecture and then dive deeper."
```

### Step 3: Detailed Design (10-15 min)

```
Pick the most critical or interesting component and go deep:
  - Database schema
  - API endpoints
  - Data flow for a specific use case
  - Caching strategy
  - How the feed is generated
```

### Step 4: Address Bottlenecks and Trade-offs (5 min)

```
"Now let me think about what could go wrong."
  - Single points of failure?
  - What if traffic 10x?
  - What if the database is slow?
  - What are the trade-offs of my design?
```

---

# Part 6: Common System Design Problems

## Design a URL Shortener (Easy)

```
Functional: given long URL, generate short URL. Short URL redirects to long URL.

High-level:
  Client → API Server → Database
                ↓
            Short URL generation

Database schema:
  urls:
    id: auto-increment
    short_code: "abc123" (unique, indexed)
    long_url: "https://very-long-url.com/..."
    created_at: timestamp
    click_count: integer

API:
  POST /shorten  { url: "https://..." }  → { short_url: "https://short.ly/abc123" }
  GET  /:code    → 301 redirect to long URL

Short code generation:
  Option 1: Base62 encode the auto-increment ID (1→"1", 62→"10", 3844→"100")
  Option 2: Random 6-char string, check for collision
  Option 3: Hash the URL (MD5/SHA), take first 6 chars

Scale:
  Cache popular URLs in Redis (99% of clicks go to 1% of URLs)
  Read replicas for database
  CDN for static redirect page
```

## Design a Chat Application (Medium)

```
Functional: 1:1 messaging, group chats, online status, read receipts.

Key decisions:
  Protocol: WebSocket (bidirectional, real-time)
  
Architecture:
  ┌────────┐    WebSocket    ┌───────────────┐
  │ Client │◄───────────────▶│  Chat Server  │
  └────────┘                 │ (WebSocket    │
                             │  Gateway)     │
                             └───────┬───────┘
                                     │
                             ┌───────▼───────┐
                             │ Message Queue │ (Kafka/Redis)
                             └───────┬───────┘
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
              ┌──────────┐    ┌──────────┐    ┌──────────┐
              │ MongoDB  │    │  Redis   │    │  S3      │
              │(messages)│    │(sessions,│    │(media)   │
              │          │    │ presence)│    │          │
              └──────────┘    └──────────┘    └──────────┘

Message flow:
  User A sends → WebSocket Gateway → check if User B is online
  If online: forward directly via WebSocket
  If offline: store in DB, send push notification

Group chat: fan-out — message goes to ALL members' connections
  For large groups (1000+ members): use message queue to avoid overloading
```

## Design a News Feed (Hard)

```
Functional: users follow others, see posts in chronological order.

Two approaches:

PUSH (Fan-out on write):
  When User A posts → immediately write to ALL followers' feeds
  Pro: feed reads are instant (pre-computed)
  Con: celebrity with 10M followers → 10M writes per post (expensive)

PULL (Fan-out on read):
  When User B opens feed → query all followed users' recent posts → merge and sort
  Pro: writes are instant (one write per post)
  Con: feed reads are slow (query N users, merge, sort)

HYBRID (what Twitter/Instagram use):
  For regular users (< 10K followers): PUSH (fast read, manageable write)
  For celebrities (> 10K followers): PULL (on-demand when fan opens feed)

Storage:
  Posts table: id, user_id, content, media_url, created_at
  Feed table: user_id, post_id, created_at (pre-computed feed for push model)
  Follows table: follower_id, followee_id

  Feed table indexed by (user_id, created_at DESC) for fast feed retrieval.
  Cache hot feeds in Redis (most active users' feeds).
```

---

# Part 7: 🧩 Interview Q&A

**Q: How would you handle a service going down?**
A: Redundancy — run multiple instances behind a load balancer. If one dies, the load balancer routes traffic to healthy ones. Health checks detect failures. Auto-scaling replaces dead instances. For databases: replicas with automatic failover. For the whole system: deploy across multiple availability zones (AZs).

**Q: How do you ensure data consistency between services?**
A: For strong consistency: use distributed transactions (2PC) or event sourcing with a single source of truth. For eventual consistency: use event-driven architecture — Service A publishes an event, Service B consumes it and updates its own data. Saga pattern for multi-step transactions with compensation (undo steps on failure).

**Q: SQL vs NoSQL — how do you decide?**
A: SQL when you need ACID transactions, complex JOINs, and your schema is stable (financial data, user accounts). NoSQL when you need flexible schema, horizontal scaling, high write throughput, or your data is naturally document-shaped (user profiles, product catalogs, real-time analytics). Many systems use BOTH — SQL for transactional data, Redis for caching, Elasticsearch for search.

**Q: How would you handle 10x traffic spike?**
A: Short-term: auto-scaling adds more servers. Caching reduces DB load. Rate limiting protects the system. Queue absorbs burst writes. Long-term: CDN for static content, database read replicas, sharding for data distribution, async processing for heavy tasks.
