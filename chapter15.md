# Design Google Drive — The Entire Chapter as One Story

This chapter is not about "cloud storage."

It is about solving one problem after another until you build something that behaves like Google Drive.

The entire flow is:

```text
Need cloud storage
        │
        ▼
Single server
        │
        ▼
Storage full
        │
        ▼
Distributed storage
        │
        ▼
Prevent data loss
        │
        ▼
Cloud object storage (S3)
        │
        ▼
Multiple servers
        │
        ▼
Metadata separation
        │
        ▼
Large file uploads
        │
        ▼
Block storage
        │
        ▼
Bandwidth problems
        │
        ▼
Delta Sync + Compression
        │
        ▼
Multiple devices
        │
        ▼
Notifications
        │
        ▼
File synchronization
        │
        ▼
Version history
        │
        ▼
Conflict resolution
        │
        ▼
Storage optimization
        │
        ▼
Failure handling
```

---

# Stage 1 — Start with a simple file server

Suppose users want to

```
Upload file
Download file
```

Simplest architecture:

```
Client
   │
Web Server
   │
MySQL
   │
Disk (/drive folder)
```

Store files like

```
/drive
   ├── UserA
   │      report.pdf
   ├── UserB
          image.jpg
```

Database stores

```
User
Filename
Path
Size
Owner
```

Life is simple.

---

## Problem #1

Disk becomes full.

Example

```
Disk = 1 TB

Users = Millions
```

Impossible.

Need distributed storage.

---

# Stage 2 — Scale storage

Instead of one disk

```
Storage1

Storage2

Storage3
```

Shard files.

Example

```
hash(userId)

↓

Storage Server
```

Now capacity grows horizontally.

Problem solved.

---

## New Problem

Suppose

```
Storage2 crashes.
```

Entire user data is gone.

Very bad.

Need durable storage.

---

# Stage 3 — Store files in Object Storage (Amazon S3)

Instead of maintaining storage ourselves

```
Client

↓

Block Server

↓

S3
```

Benefits

* automatic replication
* high durability
* virtually unlimited storage
* cross-region backup

Now file loss becomes extremely unlikely.

---

## New Problem

Everything still goes through one web server.

That server dies.

Entire system dies.

---

# Stage 4 — Separate components

Instead of

```
One giant server
```

Split system

```
           Load Balancer
                 │
      ---------------------
      │         │         │
   API1      API2      API3

Metadata DB

Metadata Cache

Block Servers

S3
```

Now

* API scales independently
* Storage scales independently
* Database scales independently

System becomes horizontally scalable.

---

# Stage 5 — Uploading large files

Uploading

```
5 GB
```

as one request is terrible.

If network breaks at

```
99%
```

Restart entire upload.

Very expensive.

---

## Solution

Resumable Upload.

Flow

```
Request Upload URL

↓

Upload chunks

↓

Interrupted?

↓

Resume
```

Only remaining data uploads.

---

# Stage 6 — Uploading entire files wastes bandwidth

Suppose file

```
100 MB
```

User edits

```
one sentence
```

Naive approach

```
Upload entire 100 MB.
```

Wasteful.

---

## Solution — Block Storage

Split file into blocks.

```
File

↓

Block1

Block2

Block3

Block4
```

Store blocks independently.

Metadata remembers order.

```
File

↓

Block IDs

↓

Reconstruct later
```

Dropbox uses 4 MB blocks.

---

# Stage 7 — Even blocks are large

Suppose only

```
Block2
```

changed.

Why upload all blocks?

---

## Solution — Delta Sync

Only changed blocks move.

```
Before

B1
B2
B3
B4

After

B2 changed
```

Upload

```
Only B2
```

Huge bandwidth savings.

---

# Stage 8 — Reduce bandwidth further

Before upload

Every block is

```
Compressed

↓

Encrypted

↓

Uploaded
```

Compression

↓

Less bandwidth

Encryption

↓

Security

---

# Stage 9 — Metadata

Never store files inside SQL.

Database stores only

```
File Name

Owner

Version

Block IDs

Hash

Status
```

Actual bytes stay inside

```
S3
```

This keeps database small.

---

# Stage 10 — Upload Flow

Two things happen simultaneously.

### Flow A

Actual file

```
Client

↓

Block Server

↓

Split

↓

Compress

↓

Encrypt

↓

S3
```

---

### Flow B

Metadata

```
Client

↓

API Server

↓

Metadata DB

↓

Status = Pending
```

After upload completes

