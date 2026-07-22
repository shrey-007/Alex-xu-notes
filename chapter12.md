# These are the notes for Chapter 12 - Designing a Chat System

This chapter is **not about sending messages**.

It is about building a **real-time distributed communication system** where millions of users stay connected simultaneously.

The biggest challenge isn't storing messages.

The biggest challenge is:

> **How do you instantly deliver messages between users who may be online, offline, connected from multiple devices, or chatting in groups?**

Like every previous chapter, think of this as **one continuous story**.

Every new concept appears because the previous solution stops working.

---

# Stage 1 - The Simplest Chat System

Suppose only two users exist.

```
Alice

Bob
```

Alice sends

```
Hello
```

Simplest architecture

```
Alice

↓

Server

↓

Bob
```

The server simply forwards the message.

Life is beautiful.

---

## Problem #1

What if Bob is offline?

```
Alice

↓

Server

↓

Bob ❌
```

Message disappears.

Very bad.

Users expect

```
Offline messages
```

---

## Solution

Store messages before delivery.

```
Alice

↓

Server

↓

Database

↓

Bob
```

If Bob comes online later,

server reads database

↓

delivers message.

Now messages are never lost.

---

# Stage 2 - How should clients communicate?

The sender wants to send messages.

The receiver wants to receive messages.

Sending is easy.

Receiving is difficult.

---

## Sender Side

Client starts communication.

HTTP works.

```
Client

↓

POST /send

↓

Server
```

Simple.

---

## Problem #2

How does server send message back?

HTTP is

```
Client

↓

Server
```

There is no

```
Server

↓

Client
```

connection.

Server cannot push messages.

Need another solution.

---

# Stage 3 - Polling

First idea

Ask repeatedly.

```
Every 2 seconds

↓

Any message?

↓

No

↓

Any message?

↓

No

↓

Any message?

↓

Yes
```

Works.

---

## Problem

Most requests return

```
No Message.
```

Millions of useless requests.

Waste of

* CPU
* Network
* Database

---

# Stage 4 - Long Polling

Instead of asking every few seconds,

client asks once.

Server waits.

```
Client

↓

Any message?

↓

(wait...)

↓

Message arrives

↓

Return
```

Better.

---

## Problem

Still inefficient.

Problems

* Timeout reconnects
* Load balancer may send next request to another server
* Server cannot easily detect disconnect
* Large number of idle connections

Need something permanent.

---

# Stage 5 - WebSocket

Instead of opening

new connection every time,

keep one connection forever.

```
Alice

⇅

Chat Server

⇅

Bob
```

One connection.

Bidirectional.

Server can push.

Client can send.

No repeated HTTP requests.

This is why modern chat apps use

```
WebSocket
```

for messaging.

---

# Stage 6 - Everything doesn't need WebSocket

Many beginners think

entire application uses WebSocket.

Wrong.

Only

```
Real-time messaging
```

needs WebSocket.

Everything else

```
Signup

Login

Profile

Settings

Friends
```

uses normal HTTP.

This naturally divides the architecture.

---

# Stage 7 - Stateless vs Stateful Services

HTTP Services

```
Login

Signup

Profile
```

don't care

which server handles request.

These are

```
Stateless
```

Any server works.

---

WebSocket is different.

```
Client

⇅

Chat Server
```

Persistent connection exists.

If client suddenly switches servers,

connection breaks.

These servers become

```
Stateful.
```

Each client remains attached

to one Chat Server.

---

# Stage 8 - Need Service Discovery

Question

Which Chat Server

should client connect to?

Cannot choose randomly.

One server may already be full.

Need intelligent selection.

---

## Solution

Service Discovery.

```
Login

↓

Authentication

↓

Service Discovery

↓

Best Chat Server

↓

Connect
```

Service Discovery chooses based on

* Geography
* Server Load
* Capacity

Tools

```
ZooKeeper
```

commonly solve this.

---

# Stage 9 - One Chat Server isn't enough

Suppose

```
50 Million DAU
```

Need many chat servers.

```
Chat1

Chat2

Chat3

Chat4
```

Service Discovery distributes users

