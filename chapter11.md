# These are the notes for Chapter 11 - Designing a News Feed System

This chapter is one of the most misunderstood chapters in *System Design Interview*.

Most people think they are building a **Facebook homepage**.

They are not.

They are building a **distributed data distribution system**.

The actual problem is

> **One user creates data. Thousands or millions of other users need to see that data quickly.**

Everything else—caches, fanout, message queues, graph database—exists only because of this one problem.

Like the Key-Value Store chapter, this chapter is one continuous story.

Every new concept appears because the previous solution stops working.

---

# Stage 1 - The simplest News Feed

Suppose Facebook has only **2 users**.

```text
Alice

Bob
```

Alice makes a post.

```text
"Hello World"
```

Bob opens Facebook.

Question:

How should Bob see Alice's post?

Simplest solution

```text
Posts Table

---------------------

Alice -> Hello World
```

Whenever Bob opens Facebook

```text
SELECT *

FROM Posts

ORDER BY time DESC
```

Return everything.

Done.

Very simple.

---

## Problem #1

Now imagine Facebook.

Millions of users.

```text
Alice

Bob

John

Mike

Sarah

...
```

Bob should **not** see everyone's posts.

He should only see

```text
Posts

↓

Friends' Posts
```

Now we need friendships.

---

## Solution

Store friendships.

```text
Friend Table

Alice → Bob

Alice → Mike

Bob → Sarah

Bob → John
```

Now when Bob opens Facebook

Step 1

```text
Find Bob's friends
```

Step 2

```text
Find their posts
```

---

# Stage 2 - Feed Generation

Suppose Bob has

```text
5000 friends.
```

To generate Bob's feed

Server does

```text
Find Friends

↓

Find Posts of every Friend

↓

Merge

↓

Sort by Time

↓

Return Feed
```

This works.

Life is beautiful.

---

## Problem #2

Suppose

```text
10 Million DAU
```

Every time someone refreshes

Server must

```text
Friend Lookup

↓

Thousands of Post Lookups

↓

Merge

↓

Sort
```

Again.

And again.

And again.

Same work repeated.

Very expensive.

---

## Solution

Store the generated feed.

Instead of computing

every request,

precompute it.

```text
Bob Feed

↓

Post 101

Post 97

Post 95

Post 92
```

Now

Bob opens Facebook

↓

Return Feed.

Very fast.

This idea is called

```text
Feed Cache
```

This is the first major idea of the chapter.

---

# Stage 3 - New post arrives

Suppose Alice posts

```text
"I got a new job."
```

Question

How does Bob's cached feed change?

Current cache

```text
Bob Feed

↓

100

98

95
```

Need

```text
Alice's New Post

↓

101

100

98

95
```

Somebody must insert

Alice's post

into

Bob's feed.

---

## Solution

Whenever someone creates a post,

immediately push it

into every friend's feed.

Example

```text
Alice

↓

Post Created

↓

Bob Feed

Mike Feed

John Feed

Sarah Feed
```

This is called

```text
Fanout on Write

(Push Model)
```

The feed is built

during writing.

---

# Stage 4 - Fanout on Write becomes expensive

Suppose Alice has

```text
5000 friends.
```

One post requires

```text
5000 feed updates.
```

Still manageable.

Now imagine

Taylor Swift.

Followers

```text
100 Million
```

One post requires

```text
100 Million feed updates.
```

Impossible.

One write becomes

100 million writes.

This is called the

```text
Hotkey Problem
```

One extremely popular user creates enormous load.

---

## Solution

Don't push to celebrities.

Instead

generate feed

when followers open app.

Follower opens app

↓

Read celebrity posts

↓

Show feed.

This is called

```text
Fanout on Read

(Pull Model)
```

Instead of pushing,

followers pull.

---

# Stage 5 - Fanout on Read creates another problem

Now Bob opens Facebook.

Server must

```text
Find Friends

↓

Find Posts

↓

Merge

↓

Sort
```

Every refresh.

Feed generation becomes slow again.

Exactly the problem

we solved earlier.

---

## Solution

Hybrid Model.

Normal users

↓

Fanout on Write

Celebrities

↓

Fanout on Read

This is exactly what Facebook,

Twitter,

Instagram,

LinkedIn,

etc.

actually do.

Why?

Most users have

```text
100-500 friends.
```

Very cheap to push.

Very few users have

millions of followers.

Those users use Pull.

Best of both worlds.

---

# Stage 6 - Who are my friends?

Suppose Alice creates a post.

Need to fanout.

