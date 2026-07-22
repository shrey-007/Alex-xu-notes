# Designing a Rate Limiter

## What problem are we solving?

A rate limiter controls how many requests a client can make within a specific period of time.

Example:

* Maximum 100 requests per minute
* Maximum 5 login attempts every 10 minutes
* Maximum 10 OTP requests per hour

If a client crosses the limit, the server rejects the request.

Typical response:

```
HTTP 429 Too Many Requests
```

---

# Why do we need a Rate Limiter?

Without rate limiting, systems become vulnerable to several problems.

### 1. Protect servers from overload

Suppose your server can process

```
10,000 requests/sec
```

One buggy client starts sending

```
100,000 requests/sec
```

CPU reaches 100%.

Memory increases.

Database gets overloaded.

Eventually the entire application crashes.

A rate limiter prevents one client from consuming all resources.

---

### 2. Prevent abuse

Examples:

* Password brute force attacks
* OTP spam
* API scraping
* Inventory hoarding
* Ticket booking bots
* Fake account creation

Instead of allowing unlimited attempts,

```
Allow only

5 login attempts/minute
```

---

### 3. Fair resource sharing

Imagine

1000 users

One user sends

5000 requests

Other users become slow.

Rate limiter ensures everyone gets a fair share.

---

### 4. Reduce infrastructure cost

Every unnecessary request

* uses CPU
* consumes RAM
* occupies network bandwidth
* hits cache
* queries database

Rejecting requests early is much cheaper.

---

### 5. Protect downstream services

Your service may call

* Payment service
* Email service
* SMS provider
* Third-party APIs

If one user floods requests,

those downstream services also fail.

Rate limiting protects the entire chain.

---

# Functional Requirements

The interviewer usually asks this first.

A rate limiter should

* Allow requests under the limit
* Reject requests above the limit
* Work for millions of users
* Be highly available
* Have low latency
* Scale horizontally
* Be configurable

Example

```
Login

5 requests/min

Search

100 requests/min

Upload

20 requests/hour
```

Different APIs may have different limits.

---

# Non Functional Requirements

* Fast (<1 ms decision)
* Highly available
* Fault tolerant
* Consistent enough
* Distributed
* Low memory usage
* Easy to update limits

---

# Basic Flow

```
User

      |

API Gateway

      |

Rate Limiter

      |

Allowed ?

    /     \

 Yes      No

 |          |

Service   HTTP 429
```

Every request first goes through the rate limiter.

Only allowed requests reach the service.

---

# First Design (Single Server)

Suppose we have one server.

```
Client

   |

Server

   |

HashMap

user1 -> 23

user2 -> 56

user3 -> 2
```

Every request

```
counter++
```

If

```
counter > limit

Reject
```

Simple.

Works well.

But only for one machine.

---

# Problems with Single Server

## Problem 1

Server crashes.

Entire rate limiter disappears.

All limits reset.

---

## Problem 2

Need multiple servers.

```
Load Balancer

      |

--------------------

Server A

Server B

Server C
```

Each server has different memory.

User requests may hit different servers.

---

Example

Limit = 100

```
Server A

40 requests

Server B

30 requests

Server C

35 requests
```

Actual

```
105 requests
```

But every server thinks

```
below 100
```

User bypasses the limit.

---

Therefore

Local memory cannot work.

Need centralized storage.

---

# Where should Rate Limiter be placed?

Possible locations

### Option 1

Inside application

```
Request

↓

Application

↓

Rate limiter

↓

Business logic
```

Advantages

Easy.

No extra infrastructure.

Disadvantages

Every service must implement it.

Duplicate code.

Hard to update.

---

### Option 2

Reverse Proxy

```
Internet

↓

NGINX

↓

Application
```

NGINX itself can rate limit.

Good for

* websites
* small APIs

---

### Option 3

API Gateway

Most common.

```
Internet

↓

Load Balancer

↓

API Gateway

↓

Microservices
```

The gateway already handles

* authentication
* logging
* routing
* SSL termination

Adding rate limiting here makes sense.

Every request passes through one place.

No duplicate logic.

Easy to manage.

---

# Designing Centralized Storage

Need one shared counter.

```
Gateway

      |

Redis

      |

Counter
```

Every gateway checks Redis.

Redis stores

```
user123

count = 57
```

Now

No matter which gateway receives the request,

everyone sees the same count.

---

# Why Redis?

Redis is ideal because

* in-memory
* extremely fast
* supports expiration
* supports atomic operations
* distributed

Latency is usually

```
<1 ms
```

Using SQL database would become a bottleneck.

---

# Request Flow

```
Client

↓

Gateway

↓

Redis

↓

Read counter

↓

Increment

↓

Check limit

↓

Allow / Reject
```

Every request performs these operations.

---

# Problem

Suppose

```
Read = 99

Increment

100

Allow
```

Another request

```
Read = 99

Increment

100

Allow
```

Now

101 requests accepted.

Race condition.

---

# Solution

Atomic operations.

Redis provides

```
INCR
```

which performs

```
Read

Increment

Write
```

as one operation.

No race conditions.

---

# Expiration

Need counters to reset.

Example

```
100 requests/minute
```

After one minute

Counter should become zero.

Redis provides TTL.

```
user123

count = 57

TTL = 35 sec
```

When TTL expires

Key disappears automatically.

No cleanup jobs required.

---

# Different Types of Limits

Limit can be applied based on

User ID

```
user123
```