across servers.

No single point of failure.

Easy horizontal scaling.

---

# Stage 10 - Need Push Notifications

Suppose Bob closes app.

WebSocket disappears.

Alice sends message.

How does Bob know?

Need

```
Push Notification.
```

Flow

```
Alice

↓

Chat Server

↓

Push Notification Server

↓

APNS / FCM

↓

Bob
```

When Bob opens app,

real messages come from chat storage,

not push notification.

Push only wakes user.

---

# Stage 11 - Where should data be stored?

Chat application stores

two kinds of data.

---

## User Data

```
Users

Profiles

Friends

Settings
```

Relational Database.

Because

* Relationships
* Transactions
* Strong consistency

---

## Chat Messages

Very different.

Requirements

```
Billions of messages

Low latency

Horizontal scaling

Recent messages accessed frequently
```

Relational databases struggle.

---

## Solution

Key-Value Store.

Examples

```
HBase

Cassandra
```

Reasons

* Fast
* Horizontal scaling
* Huge write throughput
* Efficient recent reads

---

# Stage 12 - Message Table

Need message storage.

1-to-1

```
MessageID

Sender

Receiver

Content
```

Group Chat

```
(ChannelID, MessageID)

Sender

Content
```

Partition using

```
ChannelID
```

because group queries

always happen

inside one channel.

---

# Stage 13 - Why Message ID?

Can we order messages

using timestamp?

No.

Two messages

may share same timestamp.

Need

```
Message ID
```

Requirements

* Unique
* Time sortable

Solutions

```
Snowflake

OR

Local Sequence Number
```

Local sequence works because

ordering only matters

inside one conversation,

not globally.

---

# Stage 14 - Sending a Message

Alice sends

```
Hello
```

Flow

```
Alice

↓

WebSocket

↓

Chat Server

↓

Generate MessageID

↓

Store KV Database

↓

Find Bob

↓

Bob Online?
```

---

If online

```
↓

Forward Message

↓

Bob
```

If offline

```
↓

Store Message

↓

Push Notification
```

When Bob reconnects,

messages are loaded.

---

# Stage 15 - Multiple Devices

Suppose Bob has

```
Phone

Laptop

Tablet
```

All logged in.

Alice sends message.

Need all devices

to stay synchronized.

---

## Solution

Each device stores

```
cur_max_message_id
```

Example

Phone

```
100
```

Laptop

```
95
```

Server query

```
MessageID > cur_max
```

Phone receives

```
101
```

Laptop receives

```
96

97

98

99

100

101
```

Synchronization becomes trivial.

---

# Stage 16 - Group Chat

Suppose

Group

```
Alice

Bob

John
```

Alice sends

```
Hello
```

Question

How does everyone receive it?

---

## Solution

Copy message

into each member's inbox.

```
Alice

↓

Message

↓

Bob Inbox

↓

John Inbox
```

Every recipient

only checks

its own inbox.

Simple.

---

## Problem

Suppose

Group

```
100,000 users
```

Need

```
100,000 copies.
```

Impossible.

---

## Solution

Small Groups

```
Copy per member.
```

Large Groups

Use another design

(shared message log / pull model).

Book assumes

maximum

```
100 users.
```

Therefore

copy-per-user

is acceptable.

---

# Stage 17 - Online Presence

Need green dot.

Question

How know user is online?

---

## Login

After WebSocket connects

```
Status

↓

Online
```

stored in KV Store.

---

## Logout

```
Status

↓

Offline
```

Very simple.

---

# Stage 18 - Internet Disconnect

Suppose

user enters tunnel.

Internet disappears

for

```
5 seconds.
```

Should status become

```
Offline
```

immediately?

No.

Indicator keeps blinking.

Terrible UX.

---

## Solution

Heartbeat.

Every

```
5 seconds
```

client sends

```
I'm Alive
```

If server doesn't receive heartbeat

within

```
30 seconds
```

user becomes

```
Offline.
```

Temporary disconnects

no longer matter.

---

# Stage 19 - Friends need Presence Updates

Suppose Alice comes online.

Bob's app

must immediately show

