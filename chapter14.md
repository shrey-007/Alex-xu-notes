# These are the notes for Chapter 14 - Designing YouTube

This chapter is **not about storing videos**.

It is about building a **global distributed video platform** where millions of users upload videos while billions watch them smoothly.

The biggest challenge isn't uploading a file.

The biggest challenge is:

> **How do you upload huge videos, process them into multiple formats, deliver them globally with low latency, and do all this at a reasonable cost?**

Like the Key-Value Store chapter, every concept appears because the previous solution breaks.

Think of it as one continuous story.

---

# Stage 1 - The Simplest Video Platform

Suppose a creator uploads a video.

Viewer clicks play.

Simplest architecture

```text
Creator

↓

Server

↓

Disk

↓

Viewer
```

Server stores file.

Viewer downloads file.

Life is beautiful.

---

## Problem #1

Videos are huge.

Example

```
300 MB
```

Viewer must wait until

entire file downloads.

Terrible user experience.

---

## Solution

Streaming.

Instead of downloading

```
300 MB
```

send small chunks continuously.

```text
Viewer

↓

Chunk 1

↓

Chunk 2

↓

Chunk 3
```

Video starts immediately.

---

# Stage 2 - One Server Cannot Handle Everything

Suppose

```
5 Million DAU

25 Million video views/day
```

Every viewer requests videos.

Single server becomes bottleneck.

---

## Solution

Separate responsibilities.

```text
Clients

↓

Load Balancer

↓

API Servers

↓

Storage/CDN
```

API Servers

* Login
* Upload
* Metadata
* Feed

Video data

↓

Storage/CDN

---

# Stage 3 - Where should videos be stored?

Can we store videos

inside database?

No.

Videos are huge binary objects.

Databases become expensive.

---

## Solution

Blob Storage.

```text
Upload

↓

Blob Storage
```

Stores original videos efficiently.

Metadata remains in database.

---

# Stage 4 - Need Metadata

Video is more than file.

Need

```
Title

Description

Owner

Resolution

Format

Duration

Thumbnail

Storage URL
```

Store this separately.

```text
Metadata DB

+

Metadata Cache
```

Why cache?

Metadata is requested

far more often than updated.

---

# Stage 5 - Users Worldwide

Suppose

Viewer in India

Video stored in US.

Every request crosses ocean.

Huge latency.

---

## Solution

CDN.

```text
Origin Storage

↓

CDN

↓

Nearest Edge Server

↓

Viewer
```

Popular videos stay close

to users.

Streaming becomes fast.

---

# Stage 6 - Uploading Isn't Finished Yet

Suppose creator uploads

```
movie.mp4
```

Can every device play it?

No.

Different devices support

different

* codecs
* resolutions
* bitrates

Need conversion.

---

# Stage 7 - Video Transcoding

Convert one video

into many versions.

Example

```text
Original

↓

1080p

720p

480p

360p
```

Also

```
Different codecs

Different formats
```

Now

TV

Phone

Browser

all work.

---

## Why multiple versions?

Network speed changes.

Fast internet

↓

1080p

Slow internet

↓

360p

Adaptive streaming becomes possible.

---

# Stage 8 - Upload Flow

Creator uploads.

```text
Creator

↓

API Server

↓

Original Blob Storage

↓

Transcoding Servers

↓

Encoded Storage

↓

CDN
```

In parallel

Metadata goes

```text
API

↓

Metadata Cache

↓

Metadata DB
```

When transcoding finishes

Completion Queue

↓

Completion Workers

↓

Update Metadata

↓

Video becomes available.

---

# Stage 9 - Why Completion Queue?

Suppose transcoding

takes

```
10 minutes.
```

Should client

wait?

No.

Upload succeeds immediately.

Background workers finish processing.

Completion queue

notifies system

when ready.

Asynchronous processing.

---

# Stage 10 - Video Streaming

Viewer presses Play.

Flow

```text
Viewer

↓

CDN

↓

Video Chunks

↓

Player
```

API server is

not involved

in video transfer.

Only metadata.

---

# Stage 11 - Streaming Protocols

Need standard way

to stream chunks.

Protocols

```
HLS

MPEG-DASH

Smooth Streaming
```

Purpose

Not to store videos.