```
Callback

↓

Status = Uploaded

↓

Notify users
```

Running both flows in parallel makes uploads faster.

---

# Stage 11 — Multiple Devices

Suppose

Laptop uploads

```
Report.pdf
```

Phone must immediately know.

How?

Need synchronization.

---

# Stage 12 — Notification Service

Whenever metadata changes

```
Upload

Delete

Rename

Share
```

Notification service informs devices.

```
Client1

↓

Notification Service

↓

Client2
```

Now every device knows

```
Something changed.
```

---

# Stage 13 — Download Flow

Notification only says

```
New version exists.
```

It doesn't send the file.

Flow

```
Notification

↓

Client requests metadata

↓

Metadata DB

↓

Block IDs

↓

Download blocks

↓

Reconstruct file
```

Only required blocks download.

---

# Stage 14 — Notification mechanism

Question

```
WebSocket?

or

Long Polling?
```

Google Drive mostly needs

```
Server

↓

Client
```

communication.

Client rarely sends anything.

So

```
Long Polling
```

is sufficient.

Flow

```
Client opens request

↓

Wait

↓

Server responds when file changes

↓

Client reconnects
```

Cheaper than permanent WebSocket.

---

# Stage 15 — Version History

Every edit should not overwrite previous file.

Instead

```
File v1

↓

File v2

↓

File v3
```

Database contains

```
File

FileVersion

Blocks
```

Old versions remain read-only.

Users can restore previous versions.

---

# Stage 16 — Sync Conflicts

Suppose

Laptop

```
Edit Report
```

Phone

```
Edit Report
```

at exactly the same time.

Who wins?

Strategy

```
First processed

↓

Accepted
```

Second user receives

```
Conflict
```

System shows

```
Local Copy

Server Copy
```

User chooses

* Merge
* Replace
* Keep both

---

# Stage 17 — Save Storage

Version history consumes huge storage.

Need optimization.

### 1. Deduplication

If two blocks have same hash

```
Store once

Reference many times
```

---

### 2. Version Limits

Instead of

```
10000 versions
```

Keep

```
Latest N
```

or

important versions only.

---

### 3. Cold Storage

Old inactive files

```
↓

Cheap storage
```

like

```
Amazon Glacier
```

Hot files stay in fast storage.

Huge cost reduction.

---

# Stage 18 — Reliability

Failures happen everywhere.

Solutions

| Failure                 | Solution                    |
| ----------------------- | --------------------------- |
| Load Balancer           | Backup LB + Heartbeat       |
| API Server              | Stateless, redirect traffic |
| Block Server            | Retry on another worker     |
| S3 Region Down          | Cross-region replication    |
| Metadata Cache          | Replicas                    |
| Metadata DB Master Down | Promote replica             |
| Metadata DB Slave Down  | Replace slave               |
| Notification Server     | Clients reconnect           |
| Offline Queue           | Replicated queues           |

Nothing depends on one machine.

---

# Final Architecture

```
                Client
                   │
          Load Balancer
                   │
         -------------------
         │                 │
    API Servers      Notification
         │                 │
         │
   Metadata Cache
         │
   Metadata Database
         │
     Block Servers
         │
 Compress
 Encrypt
 Delta Sync
 Chunking
         │
         ▼
     Amazon S3
         │
 Cross Region Replication
```

---

# Complete Chapter in One Picture

```text
Need cloud storage
        │
        ▼
Single server
        │
        ▼
Disk full
        │
        ▼
Sharding
        │
        ▼
Need durability
        │
        ▼
S3 Object Storage
        │
        ▼
Need scalability
        │
        ▼
Load Balancer + API Servers
        │
        ▼
Large uploads
        │
        ▼
Resumable Upload
        │
        ▼
Bandwidth waste
        │
        ▼
Block Storage
        │
        ▼
Only small changes
        │
        ▼
Delta Sync
        │
        ▼
Reduce bandwidth
        │
        ▼
Compression + Encryption
        │
        ▼
Metadata Management
        │
        ▼
Parallel Upload + Metadata Update
        │
        ▼
Notifications
        │
        ▼
File Synchronization
        │
        ▼
Long Polling
        │
        ▼
Version History
        │
        ▼
Sync Conflicts
        │
        ▼
Deduplication
        │
        ▼
Cold Storage
        │
        ▼
Failure Recovery
```

The entire chapter is just one chain of **Problem → Solution → New Problem**. Every concept exists because the previous solution exposed a new limitation. Once you follow that chain, the whole Google Drive design becomes easy to remember.

