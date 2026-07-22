# These are the notes for Chapter 10 - Designing a Notification System

This chapter looks simple because everyone has used notifications.

But designing a notification system that sends **millions of notifications every day** is much harder than it looks.

The mistake most people make is thinking this chapter is about Push Notifications.

It is not.

It is about **building a distributed system whose job is to reliably deliver messages to users through different communication channels.**

Just like the Key-Value Store chapter, think of this chapter as **one continuous story**.

Every new concept appears because the previous solution created a new problem.

---

# Stage 1 - The simplest notification system

Suppose your application only sends emails.

Your architecture looks like this.

```text
Order Service

        │
        ▼

Send Email()

        │
        ▼

SMTP Server

        │
        ▼

User
```

Whenever an order is placed,

```
Order Service

↓

sendEmail(user)
```

Done.

Very simple.

---

## Problem #1

Now Product Team says

"We also need Push Notifications."

Your Order Service becomes

```text
Order Service

 ├── Email
 ├── Push
```

Few weeks later

Marketing says

```
Need SMS.
```

Now

```text
Order Service

 ├── Email
 ├── Push
 ├── SMS
```

Another month later

```
Need WhatsApp Notifications.
```

Now

```text
Order Service

 ├── Email
 ├── Push
 ├── SMS
 ├── WhatsApp
```

Every business service now needs to understand every notification channel.

Imagine having

```
20 microservices
```

Each one now contains

* Email logic
* Push logic
* SMS logic
* Retry logic
* Authentication
* Analytics

Huge duplication.

---

## Solution

Separate notification logic.

Create one Notification Service.

```
Order Service
Billing Service
Marketing Service
Inventory Service

        │
        │
        ▼

Notification Service

        │
        ▼

Users
```

Now every service simply says

```
Send Notification
```

The Notification Service decides

* Push
* SMS
* Email

This is the **real beginning** of the chapter.

---

# Stage 2 - Notification Service still cannot send notifications

Suppose someone says

```
Send Push Notification
```

Can your Notification Service directly send data to an iPhone?

No.

Apple does not allow that.

Similarly,

Android also doesn't allow that.

SMS also doesn't.

Email also doesn't.

Every communication channel has its own delivery network.

---

## iOS

Apple provides

```
APNS

Apple Push Notification Service
```

Flow

```
Notification Service

↓

APNS

↓

iPhone
```

Your server never talks directly to the phone.

Apple delivers it.

---

## Android

Exactly same idea.

Instead of APNS,

Google provides

```
Firebase Cloud Messaging

(FCM)
```

Flow

```
Notification Service

↓

FCM

↓

Android Phone
```

---

## SMS

Phones don't expose APIs.

Telecom companies don't expose APIs.

Instead,

companies use SMS Gateways.

Example

```
Twilio

Nexmo
```

Flow

```
Notification Service

↓

Twilio

↓

Mobile Network

↓

Phone
```

---

## Email

Email is also similar.

Companies usually don't maintain email infrastructure.

Instead they use

```
SendGrid

Amazon SES

MailChimp
```

Flow

```
Notification Service

↓

Email Provider

↓

Recipient
```

---

### Important Observation

Notice something.

Every channel has

* Different API
* Different Authentication
* Different Payload
* Different Retry Logic
* Different Limits

The Notification Service hides all these differences.

Business Services don't know anything about APNS or Twilio.

---

# Stage 3 - How do we know where to send notifications?

Suppose Order Service says

```
Notify User 123
```

Question

Where?

```
Email?

Phone?

Device?

```

Notification Service doesn't know.

It only has

```
User ID
```

It needs user contact information.

---

## Solution

Collect user information during registration.

Whenever a user

* installs app
* signs up
* logs in

store

```
Email

Phone Number

Device Token
```

---

## Database

```
User Table

-----------------

UserID

Email

Phone Number
```

Another table

```
Device Table

-----------------

DeviceID

UserID

Platform

Device Token
```

---

### Why two tables?

Because

One user

↓

Many devices

```
Laptop

iPhone

Android Tablet

iPad
```

If Device Token were inside User table,

only one device could exist.

Separate Device table solves it.

---

# Stage 4 - One Notification Server

Everything looks good.

Architecture

```
Business Services

↓

Notification Server

↓

APNS

FCM

Twilio

SendGrid
```

Looks perfect.

Works perfectly.

Until scale arrives.

---

# Problem #2

Suppose your application grows.

Daily notifications

```
10 Million Push

5 Million Emails

1 Million SMS
```

One Notification Server must