Question

Who receives it?

Need friend list.

---

## Solution

Graph Database.

Unlike SQL,

friendships naturally form

a graph.

```text
Alice

 /   \

Bob  Mike

 |

Sarah

 |

John
```

Finding friends,

mutual friends,

friend recommendations

are extremely efficient

inside Graph Databases.

So

Fanout Service first asks

```text
Graph Database

↓

Friend IDs
```

---

# Stage 7 - Not every friend should receive the post

Suppose

Bob muted Alice.

Or

Alice shared post

only with

Close Friends.

Graph Database still returns Bob.

Should Bob receive post?

No.

Need filtering.

---

## Solution

User Cache.

Fetch

```text
Friend IDs

↓

User Settings
```

Check

```text
Muted?

Blocked?

Private Audience?

Custom Audience?
```

Remove users

who shouldn't receive post.

Only valid users continue.

---

# Stage 8 - Updating thousands of feeds synchronously

Suppose after filtering

still have

```text
4000 friends.
```

Notification Service updates

4000 feeds.

API waits.

User keeps staring at loading screen.

Very slow.

---

## Solution

Message Queue.

Instead of updating

feeds immediately

```text
Post Created

↓

Queue

↓

Fanout Workers
```

User gets response immediately.

Workers continue

background processing.

This decouples posting

from feed generation.

---

# Stage 9 - Workers

Workers continuously process queue.

```text
while(true)

↓

Read Job

↓

Update Feed Cache
```

Need more throughput?

Add workers.

```text
Queue

↓

Worker1

Worker2

Worker3

Worker4
```

Easy horizontal scaling.

---

# Stage 10 - What exactly is Feed Cache?

Many beginners imagine

Feed Cache stores

entire posts.

Wrong.

Imagine

Bob's feed

contains

```text
Profile Picture

Username

Images

Videos

Comments

Likes

Replies
```

Suppose

100 million users.

Memory explodes.

---

## Solution

Store only IDs.

Feed Cache contains

```text
Bob

↓

101

98

95

91
```

Just

```text
<PostID>
```

or

```text
<UserID, PostID>
```

mapping.

Very small.

Very cheap.

---

# Stage 11 - Why only IDs?

Question

Where is

actual post?

Stored elsewhere.

Later,

during retrieval,

system fetches

complete post.

Benefits

```text
Small Cache

↓

Less Memory

↓

More Users

↓

Lower Cost
```

This is an important optimization.

---

# Stage 12 - Feed Cache keeps growing forever

Suppose

Bob has used Facebook

for

```text
12 years.
```

Feed contains

millions of posts.

Should cache store all?

No.

Nobody scrolls

500,000 posts.

Huge waste of RAM.

---

## Solution

Limit Feed Cache.

Store only latest

```text
500

or

1000
```

posts.

Older posts

can always come

from database

if needed.

Cache remains small.

Cache hit rate remains high.

---

# Stage 13 - User opens Facebook

Now

Bob opens app.

Question

How is feed returned?

Current Feed Cache

```text
101

98

95
```

These are only IDs.

Cannot render UI.

Need complete objects.

---

## Solution

Hydration.

Step 1

Read IDs

↓

Feed Cache

Step 2

Fetch Posts

↓

Post Cache

Step 3

Fetch Users

↓

User Cache

Combine everything

↓

Return JSON.

This process is called

```text
Hydrating the Feed.
```

---

# Stage 14 - Media files are huge

Suppose Post contains

```text
4K Video

100 MB
```

Should database serve

millions of videos?

No.

Database becomes bottleneck.

---

## Solution

CDN.

Store media

inside

```text
Content Delivery Network
```

Flow

```text
Post Metadata

↓

Database

↓

Video URL

↓

CDN

↓

Client downloads directly
```

Application Server

never serves videos.

Only URLs.

This dramatically reduces server load.

---

# Stage 15 - Authentication

Anyone should not be able

to create posts.

Need authentication.

```text
POST

/feed
```

requires

```text
Auth Token
```

Only authenticated users

can publish posts.

---

# Stage 16 - Spam Problem

Suppose attacker sends

```text
1000 posts/sec.
```

Every post triggers

fanout.

Millions of feed updates.

System collapses.

---

## Solution

Rate Limiting.

Example

```text
Maximum

30 Posts

per hour.
```

Reject excess requests.

Protects infrastructure.

---

# Stage 17 - Why so many caches?

At first

one cache seems enough.

Actually

different data

has different access patterns.

So chapter separates caches.

---

## Feed Cache

