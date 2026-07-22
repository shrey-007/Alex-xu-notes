# These are the notes for the chapter 6 -: Designing a key-value pair database

This is probably **the most confusing chapter** in *System Design Interview* because it introduces 10-15 different concepts one after another. Most people understand each concept individually, but don't understand **why each one suddenly appears**.

The trick is to stop thinking of it as **15 different topics**, and instead think of it as **one story**.

I'll explain it as if we are building Redis/DynamoDB from scratch.

---

# Stage 1 - We start with the simplest database

Suppose someone asks you to build this API.

```text
put("name", "Shrey")
get("name")
```

The easiest implementation is

```text
HashMap<String, Object>
```

```
+--------------------+
| HashMap            |
|                    |
| name -> Shrey      |
| age  -> 23         |
| city -> Bangalore  |
+--------------------+
```

Everything is stored in RAM.

Reading

```
get("name")
```

is O(1).

Life is beautiful.

---

## Problem #1

The server memory fills up.

Suppose

```
RAM = 32 GB

Data = 20 TB
```

Impossible.

So now we need multiple machines.

```
Client

      |

+-------+
|Server1|
+-------+

+-------+
|Server2|
+-------+

+-------+
|Server3|
+-------+
```

This is the **real beginning** of the chapter. 

---

# Stage 2 - How do we divide data?

Question:

```
Where should key "abc" go?

Server1?
Server2?
Server3?
```

We need a rule.

Naive answer

```
hash(key) % number_of_servers
```

Example

```
hash("abc") % 3

= Server2
```

Works.

Then someone adds another server.

Now

```
hash(key)%4
```

Every key moves.

Millions of records need copying.

Very bad.

---

## Solution

Consistent Hashing.

Instead of

```
modulo
```

we use a hash ring.

```
           S1

      keyA

 S4              S2

      keyB

           S3
```

Rule:

Hash the key.

Move clockwise.

First server wins.

Now if one server is added

```
Only nearby keys move.
```

Instead of

```
100%
```

maybe only

```
5%
```

move.

This solves **partitioning**. 

---

# Stage 3 - Server dies

Imagine

```
keyA

↓

Server2
```

Server2 explodes.

Now

```
get(keyA)
```

returns

```
404
```

Bad.

So we store copies.

```
Server1

Server2

Server3
```

Every key exists on

```
N servers
```

Example

```
N=3
```

```
keyA

↓

S2
S3
S4
```

Now if

```
S2
```

dies,

S3 still has data.

This is **Replication**. 

---

# Stage 4 - Replication creates another problem

Now we have

```
S2

name = Shrey
```

and

```
S3

name = Shrey
```

Client updates

```
name = John
```

Suppose

```
S2 updated

S3 not updated yet
```

Now

```
Client A

↓

S2

John
```

Client B

↓

S3

Shrey

```

Different answers.

Now we need to answer

> "When should a write be considered successful?"

This introduces

```

N
W
R

```

---

# Stage 5 - N, W, R

Suppose

```

N = 3

```

Copies are

```

S1
S2
S3

```

---

## Write quorum

Suppose

```

W=2

```

Client writes

```

John

```

Coordinator sends write

```

S1 ✓

S2 ✓

S3 pending

```

Two acknowledgements received.

Return success.

Even though S3 isn't done.

---

## Read quorum

Suppose

```

R=2

```

Coordinator asks

```

S1

S2

S3

```

It waits until any 2 respond.

Now compare versions.

Return latest.

---

### Important formula

```

W + R > N

```

means

There is always at least one replica that participated in both the write and the read.

Therefore, the read is guaranteed to observe the latest write (assuming no other failures beyond the quorum model). :contentReference[oaicite:3]{index=3}

---

# Stage 6 - CAP theorem enters

Now imagine

```

S1

S2

S3

```

Suddenly

```

S3

```

cannot talk to

```

S1
S2

```

Network partition.

Now what?

Option A

Stop writes.

Everyone sees same data.

Consistency.

Users unhappy.

---

Option B

Keep accepting writes.

Users happy.

But

```

S3

```

has old data.

Availability.

This is exactly

```

CAP theorem

```

The chapter introduces CAP here because once data is distributed across machines, network partitions become unavoidable. You must choose how the system behaves during those failures. :contentReference[oaicite:4]{index=4}

---

# Stage 7 - Eventual consistency

Suppose

```

S1

John

```
```

S2

John