* Validate request
* Query database
* Read templates
* Build HTML Email
* Build Push Payload
* Call APNS
* Wait for APNS
* Call Twilio
* Wait for Twilio
* Retry failures

Everything happens on one machine.

CPU becomes busy.

Threads become blocked.

Latency increases.

---

## Problem #3

Single Point of Failure.

```
Business Services

↓

Notification Server

↓

Users
```

Notification Server crashes.

Everything stops.

No notification reaches anyone.

---

## Solution

Horizontal Scaling.

Instead of

```
1 Notification Server
```

Run

```
Load Balancer

      │

────────────────────

│      │      │

N1     N2     N3
```

Now traffic gets distributed.

Need more capacity?

Add more servers.

---

# Stage 5 - Another problem appears

Even after adding more Notification Servers,

one issue still remains.

Suppose Notification Server sends notification.

```
Notification Server

↓

APNS
```

APNS takes

```
700 milliseconds
```

During that time

Notification Server waits.

It cannot process other notifications.

Thousands of requests pile up.

Notification Servers become blocked.

---

## Solution

Message Queue.

Instead of sending immediately

```
Notification Server

↓

Queue

↓

Worker

↓

APNS
```

Notification Server simply stores event inside Queue.

Returns immediately.

Worker performs slow work.

This is called

```
Asynchronous Processing
```

This is probably the biggest improvement in the chapter.

---

# Stage 6 - Why multiple queues?

Suppose there is only one queue.

```
Notification Queue
```

Now

Twilio crashes.

SMS messages cannot leave queue.

Queue fills.

Soon

Email Notifications

also wait behind SMS.

Push Notifications

also wait.

Entire system slows down.

---

## Solution

Separate queues.

```
Push Queue

SMS Queue

Email Queue
```

Now

If Twilio fails

Only

```
SMS Queue
```

grows.

Push continues.

Email continues.

Failures become isolated.

---

# Stage 7 - Workers

Someone has to consume queues.

Introduce Workers.

```
Queue

↓

Worker

↓

Provider
```

Worker continuously does

```
while(true)

{

Read Queue

Send Notification

Delete Message

}
```

Need more throughput?

Simply add more workers.

```
Push Queue

↓

Worker1

Worker2

Worker3

Worker4
```

Very easy horizontal scaling.

---

# Stage 8 - What if worker crashes?

Imagine

Worker removes notification from queue.

Before sending,

machine crashes.

Notification disappears forever.

Customer never receives OTP.

Very dangerous.

---

## Solution

Persist notifications.

Store every notification before processing.

```
Notification Request

↓

Notification Log Database

↓

Queue

↓

Worker
```

Now

Even if queue loses message,

database still contains original event.

Notification can be recovered.

This gives reliability.

---

# Stage 9 - Third-party providers fail

Suppose

```
APNS

↓

Unavailable
```

Should we immediately fail?

No.

Failures are often temporary.

---

## Solution

Retry Mechanism.

```
Attempt 1

↓

Fail

↓

Wait

↓

Attempt 2

↓

Fail

↓

Wait

↓

Attempt 3

↓

Success
```

Most temporary failures disappear after retry.

---

# Stage 10 - Retries create another problem

Imagine

Worker sends notification.

APNS receives it.

APNS sends success response.

Network fails before response reaches Worker.

Worker thinks

```
Notification Failed.
```

Retries.

User receives

```
Two notifications.
```

Distributed systems cannot guarantee

```
Exactly Once Delivery.
```

---

## Solution

Deduplication.

Every notification gets

```
Event ID
```

Whenever Worker processes notification

```
Already Seen?

↓

Yes

Discard

↓

No

Send
```

Duplicates reduce significantly.

Exactly-once delivery is practically impossible in distributed systems, so we settle for **at-least-once delivery with deduplication**.

---

# Stage 11 - Millions of notifications look identical

Imagine sending

```
Your order has shipped.

```

to

```
10 million users.
```

Building message every time wastes CPU.

---

## Solution

Templates.

Store

```
Hello {Name}

Your Order {OrderID}

has shipped.
```

At runtime

```
Template

+

Variables

↓

Final Notification
```

Benefits

* Faster
* Consistent
* Easier localization
* Easier maintenance

---

# Stage 12 - Users get irritated

Suppose Marketing sends

```
20 Push Notifications

10 Emails

8 SMS
```

every day.

Users uninstall app.

---

## Solution

Notification Settings.

Store

```
User

↓

Push = Yes

Email = No

SMS = Yes
```

Before sending

Notification Server checks

```
Opted In?

↓

Yes

Continue

↓

No

Discard
```

Users remain in control.

---

