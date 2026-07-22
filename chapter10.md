# Design a Notification System

A notification system is a platform whose only responsibility is to reliably deliver information from a producer (service) to a user through multiple communication channels.

Examples:

* Push notification (iOS, Android)
* SMS
* Email

The notification system itself **does not deliver notifications directly**. It coordinates everything required to get the notification delivered by specialized providers like APNS, FCM, Twilio, SendGrid, etc.

---

# Why do we even need a notification system?

Imagine every microservice sends notifications itself.

```
Order Service ---> APNS
              ---> FCM
              ---> Twilio
              ---> Email Server

Payment Service ---> APNS
                ---> Twilio

Marketing Service ---> APNS
                  ---> Email
```

Immediately many problems appear.

* Every service has to understand APNS.
* Every service has to understand FCM.
* Every service has to understand SMS providers.
* Every service stores notification templates.
* Every service manages retries.
* Every service manages rate limits.
* Every service tracks delivery.
* Every service stores user preferences.

Every team duplicates the same code.

Now imagine changing SMS provider.

Instead of changing one place...

...you modify 40 services.

This is terrible architecture.

So we introduce a dedicated Notification System.

```
All Services
      |
      |
Notification Service
      |
 ----------------------------
 |      |        |          |
APNS   FCM    Twilio   SendGrid
```

Now every service simply says

> "Send this notification."

The notification system handles everything else.

---

# What are the requirements?

From the chapter requirements:

Supports

* Push notifications
* SMS
* Email

Must be

* Near real-time
* Millions of notifications/day
* Multiple devices
* Triggered by applications or scheduled jobs
* Users can opt out

These requirements drive the entire design.

Notice how every later design decision comes from one of these requirements.

---

# High Level Design

Before discussing servers, understand the complete lifecycle.

```
Business Event

↓

Notification Request

↓

Notification Processing

↓

Notification Delivery

↓

User Receives Notification
```

Everything in the chapter is simply expanding each stage.

---

# Step 1 — Notification Generation

Something must happen before a notification exists.

Examples

```
Order Placed

↓

Send Confirmation Email
```

```
Payment Failed

↓

Send SMS
```

```
Package Out for Delivery

↓

Push Notification
```

```
Breaking News

↓

Send Push Notification
```

These systems are called **notification producers**.

The chapter calls them

```
Service 1
Service 2
...
Service N
```

These are simply business services.

They do NOT send notifications.

They only request them.

---

# Step 2 — Different notification channels

Now the notification system decides

Which channel?

Because every channel works differently.

---

# iOS Push Notification

Apple controls iPhones.

You cannot directly send notifications to an iPhone.

Instead

```
Your Server

↓

Apple Push Notification Service (APNS)

↓

iPhone
```

Apple acts as a delivery network.

---

## Device Token

Question:

How does APNS know which phone?

Answer:

Every installation generates

```
Device Token
```

Think of it as

```
House Address
```

Without address

Courier cannot deliver.

Without device token

APNS cannot deliver.

---

# Payload

Payload contains

```
Title

Body

Badge

Sound

Custom Data
```

Example

```
{
 title:"Order Delivered",
 body:"Your order has arrived",
 orderId:123
}
```

APNS receives this payload.

It pushes it to the phone.

---

# Android

Exactly same idea.

Instead of APNS

Google provides

```
Firebase Cloud Messaging

(FCM)
```

Flow

```
Notification Server

↓

FCM

↓

Android Device
```

Same concept.

Different provider.

---

# SMS

Unlike push notifications,

phones don't expose APIs.

Instead companies use SMS gateways.

Example

```
Twilio

↓

Mobile Network

↓

Phone
```

Your server never talks to telecom operators.

Twilio does.

---

# Email

Same story.

Sending email yourself is difficult.

Problems

* Spam detection
* Blacklisting
* DKIM
* SPF
* Bounce handling
* Analytics

Instead use

```
SendGrid

MailChimp

Amazon SES
```

Again

Your notification system delegates delivery.

---

# Observation

Every notification channel has

Different API

Different authentication

Different retry

Different limits

Different response format

So the notification system acts as a common abstraction.

Business services never care whether it's APNS or Twilio.

---

# Contact Information Collection

Now another question appears.

How do we know

where to send notifications?

Suppose Order Service wants to notify User 123.

How does Notification System know

* phone number?
* email?
* device token?

It needs contact information.

---

# When is contact information collected?

When user

* signs up
* installs app
* logs in

The API server stores

```
Email

Phone Number

Device Tokens
```

---

# Database Design

User Table

```
User ID

Email

Phone Number
```

Device Table

```
Device ID

User ID

Platform

Device Token
```

Why separate tables?

Because

One user

↓

Many devices

Example

```
iPhone

iPad

Android Tablet
```