```
```

S3

Shrey

```

Eventually

```

S3

↓

John

```

after synchronization.

Eventually

everyone agrees.

That's Eventual Consistency. :contentReference[oaicite:5]{index=5}

---

# Stage 8 - Simultaneous writes

Now imagine

Original

```

name = John

```

User A

```

John Delhi

```

User B

```

John Mumbai

```

Both happen simultaneously.

Which one wins?

No answer.

Now we need

```

Versioning

```

---

# Stage 9 - Vector clocks

Instead of storing

```

John

```

we store

```

John

Version

(S1,2)

```

Another server updates

```

(S1,2)

↓

(S1,2)(S2,1)

```

Another update

```

(S1,2)(S3,1)

```

Now we immediately know

```

Conflict.

```

The client (or application) must merge them.

Vector clocks do not resolve conflicts automatically; they **detect** them and preserve enough history to merge safely later. :contentReference[oaicite:6]{index=6}

---

# Stage 10 - Servers die

Now

```

S2

```

crashes.

How do others know?

They keep sending

```

Heartbeat

```

"I'm alive."

If heartbeat stops

↓

Gossip spreads

↓

Everyone marks S2 dead.

This is **Gossip Protocol**. :contentReference[oaicite:7]{index=7}

---

# Stage 11 - What if a replica is temporarily down?

Suppose

```

Need 3 replicas

S2 dead

```

Instead of failing

temporarily store copy on

```

S4

```

Later

```

S2 returns

↓

S4 sends data back

```

This is

```

Hinted Handoff

```

---

# Stage 12 - Replica missed thousands of writes

Now

```

S2

```

was offline for

```

2 days

```

How do we sync?

Don't copy

```

Entire database

```

Instead

Compare hashes.

That's

```

Merkle Tree

```

It tells exactly

```

Bucket 17 differs.

Only sync bucket 17.

```

instead of

```

Sync 20 TB.

```

This is anti-entropy repair using Merkle trees. :contentReference[oaicite:8]{index=8}

---

# Stage 13 - Final architecture

Now everything comes together.

```

```
           Client
              |
              |
       Coordinator
              |
    -------------------
    |        |        |
   S1       S2       S3
    \       |       /
     \------|------/
         Replicas
```

```

Each node

- stores data
- replicates
- gossips
- repairs
- participates in quorum
- joins/leaves using consistent hashing

No master node.

Everything is decentralized. :contentReference[oaicite:9]{index=9}

---

# Stage 14 - How writes and reads actually happen

### Write

```

Client

↓

Coordinator

↓

Commit Log
↓
Memory Table
↓
SSTable (disk)

```

Why?

- **Commit Log**: don't lose data if the server crashes immediately after the write.
- **Memory Table (MemTable)**: keep writes fast by writing to RAM.
- **SSTable**: periodically flush sorted data from memory to disk for efficient storage and reads. :contentReference[oaicite:10]{index=10}

### Read

```

Client

↓

Memory

↓

Found?

```

Yes

↓

Return

No

↓

Bloom Filter

↓

Which SSTable?

↓

Read Disk

↓

Return
```

The Bloom filter avoids checking every SSTable on disk by quickly ruling out files that definitely don't contain the key. 

---

# The entire chapter in one picture

```text
Need key-value store
        │
        ▼
One server
        │
        ▼
Memory full
        │
        ▼
Multiple servers
        │
        ▼
How divide data?
        │
        ▼
Consistent Hashing
        │
        ▼
Server dies
        │
        ▼
Replication
        │
        ▼
Replicas become different
        │
        ▼
Quorum (N, W, R)
        │
        ▼
Network partitions
        │
        ▼
CAP theorem
        │
        ▼
Eventually synchronize
        │
        ▼
Eventual Consistency
        │
        ▼
Concurrent writes
        │
        ▼
Vector Clocks
        │
        ▼
Node failures
        │
        ▼
Gossip Protocol
        │
        ▼
Temporary failure
        │
        ▼
Hinted Handoff
        │
        ▼
Long-term repair
        │
        ▼
Merkle Trees
        │
        ▼
Efficient storage
        │
        ▼
Commit Log + MemTable + SSTables + Bloom Filter
```

This flow is the "connecting thread" that the book doesn't make explicit. Every new concept is introduced because the previous solution creates a new problem. Once you view the chapter as a sequence of **problem → solution → new problem**, the design becomes much easier to remember.
 