# Stage 13 - Even opted-in users can be spammed

Suppose user has opted in.

Marketing accidentally sends

```
500 notifications.
```

Opt-in won't stop it.

Need another protection.

---

## Solution

Rate Limiting.

Example

```
Maximum

5 Promotional Pushes

per day
```

Additional notifications are rejected or delayed.

This protects users from spam and protects providers from unnecessary traffic.

---

# Stage 14 - Anyone can call Notification API

Imagine

```
POST

/sendNotification
```

is public.

Attackers send

```
50 million notifications.
```

Huge disaster.

---

## Solution

Authentication.

Only trusted internal services

or verified clients

can call Notification APIs.

For Push Providers,

credentials like

```
App Key

App Secret
```

are also required.

---

# Stage 15 - How do we know system is healthy?

Suppose Queue contains

```
20 million messages.
```

Nobody notices.

Users receive notifications

after

```
6 hours.
```

Need monitoring.

---

## Solution

Monitor queue length.

```
Queue Size

↓

Growing?

↓

Workers insufficient

↓

Add more Workers
```

Queue depth becomes one of the most important operational metrics.

---

# Stage 16 - Business wants analytics

Sending notification isn't enough.

Product Team asks

```
How many users opened it?

How many clicked?

How many purchased?
```

Need event tracking.

---

## Solution

Analytics.

Track

```
Sent

↓

Delivered

↓

Opened

↓

Clicked

↓

Purchased
```

These metrics help improve engagement and marketing campaigns.

---

# Stage 17 - Final architecture

Now every concept comes together.

```text
                    Business Services
        (Order, Billing, Marketing, etc.)
                          │
                          ▼
                Notification API Servers
        (Authentication + Validation + Rate Limit)
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
       Cache             Database     Notification Templates
          │               │
          └───────────────┘
                  │
                  ▼
        Create Notification Event
                  │
                  ▼
        Persist Notification Log
                  │
                  ▼
      ┌───────────┼────────────┐
      │           │            │
      ▼           ▼            ▼
 Push Queue    SMS Queue    Email Queue
      │           │            │
      ▼           ▼            ▼
   Workers     Workers      Workers
      │           │            │
      ▼           ▼            ▼
    APNS        Twilio      SendGrid
      │           │            │
      ▼           ▼            ▼
   iPhone      Phone SMS      Email
                  │
                  ▼
      Analytics & Monitoring
```

---

# Stage 18 - How a notification actually travels

Suppose your Amazon order is shipped.

```
Order Shipped
```

↓

Order Service generates event.

↓

Calls Notification API.

↓

Notification Server authenticates request.

↓

Checks user's notification settings.

↓

Fetches email, phone number and device tokens.

↓

Loads notification template.

↓

Creates final payload.

↓

Stores notification in Notification Log.

↓

Places notification into Push Queue.

↓

Worker reads Push Queue.

↓

Worker sends request to APNS.

↓

APNS delivers notification to iPhone.

↓

User's phone vibrates.

↓

App reports

```
Delivered
```

↓

User opens notification.

↓

Analytics records

```
Opened
```

↓

Business dashboard updates.

---

# The entire chapter in one picture

```text
Need notifications
        │
        ▼
Business services send directly
        │
        ▼
Too many notification channels
        │
        ▼
Notification Service
        │
        ▼
Need delivery providers
(APNS / FCM / SMS / Email)
        │
        ▼
Need user contact information
        │
        ▼
Single Notification Server
        │
        ▼
SPOF + Performance Bottleneck
        │
        ▼
Multiple Notification Servers
        │
        ▼
Servers block while sending
        │
        ▼
Message Queues
        │
        ▼
Need consumers
        │
        ▼
Workers
        │
        ▼
Workers can fail
        │
        ▼
Notification Log (Persistence)
        │
        ▼
Third-party failures
        │
        ▼
Retry Mechanism
        │
        ▼
Retries create duplicates
        │
        ▼
Deduplication
        │
        ▼
Millions of similar messages
        │
        ▼
Notification Templates
        │
        ▼
Users don't want every notification
        │
        ▼
Notification Settings
        │
        ▼
Still too many notifications
        │
        ▼
Rate Limiting
        │
        ▼
Need secure APIs
        │
        ▼
Authentication
        │
        ▼
Need operational visibility
        │
        ▼
Monitoring + Analytics
```

This is the connecting thread that the book doesn't explicitly explain. Every concept is introduced because the previous solution exposes a limitation. Once you see the chapter as **Problem → Solution → New Problem**, the entire notification system becomes a single logical evolution rather than a list of unrelated components.