If device token were inside User table

Only one token could exist.

Normalization solves this.

```
User

↓

Device1

↓

Device2

↓

Device3
```

Now push notification goes to every device.

---

# Initial Notification Flow

The book first proposes a simple architecture.

```
Business Services

↓

Notification Server

↓

Third Party Services

↓

Users
```

Simple.

Easy.

Works.

But...

Only for small scale.

---

# Responsibilities of Notification Server

It

* receives request
* validates request
* builds payload
* contacts providers
* waits for response

Everything happens here.

This simplicity causes future problems.

---

# Problem 1 — Single Point of Failure

Only one notification server exists.

```
Service

↓

Notification Server

↓

Users
```

If server dies

Entire notification system dies.

No push.

No SMS.

No Email.

Complete outage.

---

# Problem 2 — Difficult to Scale

Suppose suddenly

Black Friday Sale.

Instead of

```
100 notifications/sec
```

Now

```
50,000 notifications/sec
```

One notification server now must

* query database
* fetch templates
* contact APNS
* contact Twilio
* contact SendGrid
* build HTML emails
* process retries

One machine cannot keep growing forever.

Vertical scaling eventually reaches hardware limits.

---

# Problem 3 — Performance Bottleneck

Different notification types have different speeds.

Push

```
20 ms
```

Email

```
500 ms
```

HTML generation

```
300 ms
```

Waiting for provider

```
1 second
```

One slow email blocks everything.

Eventually

CPU

Memory

Threads

Connections

all become bottlenecks.

---

# Improved Design

Instead of one overloaded server

Break responsibilities apart.

This is classic distributed system thinking.

```
Producer

↓

Notification Server

↓

Queue

↓

Workers

↓

Provider

↓

User
```

Every box has one responsibility.

---

# Database and Cache moved out

Instead of embedding storage

```
Notification Server

↓

Database
```

becomes

```
Notification Servers

↓

Shared Database
```

Now many notification servers can run.

---

# Horizontal Scaling

Earlier

```
1 Server
```

Now

```
LB

↓

Notification 1

Notification 2

Notification 3

Notification 4
```

Traffic spreads.

Need more capacity?

Add servers.

No code changes.

---

# Why Cache?

Notification processing repeatedly needs

* user info
* device token
* templates
* notification settings

Reading DB every time becomes expensive.

Instead

```
Notification Server

↓

Cache

↓

Database
```

Most reads hit cache.

Benefits

* lower latency
* lower DB load
* better throughput

---

# Why Message Queue?

This is the most important improvement.

Without queue

```
Notification Server

↓

APNS
```

Server waits.

Cannot continue.

With queue

```
Notification Server

↓

Queue

↓

Worker

↓

APNS
```

Notification server immediately returns.

Worker handles slow work later.

This is called **asynchronous processing**.

---

# Why separate queues?

The chapter gives

```
Push Queue

SMS Queue

Email Queue
```

Instead of

```
One Queue
```

Why?

Suppose

Twilio goes down.

SMS workers stop.

If everything shares one queue

SMS backlog fills queue.

Push notifications also get delayed.

Entire system slows.

Separate queues isolate failures.

```
SMS Failure

↓

Only SMS Queue grows

↓

Push Queue unaffected

↓

Email Queue unaffected
```

This is fault isolation.

---

# Workers

Workers continuously poll queues.

```
while(true)

↓

Read Queue

↓

Send Notification

↓

Acknowledge
```

Need more throughput?

Add workers.

Simple horizontal scaling.

---

# Complete Improved Flow

Everything now connects naturally.

```
Business Service

↓

Notification API

↓

Validation

↓

Fetch User

↓

Fetch Template

↓

Fetch Settings

↓

Queue

↓

Worker

↓

Provider

↓

User
```

Every step exists because the previous step alone wasn't enough.

---

# Reliability

Distributed systems fail.

Provider fails.

Worker crashes.

Network breaks.

Database unavailable.

So reliability becomes critical.

---

# Preventing Data Loss

Requirement

Notifications may be delayed.

But

Must never disappear.

Suppose worker crashes after dequeuing.

Without persistence

Notification gone forever.

Solution

Store notification event.

```
Notification Created

↓

Notification Log Database

↓

Queue

↓

Worker
```

Now even if queue loses message

Database still contains original event.

Recovery possible.

---

# Exactly Once Delivery

Can we guarantee

```
Exactly once?
```

No.

Because failures happen between every network call.

Example

Worker sends notification.

Provider receives it.

Provider sends success response.

Network breaks.

Worker never receives response.

Worker thinks

"Failed."

Retries.

User gets duplicate.

Distributed systems cannot perfectly distinguish

```
Did provider receive?

OR

Did response disappear?
```

Impossible in general.

---

# Deduplication

Instead of perfect guarantee