API Key

```
api_key
```

IP Address

```
192.x.x.x
```

Organization

```
Company A
```

Device

```
Phone ID
```

JWT

```
User claim
```

Depends on business requirement.

---

# Configuration Storage

Hardcoding limits is bad.

Instead

```
Database

↓

Configurations

↓

Gateway
```

Example

```
Login

5/min

Search

100/min

Upload

20/hour
```

Gateway periodically reloads configuration.

No restart required.

---

# Multiple API Gateways

Real systems have many gateways.

```
Load Balancer

     |

-------------------------

Gateway 1

Gateway 2

Gateway 3

Gateway 4

      |

Redis Cluster
```

All gateways share Redis.

This provides horizontal scaling.

---

# Redis Becomes Bottleneck

Millions of requests.

Every request

↓

Redis

Eventually Redis itself becomes overloaded.

---

Solutions

## Redis Cluster

Instead of one Redis

```
Gateway

      |

----------------

Redis 1

Redis 2

Redis 3

Redis 4
```

Data is sharded.

Load is distributed.

---

## Replication

One master

Many replicas

```
Master

↓

Replica

↓

Replica
```

Improves read scalability and availability.

Writes still go to the primary.

---

# What if Redis crashes?

Without Redis

Gateway cannot determine

Allowed?

Rejected?

---

Possible approaches

## Fail Open

Allow requests.

Advantages

System continues working.

Disadvantages

Attackers can abuse the system.

Useful for

Low-risk public APIs.

---

## Fail Closed

Reject everything.

Advantages

Protects infrastructure.

Disadvantages

Real users are blocked.

Useful for

Payment APIs

Banking

Sensitive operations.

---

Some systems also maintain a small local fallback cache with recently seen counters or temporarily use conservative local limits until Redis recovers.

---

# Should Gateway Cache Counters Locally?

Possible.

```
Gateway

↓

Local Cache

↓

Redis
```

Advantages

Less Redis traffic.

Lower latency.

Problems

Synchronization.

Different gateways have different cache values.

Usually only small optimizations are cached locally, while the authoritative counter remains in Redis.

---

# Monitoring

Need visibility into rate limiting.

Metrics

* Allowed requests
* Blocked requests
* Redis latency
* Redis errors
* Gateway latency
* Top blocked users
* Top abusive IPs

Without monitoring,

you won't know if the limiter is protecting the system or incorrectly blocking legitimate traffic.

---

# Logging

Log

```
User

IP

Timestamp

Endpoint

Reason

Limit

Current Count
```

Useful for

* debugging
* fraud detection
* audits

---

# Alerts

Alert when

* Redis CPU high
* Redis unavailable
* Block rate suddenly spikes
* Gateway latency increases
* Redis memory almost full

---

# Scaling

As traffic grows

```
Clients

↓

Load Balancer

↓

Many API Gateways

↓

Redis Cluster

↓

Configuration Store
```

No application servers are involved until the request passes rate limiting.

This protects downstream services from unnecessary traffic.

---

# Different Levels of Rate Limiting

A production system often applies multiple limits simultaneously.

```
Global

↓

Organization

↓

User

↓

Endpoint
```

Example

```
Entire company

1,000,000/day

↓

User

1000/hour

↓

Login

5/minute

↓

OTP

3/hour
```

The request is rejected if **any** applicable limit is exceeded.

---

# Common Interview Discussion Points

### Why not store counters in application memory?

Because requests are distributed across multiple servers, leading to inconsistent counts and lost data if a server crashes.

---

### Why Redis instead of SQL?

* Much lower latency
* Atomic increment operations
* Built-in TTL
* Better suited for high-frequency counters

---

### Why place it before application servers?

Rejecting requests early saves CPU, memory, database connections, and network bandwidth.

---

### Why use an API Gateway?

A gateway is a central entry point for all traffic. Implementing rate limiting there avoids duplicate implementations across services and ensures consistent enforcement.

---

### Why use TTL?

It automatically resets counters after the configured time window without requiring cleanup jobs.

---

### Why support dynamic configuration?

Business requirements change frequently. Operations teams should be able to modify limits without redeploying applications.

---

# High-Level Architecture

```
                        Clients
                           |
                    Load Balancer
                           |
             +-------------+-------------+
             |             |             |
        API Gateway   API Gateway   API Gateway
             |             |             |
             +-------------+-------------+
                           |
                     Redis Cluster
                           |
                Configuration Store
                           |
                    Monitoring & Logs
                           |
                    Backend Services
```

---

# Design Summary

1. Every client request first reaches the API Gateway.
2. The gateway identifies the client (user ID, API key, IP, etc.).
3. The gateway checks the configured limit for the requested API.
4. The gateway performs an atomic counter update in Redis.
5. Redis returns the updated count.
6. If the count is within the configured limit, the request is forwarded to the backend service.
7. If the limit is exceeded, the gateway immediately returns **HTTP 429 Too Many Requests**.
8. Redis automatically expires counters using TTL, resetting limits for the next time window.
9. Configuration is managed separately so limits can be changed without code deployment.
10. Monitoring, logging, and alerting ensure the rate limiter remains reliable, scalable, and easy to operate in production.

This architecture is the standard high-level design used in large-scale systems. The specific rate-limiting algorithm (Token Bucket, Leaky Bucket, Fixed Window, Sliding Window, etc.) is an implementation detail that plugs into this architecture without changing the overall system design.