green dot.

Need notification.

---

## Solution

Publish-Subscribe.

```
Alice Online

↓

Presence Server

↓

Publish Event

↓

Bob

↓

John

↓

Mike
```

Subscribers instantly update UI.

---

## Problem

Suppose group has

```
100,000 members.
```

Alice changes status.

Need

```
100,000 events.
```

Expensive.

---

## Solution

Large groups

don't receive

real-time presence.

Instead

fetch status

when opening group

or refreshing list.

---

# Stage 20 - Final Architecture

```
                     Client
                        │
         ┌──────────────┴──────────────┐
         │                             │
     HTTP APIs                  WebSocket
(Login/Profile/etc.)          Real-time Chat
         │                             │
         ▼                             ▼
     API Servers                 Chat Servers
         │                             │
         ▼                             ▼
 Service Discovery             Presence Servers
         │                             │
         ▼                             ▼
 Relational Database          Message Queue
(User/Profile/Friends)              │
                                    ▼
                             Key-Value Store
                              (Chat History)
                                    │
                                    ▼
                        Push Notification Server
                                    │
                               APNS / FCM
```

---

# Stage 21 - Complete Message Flow

Alice sends

```
Hello
```

↓

WebSocket

↓

Chat Server

↓

Generate Message ID

↓

Store in KV Store

↓

Bob Online?

↓

Yes

↓

Forward through Bob's WebSocket

↓

Bob receives instantly.

OR

↓

Bob Offline

↓

Store message

↓

Send Push Notification

↓

Bob opens app

↓

Fetch

```
MessageID > cur_max_message_id
```

↓

Receive all missed messages.

---

# The entire chapter in one picture

```text
Need Chat
        │
        ▼
Forward messages
        │
        ▼
Offline users lose messages
        │
        ▼
Store Messages
        │
        ▼
Need server-to-client communication
        │
        ▼
Polling
        │
        ▼
Too many useless requests
        │
        ▼
Long Polling
        │
        ▼
Timeouts + inefficiency
        │
        ▼
WebSocket
        │
        ▼
Need persistent servers
        │
        ▼
Stateful Chat Servers
        │
        ▼
Need server selection
        │
        ▼
Service Discovery
        │
        ▼
Need scalability
        │
        ▼
Multiple Chat Servers
        │
        ▼
Need notifications for offline users
        │
        ▼
Push Notifications
        │
        ▼
Need storage
        │
        ▼
Relational DB + Key-Value Store
        │
        ▼
Need message ordering
        │
        ▼
Message IDs
        │
        ▼
Need multi-device support
        │
        ▼
cur_max_message_id
        │
        ▼
Need Group Chat
        │
        ▼
Per-user Inbox Copies
        │
        ▼
Need Online Status
        │
        ▼
Presence Server
        │
        ▼
Temporary disconnects
        │
        ▼
Heartbeat
        │
        ▼
Friends need updates
        │
        ▼
Publish-Subscribe
        │
        ▼
Large groups become expensive
        │
        ▼
Fetch Presence On Demand
```

## The connecting thread

The chapter starts with the simplest idea: **forward messages between two users**. That immediately fails because users are often offline, so messages must be **persisted**. Next, the challenge becomes **real-time delivery**. HTTP works for sending but cannot push data to clients, leading from **Polling → Long Polling → WebSocket**, with WebSocket ultimately providing a persistent bidirectional connection. Persistent connections make chat servers **stateful**, so **service discovery** is introduced to assign users to appropriate servers. As the system scales, **push notifications** are added for offline users, and **key-value storage** is chosen because chat history involves massive, low-latency reads and writes. **Message IDs** guarantee ordering without relying on timestamps. Supporting **multiple devices** requires synchronization via each device's `cur_max_message_id`. Extending from one-to-one chats to **group chats** introduces per-recipient inbox copies, which work well for small groups but not massive ones. Finally, the system adds **presence servers**, **heartbeats**, and **publish-subscribe** so users can see who is online, while avoiding excessive updates in very large groups. Every concept in the chapter is introduced because the previous design encounters a scalability or real-time communication limitation.

