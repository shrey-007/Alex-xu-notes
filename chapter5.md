# Why this chapter exists

The previous chapters taught how to scale a system by adding more machines. But adding machines introduces a new problem:

**How do you decide which server stores which data?**

If you answer this incorrectly, every time you add or remove a server, almost all data moves between servers, making scaling useless.

Consistent Hashing exists to solve exactly one problem:

> **Distribute data across servers so that adding or removing servers moves only a small amount of data.**

Everything in this chapter builds toward solving that single problem.

The logical flow is:

```
Need multiple servers
        ↓
Need data distribution
        ↓
Traditional hashing
        ↓
Rehashing problem
        ↓
Need better algorithm
        ↓
Consistent Hashing
        ↓
Hash Ring
        ↓
Server Lookup
        ↓
Add/Remove Server
        ↓
Only few keys move
        ↓
Still has imbalance
        ↓
Virtual Nodes
        ↓
Balanced distribution
        ↓
Find affected keys during scaling
```

Everything connects naturally.

---

# Step 1 — The original problem

Imagine you have one cache server.

```
Client
   |
   |
Server
```

Every key goes into this server.

Easy.

Now traffic increases.

One server cannot handle it.

You decide to horizontally scale.

```
          Client
             |
    -----------------
    |       |       |
 Server1 Server2 Server3
```

Now the first question becomes:

**Where should key "user123" be stored?**

If one client stores it in Server1 and another client searches in Server3, the data appears lost.

Every client in the world must independently reach the same server for the same key.

Therefore we need a deterministic function.

```
Input Key
     |
 Hash Function
     |
Server Number
```

This is where traditional hashing starts.

---

# Step 2 — Traditional Hashing

Suppose there are 4 servers.

```
S0
S1
S2
S3
```

Take a key.

```
user123
```

Generate its hash.

```
hash(user123)
= 987654321
```

Now convert this huge number into one of four servers.

```
987654321 % 4

=1
```

Therefore

```
user123 → Server1
```

Every client performs the same calculation.

Everyone reaches Server1.

Perfect.

The formula becomes

```
Server = hash(key) % N
```

where

```
N = number of servers
```

This is extremely common.

---

# Step 3 — Why modulo works

Suppose

```
hash(A)=13

13 % 4 =1
```

```
hash(B)=24

24 % 4 =0
```

```
hash(C)=91

91 %4 =3
```

No matter how large the hash becomes,

Modulo compresses it into

```
0
1
2
3
```

which correspond to server indexes.

Very simple.

Very efficient.

Works perfectly...

...until scaling happens.

---

# Step 4 — The Rehashing Problem

Suppose we initially have

```
4 servers

S0
S1
S2
S3
```

Keys are

```
A → S1

B → S2

C → S0

D → S3
```

Everything works.

Now suppose Server1 crashes.

Now only

```
S0
S2
S3
```

remain.

The total servers become

```
N=3
```

Notice something important.

The hash values of keys never changed.

```
hash(A)
```

is exactly the same.

Only

```
%4
```

became

```
%3
```

Example

```
Old

13 %4=1

Server1
```

After failure

```
13 %3=1
```

Looks okay.

Another key

```
14 %4=2

Server2
```

After removal

```
14 %3=2
```

Still okay.

Another

```
15 %4=3

Server3
```

Now

```
15 %3=0

Server0
```

Changed.

Another

```
16 %4=0

Server0
```

Now

```
16 %3=1

Server2
```

Changed.

Almost every key changes.

Not just the keys from Server1.

This is the biggest problem.

---

# Step 5 — Why this is disastrous

Imagine Redis cache.

Millions of users.

Initially

```
UserA → Server1
```

After removing one server

Client calculates

```
UserA → Server3
```

Server3 doesn't have it.

Cache miss.

Now the application asks Database.

Millions of keys do this together.

```
Cache
↓

Miss

↓

Database

↓

Millions of Requests
```

Database suddenly receives enormous traffic.

This is called

**Cache Miss Storm**

or

**Cache Stampede**

Removing one server can overload the database.

Exactly what scaling was supposed to avoid.

This is why modulo hashing is unsuitable for distributed systems.

---

# Step 6 — What should the ideal algorithm do?

We ask ourselves:

Suppose one server dies.

Should every key move?

No.

Only the keys inside that server should move.

Nothing else.

Similarly,

When adding one server,

Should every key move?

Again no.

Only a few keys should shift.

This is the design goal.

Consistent Hashing achieves exactly this.

---

# Step 7 — Main idea of Consistent Hashing

Instead of assigning keys to

```
Server Number
```

we assign both

```
Servers
```

and

```
Keys
```

onto the same circle.

That circle is called

**Hash Ring**

This is the biggest conceptual shift.

Old approach

```
Key

↓

Modulo

↓

Server Number
```

New approach

```
Key

↓

Position on Ring

↓

Nearest Server
```

Completely different algorithm.

---

# Step 8 — Why a ring?

Hash values already range from

```
0

to

2^160−1
```

(using SHA-1).

Normally this looks like

```
0-------------------------2^160-1
```

Notice

```
0
```

and

```
2^160−1
```

are actually adjacent if we wrap around.

So we join both ends.

```
0
↑
|
|
|
↓

2^160−1
```

forming

```
Circle
```

This circle is called the Hash Ring.

The ring itself has no magical property—it simply avoids edge cases at the beginning and end of the hash space because after the maximum value you naturally wrap back to 0.

---

# Step 9 — Put servers on the ring

Hash each server name.

