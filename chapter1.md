# Scale From Zero to Millions of Users

This chapter is about **how a system evolves as traffic increases**. You never build the architecture for 100 million users on Day 1 because it would be unnecessarily expensive, difficult to maintain, and full of unused components. Instead, you start with the simplest architecture that solves today's problems, then add components only when new bottlenecks appear. Every step in this chapter exists because the previous architecture eventually fails. 

---

# Stage 1: Single Server Architecture

The first version of almost every startup is a **single machine**. Everything runs on one server:

* Web server
* Backend application
* Database
* Cache
* Static files
* Business logic

```
                Users
                  |
             DNS Lookup
                  |
              Single Server
        -------------------------
        Web + App + DB + Cache
```

## Why start with a single server?

Because initially:

* Very few users
* Very little traffic
* Very little data
* One developer or a small team
* Low budget

Adding distributed systems at this stage only increases complexity without solving any real problem.

---

## Request Flow

Suppose a user opens

```
api.mysite.com
```

### Step 1

Browser asks DNS

```
Where is api.mysite.com?
```

DNS returns

```
15.125.23.214
```

DNS is usually provided by third-party providers.

---

### Step 2

Browser sends HTTP request

```
GET /users/12
```

to

```
15.125.23.214
```

---

### Step 3

Application receives request.

Business logic runs.

Example

```
Find User 12
```

Application queries database.

---

### Step 4

Database returns data.

Application converts it into JSON.

Browser receives

```json
{
   "id":12,
   "name":"Shrey"
}
```

Request complete.

---

## Why is this good?

Simple.

Easy debugging.

Cheap.

Easy deployment.

One server.

One log.

One process.

---

## Problems

Eventually users increase.

Now everything competes for the same resources.

Example

Database query uses CPU.

Web server needs CPU.

Cache needs RAM.

OS needs RAM.

Eventually

```
CPU = 100%
RAM = Full
Disk IO = High
```

Everything slows down.

One machine cannot scale forever.

If server dies

Everything dies.

Entire company becomes unavailable.

This becomes our first bottleneck. 

---

# Stage 2: Separate Database

Instead of keeping everything together

Split into

```
Web Server
```

and

```
Database Server
```

```
Users

     |

 Web Server

     |

 Database
```

---

## Why?

Application and database have completely different workloads.

### Web server

Needs

* CPU
* Threads
* Network

### Database

Needs

* RAM
* Disk
* Storage optimization

Scaling them together wastes resources.

---

## Benefits

Now

If application becomes busy

Upgrade only web server.

If database becomes huge

Upgrade only database.

Independent scaling.

Much better utilization.

---

# SQL vs NoSQL

Now comes another question.

Which database?

---

## SQL

Examples

* MySQL
* PostgreSQL
* Oracle

Stores

```
Rows

Columns
```

Supports

```
JOIN

Transactions

ACID
```

Best when relationships exist.

Example

```
Users

Orders

Payments

Products
```

Need joins.

SQL wins.

---

## NoSQL

Examples

* Cassandra
* DynamoDB
* CouchDB
* MongoDB

Stores

```
JSON

Key Value

Documents

Columns

Graphs
```

Best when

* Huge data
* Flexible schema
* Very low latency
* Massive scaling
* Few joins

Example

User sessions

Cache

Analytics

Logs

Social feeds

---

NoSQL sacrifices relational features to gain scalability and flexibility. 

---

# Stage 3: Vertical Scaling

Traffic increases.

Current server becomes slow.

Simplest solution

Buy larger hardware.

```
8 GB RAM

↓

64 GB RAM
```

```
4 CPU

↓

32 CPU
```

This is

```
Scale Up
```

---

## Why?

No architecture changes.

No code changes.

Very easy.

---

## Problems

Hardware has limits.

Eventually

```
Largest machine available
```

Still isn't enough.

Very expensive.

Single point of failure.

If machine dies

Everything dies.

Not suitable for large companies. 

---

# Stage 4: Horizontal Scaling

Instead of buying bigger servers

Buy more servers.

```
Server 1

Server 2

Server 3

Server 4
```

Now users must be distributed among them.

Question

Who decides where requests go?

Answer

Load Balancer.

---

# Stage 5: Load Balancer

```
Users

     |

 Load Balancer

 /    |     \

S1    S2    S3
```

Users never directly contact servers.

They contact load balancer.

Load balancer forwards request.

---

## Why?

Without load balancer

```
10000 users

↓

One Server
```

Server crashes.

Other servers remain idle.

Terrible resource utilization.

---

Load balancer distributes requests.

Example

Round Robin

```
Req1 → S1

Req2 → S2

Req3 → S3
```

Every server receives similar load.

---

## Benefits

Higher throughput.

Fault tolerance.

Easy scaling.

If one server dies

Traffic automatically shifts.

```
S2 dies

↓

Only S1 and S3 receive traffic
```

Application stays alive.

Huge improvement in availability.

Also, backend servers can use **private IPs**, reducing their exposure to the public internet. 

---