Purpose is

how player

downloads chunks

and changes quality.

---

# Stage 12 - One Transcoding Server Becomes Bottleneck

Suppose

every uploaded video

passes through

one pipeline.

Slow.

Need parallelism.

---

# Stage 13 - DAG Pipeline

Instead of

one giant job,

split work.

```text
Original Video

↓

Split

↓

Video

Audio

Metadata

↓

Encode

Thumbnail

Watermark

↓

Merge
```

Independent tasks

run simultaneously.

Much faster.

---

# Stage 14 - Why DAG?

Different creators

need different processing.

Some require

```
Thumbnail

Watermark

HDR

Captions
```

Others don't.

Need configurable pipeline.

DAG lets us

build custom workflows

without changing code.

---

# Stage 15 - Preprocessor

First component

before encoding.

Responsibilities

* Split video into GOP chunks
* Generate DAG
* Store temporary copies
* Retry support

Why split?

Smaller chunks

enable parallel processing

and resumable uploads.

---

# Stage 16 - GOP

Video isn't random bytes.

It consists of

```
Frames
```

Frames grouped into

```
GOP
```

Each GOP

can be processed independently.

Perfect for

parallel encoding.

---

# Stage 17 - DAG Scheduler

Now DAG exists.

Need execution order.

Scheduler converts

graph

↓

Stages

↓

Tasks

↓

Queue

Dependencies respected.

Independent tasks

run together.

---

# Stage 18 - Resource Manager

Many workers exist.

Question

Who executes

which task?

Need scheduler.

Components

```
Task Queue

Worker Queue

Running Queue
```

Scheduler

assigns

best worker

to highest priority task.

Maximizes utilization.

---

# Stage 19 - Workers

Workers perform

actual jobs.

Example

```
Video Encoding

Audio Encoding

Thumbnail

Watermark
```

Each worker specializes

in one task.

Easy horizontal scaling.

---

# Stage 20 - Temporary Storage

Intermediate files

cannot stay

only in memory.

Need storage.

Stores

```
Split Videos

Audio

Metadata

Intermediate Outputs
```

If worker crashes,

retry starts

from saved stage

instead of

entire upload.

---

# Stage 21 - Need Faster Uploads

Uploading

1GB file

at once

is risky.

Connection drops.

Restart.

Terrible.

---

## Solution

Chunk Upload.

Split

by GOP.

```text
Chunk1

Chunk2

Chunk3
```

Failed chunk

retries alone.

Resume upload.

Much faster.

---

# Stage 22 - Global Upload Centers

Indian user

uploading

to US

is slow.

Need

regional upload centers.

```text
India

↓

Asia Upload Center

↓

Origin Storage
```

Usually implemented

using CDN edge locations.

---

# Stage 23 - Need More Parallelism

Suppose

Encoding waits

for Download.

Watermark waits

for Encoding.

Everything blocks.

---

## Solution

Message Queues.

```text
Download

↓

Queue

↓

Encoding

↓

Queue

↓

Thumbnail
```

Each stage

works independently.

Loose coupling.

High throughput.

---

# Stage 24 - Security

Anyone should not

upload files

into storage.

Need authorization.

---

## Solution

Pre-signed URL.

Flow

```text
Client

↓

API

↓

Signed Upload URL

↓

Upload Directly

↓

Blob Storage
```

API never handles

video bytes.

Much more scalable.

---

# Stage 25 - Protect Videos

Creators fear piracy.

Need protection.

Methods

```
DRM

AES Encryption

Watermark
```

DRM

controls playback.

AES

encrypts content.

Watermark

identifies ownership.

---

# Stage 26 - CDN is Expensive

Example

```
150 TB/day
```

Streaming costs

huge money.

Need optimization.

---

## Solution

Long Tail Distribution.

Reality

Few videos

get

millions of views.

Most videos

barely watched.

So

Only popular videos

stay in CDN.

Others

served

from origin.

---

Additional savings

* Encode unpopular videos on-demand
* Store only required resolutions
* Cache only in regions where popular
* Build own CDN at very large scale

---

# Stage 27 - Error Handling

Large systems fail.

Need retries.

Recoverable

```
Encoding Failure

↓

Retry
```

Non-recoverable