```
hash(Server0)

↓

Position
```

Do this for every server.

```
Server0

↓

Position 10
```

```
Server1

↓

Position 90
```

```
Server2

↓

Position 220
```

```
Server3

↓

Position 310
```

Now servers occupy positions on the circle.

Notice:

There is **no modulo** anymore.

We are using the hash value directly as the server's position.

---

# Step 10 — Put keys on the ring

Now hash every key.

```
KeyA

↓

Position 75
```

```
KeyB

↓

Position 180
```

Keys and servers now coexist on the same ring.

---

# Step 11 — Server Lookup

Question:

Where should KeyA go?

Rule:

Move clockwise until the first server appears.

Example:

```
KeyA
↓

75

↓

Clockwise

↓

Server1
```

Therefore

```
KeyA → Server1
```

Every client follows the same rule.

No central lookup table is required.

This is deterministic.

---

# Step 12 — Adding a server

Suppose new Server4 appears.

Hash it.

It lands between

```
Key0

and

Server0
```

Now what changes?

Only keys between

```
Previous Server

↓

New Server
```

move.

Everything else still reaches exactly the same server.

This is the entire magic.

Instead of

```
90%

keys moving
```

only

```
small fraction
```

move.

---

# Step 13 — Why only those keys move

Each server owns the keys in the interval from its predecessor (moving clockwise) up to itself.

When a new server is inserted, it "steals" only part of its successor's interval.

No other intervals change.

That's why only nearby keys are redistributed.

---

# Step 14 — Removing a server

Suppose

```
Server1
```

dies.

Only its owned interval loses its owner.

Those keys are simply assigned to the next server clockwise.

Every other server's interval remains unchanged.

Again,

very little movement.

---

# Step 15 — Does this completely solve everything?

No.

It solves

**minimal key movement**

but introduces another issue.

---

# Step 16 — Unequal Partitions

Suppose server positions happen to be

```
S0

-------------------------

S1

--

S2

-----------------------------

S3
```

Notice

S2 owns a huge interval.

S1 owns a tiny interval.

Since each interval receives keys uniformly, S2 will likely store far more data than S1.

So load becomes uneven.

The algorithm depends on random server positions, which may accidentally create large and small gaps.

---

# Step 17 — Non-uniform Key Distribution

This is a consequence of unequal partitions.

Suppose

```
S2 owns

50%

of ring
```

Then approximately

```
50%

of keys
```

also belong to S2.

```
S0

10%

Keys
```

```
S1

15%

Keys
```

```
S2

50%

Keys
```

```
S3

25%

Keys
```

One server becomes overloaded while others are underutilized.

This creates hotspots.

---

# Step 18 — Solution: Virtual Nodes

Instead of placing one point for each server, place many points.

Instead of

```
S0
```

create

```
S0_1

S0_2

S0_3

...

S0_100
```

All of these represent the same physical server.

Do this for every server.

Now each physical server owns many small partitions scattered across the ring instead of one large partition.

---

# Step 19 — Why Virtual Nodes work

Suppose

```
100 virtual nodes

for every server
```

Now one server no longer depends on one lucky position.

It owns

```
100
```

small regions.

Some are large.

Some are small.

Average ownership becomes almost equal.

This is similar to averaging many random samples—the extremes cancel out.

As the number of virtual nodes increases, the distribution becomes smoother and closer to uniform.

---

# Step 20 — Tradeoff

Virtual nodes improve balance, but they are not free.

Advantages:

* Better load balancing.
* Reduced hotspots.
* More even storage and request distribution.

Costs:

* More metadata to maintain (mapping virtual nodes to real servers).
* Larger routing tables.
* Slightly more memory usage.

The number of virtual nodes is therefore a tuning parameter. Systems often use tens to hundreds of virtual nodes per physical server, depending on scale.

---

# Step 21 — Finding affected keys

When topology changes, you don't want to scan every key.

You only need the interval whose ownership changed.

### Adding a server

If server `S4` is inserted:

* Start at `S4`.
* Move **anticlockwise** until the previous server.
* All keys in that interval now belong to `S4`.

Everything else stays where it is.

### Removing a server

If `S1` is removed:

* Take the interval that `S1` owned (from its predecessor to `S1`).
* Those keys are reassigned to the next server clockwise.

Again, no other keys move.

This is why consistent hashing enables efficient scaling.

---

# Complete Connection of the Chapter

Every topic in the chapter is solving the limitation introduced by the previous topic:

```
Need horizontal scaling
        ↓
Need to distribute data
        ↓
Use hash(key) % N
        ↓
Works only when N never changes
        ↓
Server added/removed
        ↓
Almost every key gets remapped
        ↓
Cache miss storm
        ↓
Need an algorithm with minimal remapping
        ↓
Consistent Hashing
        ↓
Represent hash space as a ring
        ↓
Place servers on the ring
        ↓
Place keys on the ring
        ↓
Assign each key to the first server clockwise
        ↓
Adding/removing a server changes only one interval
        ↓
Minimal key movement achieved
        ↓
But server positions create unequal partitions
        ↓
Uneven load and hotspots
        ↓
Introduce virtual nodes
        ↓
Each physical server owns many small partitions
        ↓
Balanced distribution with minimal remapping
        ↓
During scaling, redistribute only the affected interval
```

That progression is the core design story of consistent hashing: **start with a simple distribution method, identify why it fails under dynamic scaling, redesign the mapping using a hash ring to minimize data movement, then refine it with virtual nodes to achieve balanced load.**