Reduce duplicates.

Every notification has

```
Event ID
```

Worker checks

```
Seen?

↓

Yes

Discard

↓

No

Process
```

Duplicate retries become harmless.

---

# Notification Templates

Millions of notifications have same format.

Instead of

building entire notification every time

Store template.

```
Hello {name}

Your order {orderId}
has shipped.
```

Runtime becomes

```
Template

+

Variables

↓

Final Notification
```

Benefits

* consistency
* easier maintenance
* fewer mistakes
* localization
* faster generation

---

# Notification Settings

Users don't want every notification.

Store preferences.

```
User

↓

Push = Yes

SMS = No

Email = Yes
```

Before queueing

Notification Server checks

```
Opted In?

↓

Yes

Continue

↓

No

Drop
```

No unnecessary provider calls.

Better user experience.

---

# Rate Limiting

Suppose buggy service generates

```
10,000 notifications
```

for same user.

Without limit

User receives

```
10,000 pushes
```

Probably uninstalls app.

Rate limiter protects users.

Example

```
Maximum

5 promotional notifications/day
```

System simply rejects excess.

---

# Retry Mechanism

Provider unavailable.

Should we fail forever?

No.

Worker

```
Attempt 1

↓

Fail

↓

Queue Again

↓

Attempt 2

↓

Fail

↓

Attempt 3

↓

Success
```

After maximum retries

Generate alert.

Developers investigate.

---

# Authentication

Anyone should not send notifications.

Otherwise attackers could spam users.

Notification APIs are protected.

Only trusted internal services

or authenticated clients

can call

```
POST /send
```

---

# Monitoring Queue

Queue length tells system health.

Small queue

```
Workers keeping up.
```

Growing queue

```
Workers too slow.
```

Very large queue

```
Need more workers.

OR

Provider slow.

OR

Worker failure.
```

Queue size becomes an important operational metric.

---

# Event Tracking

Delivery alone isn't enough.

Business wants answers.

Did user

* receive?
* open?
* click?
* purchase?

Notification system emits events.

```
Delivered

↓

Opened

↓

Clicked

↓

Purchased
```

Analytics uses this.

Marketing improves campaigns.

---

# Final Architecture

Everything is now connected in one continuous pipeline.

```
Business Services
        │
        ▼
Notification API
        │
        ▼
Authentication
        │
        ▼
Validation
        │
        ▼
Fetch User Information
        │
        ▼
Fetch Notification Settings
        │
        ▼
Load Notification Template
        │
        ▼
Generate Notification Payload
        │
        ▼
Persist Notification Log
        │
        ▼
Apply Rate Limiting
        │
        ▼
Publish to Channel Queue
        │
        ├──────── Push Queue
        ├──────── SMS Queue
        └──────── Email Queue
                  │
                  ▼
              Workers
                  │
                  ▼
Retry on Failure
                  │
                  ▼
Third-party Providers
(APNS / FCM / Twilio / SendGrid)
                  │
                  ▼
User Devices
                  │
                  ▼
Delivery Events
                  │
                  ▼
Analytics & Monitoring
```

# How every topic connects

The chapter is not a collection of independent concepts. Each topic exists because the previous design exposed a limitation:

1. **Business services generate events** → something needs to notify users.
2. **Different channels (Push/SMS/Email)** → require a common notification service because each provider has different APIs.
3. **Providers need destinations** → therefore collect and store user contact information and device tokens.
4. **A single notification server works initially** → but creates SPOF, scaling issues, and performance bottlenecks.
5. **To remove those bottlenecks** → separate storage, add multiple notification servers, caches, and asynchronous message queues.
6. **Queues require consumers** → workers are introduced to process notifications independently and in parallel.
7. **Distributed systems can fail** → add persistent notification logs so notifications are never lost.
8. **Retries introduce duplicates** → add deduplication using notification/event IDs.
9. **Generating every notification manually is inefficient** → introduce reusable notification templates.
10. **Users may not want every notification** → store per-channel notification preferences and check them before sending.
11. **Even opted-in users can be overwhelmed** → apply rate limiting to control notification frequency.
12. **Third-party providers are unreliable** → implement retries and alerting for persistent failures.
13. **Notification APIs could be abused** → protect them with authentication and authorization.
14. **Queues can become backlogged** → monitor queue depth and scale workers horizontally when necessary.
15. **Delivery is not enough for the business** → track opens, clicks, and engagement through analytics.

Every improvement is driven by one of four fundamental distributed-system goals:

* **Scalability** (horizontal notification servers, workers, queues, cache)
* **Reliability** (persistent logs, retries, deduplication)
* **Extensibility** (channel abstraction, templates, pluggable providers)
* **Operational visibility** (monitoring, metrics, event tracking)

That progression is the core design story of the chapter.