```
Corrupted Video

↓

Reject Upload
```

Component failures

```
Upload

↓

Retry

Worker Down

↓

New Worker

Queue Down

↓

Replica

API Down

↓

Load Balancer

DB Master Down

↓

Promote Replica

Cache Down

↓

Read Replica
```

Every component

has recovery strategy.

---

# Stage 28 - Final Architecture

```text
                 Client
                    │
         ┌──────────┴──────────┐
         │                     │
      Upload                Watch Video
         │                     │
         ▼                     ▼
     API Servers             CDN
         │                     │
         ▼                     │
 Metadata DB + Cache           │
         │                     │
         ▼                     ▼
 Original Blob Storage ──► Video Chunks
         │
         ▼
   Preprocessor
         │
         ▼
       DAG
         │
         ▼
   Resource Manager
         │
         ▼
    Task Workers
         │
         ▼
 Encoded Blob Storage
         │
         ▼
        CDN
```

---

# Stage 29 - Complete Upload Flow

Creator uploads

↓

Pre-signed URL

↓

Original Blob Storage

↓

Preprocessor

↓

Split into GOP

↓

Generate DAG

↓

Scheduler

↓

Workers

↓

Encode

↓

Temporary Storage

↓

Encoded Storage

↓

CDN

↓

Completion Queue

↓

Completion Handler

↓

Metadata Updated

↓

Video Ready.

---

# Stage 30 - Complete Watch Flow

Viewer presses Play.

↓

API fetches metadata

↓

Returns CDN URL

↓

Player contacts nearest CDN

↓

Streaming protocol

↓

Adaptive bitrate

↓

Video chunks arrive continuously

↓

Viewer watches immediately.

---

# The entire chapter in one picture

```text
Need Video Platform
        │
        ▼
Store Video
        │
        ▼
Downloading is slow
        │
        ▼
Streaming
        │
        ▼
One server overloaded
        │
        ▼
API + Storage Separation
        │
        ▼
Need scalable storage
        │
        ▼
Blob Storage
        │
        ▼
Need metadata
        │
        ▼
Metadata DB + Cache
        │
        ▼
Global users
        │
        ▼
CDN
        │
        ▼
Different devices
        │
        ▼
Video Transcoding
        │
        ▼
Need multiple resolutions
        │
        ▼
Adaptive Streaming
        │
        ▼
Encoding becomes slow
        │
        ▼
DAG Pipeline
        │
        ▼
Need execution order
        │
        ▼
DAG Scheduler
        │
        ▼
Need worker allocation
        │
        ▼
Resource Manager
        │
        ▼
Task Workers
        │
        ▼
Need resumable processing
        │
        ▼
Temporary Storage
        │
        ▼
Need faster uploads
        │
        ▼
GOP Chunk Upload
        │
        ▼
Need global uploads
        │
        ▼
Regional Upload Centers
        │
        ▼
Need loose coupling
        │
        ▼
Message Queues
        │
        ▼
Need security
        │
        ▼
Pre-Signed URLs
        │
        ▼
Need copyright protection
        │
        ▼
DRM / AES / Watermark
        │
        ▼
CDN becomes expensive
        │
        ▼
Long-tail Optimization
        │
        ▼
Need fault tolerance
        │
        ▼
Retries + Replication + Recovery
```

## The connecting thread

The chapter starts with the simplest idea: **store a video and let users download it**. That quickly fails because videos are large, so **streaming** is introduced. As users grow globally, a single server cannot handle requests, leading to the separation of **API servers, blob storage, metadata storage, and CDN**. Different devices and network speeds require **video transcoding** into multiple formats and resolutions. Since transcoding is computationally expensive, the work is broken into a **DAG of independent tasks**, scheduled and executed by workers through a **resource manager**. To improve throughput, videos are split into **GOP chunks**, uploads become resumable, and **message queues** decouple each processing stage. Security is added through **pre-signed upload URLs**, **DRM**, **AES encryption**, and **watermarking**. Finally, because CDN bandwidth dominates operating cost, the system exploits the **long-tail access pattern** by caching only popular videos and optimizing where and when different versions are stored. Every concept in the chapter is introduced because the previous design reaches a scalability, compatibility, performance, cost, or reliability limit.