# Stage 6: Database Replication

Web tier is now redundant.

Database is still one machine.

If database dies

Whole application dies.

Need replication.

---

Architecture

```
            Master

          /    |    \

     Slave  Slave  Slave
```

---

## Master

Handles

```
INSERT

UPDATE

DELETE
```

---

## Slaves

Handle

```
SELECT
```

---

## Why?

Most applications

```
Reads >> Writes
```

Example

Instagram

Millions of profile views

Very few profile edits

Reads dominate.

Perfect for replicas.

---

## Benefits

Read scaling.

High availability.

Reliability.

Backup.

Disaster recovery.

---

## Failure Scenarios

### Slave dies

Traffic shifts to other slaves.

Eventually create new slave.

---

### Master dies

Promote slave.

Slave becomes master.

Problem:

Slave may lag behind because replication is often asynchronous.

Need recovery scripts or advanced replication strategies to catch up. 

---

# Stage 7: Cache

Even with replicas

Database is still repeatedly asked the same questions.

Example

```
User Profile

100,000 requests

Same data
```

Database executes same query repeatedly.

Wasteful.

Solution

Cache.

---

Flow

```
Request

↓

Cache?

↓

Yes

↓

Return

↓

No

↓

Database

↓

Save in Cache

↓

Return
```

This is Read Through Cache.

---

## Why?

Memory

```
~100x faster

than disk
```

Huge latency reduction.

Huge database load reduction.

---

## Problems

### Cache Expiration

Without TTL

Old data remains forever.

Users see stale information.

With very short TTL

Frequent cache misses.

Database overloaded again.

Need balance.

---

### Consistency

Database updated.

Cache still old.

Users see incorrect data.

Need cache invalidation or update strategies.

---

### Single Cache Server

If cache dies

Everything falls back to database.

Database suddenly overloaded.

Solution

Multiple cache servers.

---

### Full Cache

Need eviction.

Popular policies

LRU

LFU

FIFO

Choose based on access patterns. 

---

# Stage 8: CDN

Static files

```
Images

Videos

CSS

JS
```

Don't change often.

No reason for application servers to serve them.

Move to CDN.

---

## Why?

User in India shouldn't download an image from a US server if there's a nearby copy.

CDNs cache static assets at edge locations close to users, reducing latency and unloading origin servers.

If the asset isn't cached, the CDN fetches it from the origin, stores it for the configured TTL, and serves future requests from the edge. Handle cache expiry, invalidation (or versioned URLs), cost, and origin fallback for CDN outages. 

---

# Stage 9: Stateless Web Tier

A common mistake is storing user session data in the web server's memory.

Then every request from that user must go to the same server (sticky sessions).

This complicates scaling and recovery.

Move session state to shared storage (database, Redis, or another distributed store).

Now any web server can handle any request because state is externalized.

Benefits:

* Easy horizontal scaling
* Simpler load balancing
* Better fault tolerance
* Autoscaling becomes practical because servers are interchangeable. 

---

# Stage 10: Multiple Data Centers

When users are spread across regions, one data center creates high latency and becomes a regional single point of failure.

Deploy multiple data centers.

Use GeoDNS to send users to the nearest healthy location.

Challenges include:

* Correct traffic routing
* Replicating data across regions
* Handling failover when one region goes down
* Keeping deployments consistent everywhere

Multi-region systems improve both latency and availability but introduce distributed data consistency challenges. 

---

# Stage 11: Message Queue

Some tasks don't need an immediate response.

Examples:

* Sending email
* Processing images
* Generating PDFs
* Notifications

Instead of doing them during the request, publish a message to a queue.

```
User

↓

Web Server

↓

Message Queue

↓

Worker
```

The web server returns quickly while background workers process the job asynchronously.

Benefits:

* Decouples producers and consumers
* Absorbs traffic spikes
* Workers scale independently
* Jobs survive temporary service outages because they're buffered in the queue. 

---

# Stage 12: Logging, Metrics, and Automation

As the system grows, operating it becomes as important as writing code.

You need:

* **Logging** to diagnose failures.
* **Metrics** (CPU, memory, latency, business KPIs) to understand system health.
* **Automation** (CI/CD, testing, deployment) to reduce manual errors and increase developer productivity. 

---

# Stage 13: Database Sharding

Eventually even a replicated database becomes too large.

One machine cannot store or process everything efficiently.

Split the data across multiple database servers (shards).

Example:

```
user_id % 4

0 → Shard 0

1 → Shard 1

2 → Shard 2

3 → Shard 3
```

The routing function (sharding key) determines which shard stores a given user's data.

A good sharding key distributes data evenly and allows efficient routing.

Challenges include:

* **Resharding:** moving data when shards fill up or distribution becomes uneven.
* **Hotspots (celebrity problem):** one shard receives disproportionate traffic.
* **Cross-shard joins:** joins become difficult, often requiring denormalization or application-level aggregation.

Sharding is one of the final steps in scaling because it significantly increases operational complexity, but it allows databases to continue growing beyond the limits of a single machine. 