Stores

```text
Feed IDs.
```

---

## Content Cache

Stores

```text
Posts
```

Popular posts

go into

```text
Hot Cache.
```

Frequently viewed

↓

Stay in RAM.

---

## Social Graph Cache

Stores

```text
Friend Relationships.
```

Avoids querying graph database

every request.

---

## Action Cache

Stores

```text
Likes

Replies

Shares

Bookmarks
```

Very frequently updated.

Separate cache

avoids contention.

---

## Counter Cache

Stores

```text
Likes = 105

Comments = 29

Followers = 2500
```

Instead of counting

every request,

return cached value.

Huge performance improvement.

---

# Stage 18 - Final Architecture

Now every concept comes together.

```text
                    User
                     │
                     ▼
               Load Balancer
                     │
                     ▼
                Web Servers
        (Authentication + Rate Limit)
                     │
         ┌───────────┴───────────┐
         │                       │
         ▼                       ▼
    Post Service          News Feed Service
         │                       │
         ▼                       ▼
 Database + Post Cache      Feed Cache
         │                       │
         ▼                       ▼
     Fanout Service       Fetch Feed IDs
         │                       │
         ▼                       ▼
   Graph Database         User Cache
         │                       │
         ▼                       ▼
     User Cache            Post Cache
         │                       │
         ▼                       ▼
      Message Queue        Hydrate Feed
         │                       │
         ▼                       ▼
    Fanout Workers          CDN(Media)
         │                       │
         ▼                       ▼
 Update Feed Cache         Return JSON Feed
```

---

# Stage 19 - How publishing actually happens

Suppose

Alice posts

```text
"Hello Facebook."
```

Flow

```text
Alice

↓

POST /feed

↓

Authentication

↓

Rate Limiting

↓

Post Service

↓

Store Post

↓

Graph DB

↓

Find Friends

↓

User Cache

↓

Filter Muted Users

↓

Message Queue

↓

Fanout Workers

↓

Update Feed Cache

↓

Send Notification
```

Alice immediately gets

```text
200 OK
```

Workers continue

background fanout.

---

# Stage 20 - How feed retrieval actually happens

Bob opens app.

```text
GET /feed
```

↓

Load Balancer

↓

News Feed Service

↓

Feed Cache

↓

Get Post IDs

↓

Post Cache

↓

User Cache

↓

CDN URLs

↓

Hydrate Feed

↓

Return JSON

↓

Client renders feed.

---

# The entire chapter in one picture

```text
Need News Feed
        │
        ▼
Read all posts
        │
        ▼
Need only friends' posts
        │
        ▼
Friend Relationships
        │
        ▼
Generate feed every request
        │
        ▼
Too slow
        │
        ▼
Feed Cache
        │
        ▼
Need to update cache on new post
        │
        ▼
Fanout on Write
        │
        ▼
Celebrities overload system
        │
        ▼
Fanout on Read
        │
        ▼
Need best of both
        │
        ▼
Hybrid Fanout
        │
        ▼
Need friend lookup
        │
        ▼
Graph Database
        │
        ▼
Need privacy filtering
        │
        ▼
User Cache
        │
        ▼
Feed updates are slow
        │
        ▼
Message Queue
        │
        ▼
Need background processing
        │
        ▼
Fanout Workers
        │
        ▼
Feed Cache becomes huge
        │
        ▼
Store only Post IDs
        │
        ▼
Need complete objects
        │
        ▼
Hydration
        │
        ▼
Media is too large
        │
        ▼
CDN
        │
        ▼
Prevent spam
        │
        ▼
Authentication + Rate Limiting
        │
        ▼
Different data has different access patterns
        │
        ▼
Feed Cache
Post Cache
Graph Cache
Action Cache
Counter Cache
```

The connecting thread that the book doesn't explicitly state is this:

The system starts by **computing the feed when the user opens the app**, but that is too expensive. So it **precomputes feeds during writes** (fanout on write). That works until users with millions of followers appear, making writes prohibitively expensive. The solution is a **hybrid fanout strategy**: push for normal users and pull for celebrities. Once fanout is introduced, the bottleneck shifts to updating thousands of feeds synchronously, leading to **message queues and fanout workers**. As the system scales further, memory becomes the bottleneck, so only **post IDs** are stored in the feed cache, with full objects assembled later through **hydration**. Finally, specialized caches (feed, content, graph, action, counter) and CDNs are introduced because different kinds of data have different access patterns and performance requirements. Every concept in the chapter follows directly from the scaling problem created by the previous solution.

