# System Design of a Notification System

Source video: [Design Notifications System Design](https://www.youtube.com/watch?v=e8cX9pQdu7Y) (Hindi, about 52 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow. Building a notification system takes a genuinely large amount of engineering effort. It isn't just "call an email API", it involves queues, pub/sub systems, retry logic, and an entire decision layer around whether a notification should even be sent in the first place.

---

## 1. The Naive, Synchronous Starting Point

Imagine a simple social media application. When a user signs up, you want to trigger a welcome email.

```
User
  |
  |  POST /signup  { firstName, lastName, email, password }
  v
Server:
   1. Insert the user into the database
   2. Call sendEmail() directly (talks to Gmail's API, or any email provider)
   3. Return "Welcome to the platform" response
```

This works. It is a very simple, synchronous architecture. But it does not scale, for three separate reasons.

**Problem 1: Added latency.** Calling an external email provider's API takes real time (the video uses an example of about 3 seconds). Since this call happens synchronously, in the middle of the request, every single sign-up now takes at least those extra few seconds, on top of the database write itself.

**Problem 2: External dependency downtime.** Your sign-up flow now depends on a third-party service (Gmail's API, AWS SES, SendGrid, or whichever provider you use). If that external service happens to be down or unresponsive, even briefly, your `sendEmail()` call fails, and because it's a synchronous, blocking call, your entire sign-up flow fails along with it. Users would be unable to create an account, purely because a welcome email couldn't be sent. That's a bad trade-off: losing a user over a non-critical notification.

**Problem 3: Rate limiting (the problem you will hit 100% of the time).** Every external notification provider enforces some form of **rate limiting**: a cap on how many requests you can send in a given time window. For example, a provider might allow only 30 emails per second.

> **Example:** Suppose 100 users sign up at almost the exact same moment, meaning your server tries to call the email API 100 times in that instant. Only 30 of those calls succeed, since that's the provider's rate limit. The remaining 70 calls fail immediately. Since this is a synchronous call, those 70 users' sign-up requests fail too, even though their accounts might have already been correctly created in the database. Purely because of the email provider's rate limit, real users lose the ability to sign up.

This is the core problem with a fully synchronous architecture: a non-critical side effect (sending an email) is tightly coupled to, and capable of breaking, a critical operation (account creation).

---

## 2. First Improvement: Offload to an Internal Email Server

A first instinct is to move the email-sending logic to a separate, internal server, rather than calling the external provider directly from the main sign-up route.

```
Sign-up Server
   |
   |  "Send this email request to the Email Server"
   v
Email Server (internal, not publicly exposed)
   |
   |  Email Server talks to Gmail's API (or whichever provider)
   v
Email is sent (or fails, if the provider is down)
```

Since the request from the sign-up server to the internal email server is now an **internal** call (fully within your own infrastructure), it essentially never fails on its own. This solves the crash-propagation problem: the sign-up flow itself no longer breaks just because the external email provider is having issues.

**But the rate-limiting problem is still there.** If 100 users sign up at once, the email server itself will still try to call the external provider 100 times, hit the same 30-per-second rate limit, and the same 70 emails will silently fail to send. The difference now is that sign-up itself succeeds for all 100 users; it's only the welcome email that's missing for 70 of them. This is a real improvement (users can still sign up), but it's not yet a scalable design, and those 70 missing emails are still a problem worth solving.

---

## 3. Introducing a Queue

Instead of sending the email request directly to the email server, you can place it into a **queue** first.

> **Definition:** A queue, in its simplest form, is a First-In-First-Out (FIFO) data structure. You add ("enqueue") items to it, and something else picks them up later to process them.

```
Sign-up Server
   |
   |  enqueue: "send welcome email to user X"
   v
Queue: [msg1, msg2, msg3, ...]
   |
   |  a worker watches the queue and picks up messages one at a time
   v
Worker --> calls the email provider's API --> email sent
```

The email server is no longer really a "server" in the traditional sense; it's now a **worker** that continuously watches the queue and processes messages from it.

**Why this helps:**

1. **Enqueueing is fast.** Adding a message to a queue is a constant-time operation (O(1)). The sign-up route no longer waits for the actual email to be sent; it just drops a message into the queue and immediately returns a response. This removes the added latency problem entirely.
2. **Failed messages can be retried.** If the worker picks up a message and the email provider is down at that moment, the send fails. Instead of discarding it, the worker can simply put the message back into the queue to try again later.
3. **You can now control your own rate, proactively.** Since you know your provider's limit (say, 30 emails per second), the worker can deliberately pace itself: pull and send messages in a controlled batch, then pause briefly (say, for a couple of seconds) before pulling more. In practice, teams often build in a safety margin, for example treating a 30-per-second limit as if it were only 25 per second, so they never actually get close to the real limit.

### A More Professional Version of This Architecture

```
Users
  |
  v
Reverse Proxy
  |
  v
Load Balancer
  |
  v
Application Servers (auto-scaled)
  |
  |  on sign-up, enqueue a message
  v
Queue (e.g. AWS SQS)
  |
  |  a worker (e.g. an EC2 instance) watches the queue
  v
Worker --> AWS SES (or any email provider) --> email sent
```

This gives you a decent, scalable system for sending a single kind of notification (a welcome email), complete with retry capability and rate-limit control.

---

## 4. Avoiding Infinite Retries: Exponential Backoff and Dead Letter Queues

There's a subtle problem with simply "retry forever." If the email provider happens to be down for an extended period (say, an entire day), naively retrying a failed message immediately, over and over, creates an effectively infinite loop that wastes compute for no benefit, since the message clearly isn't going to succeed any time soon.

> **Definition:** Exponential backoff is a retry strategy where, after each failed attempt, you wait progressively longer before trying again (for example, retry after 1 second, then 2 seconds, then 4, then 8, doubling each time), instead of retrying immediately and repeatedly.

```
Attempt 1: fails -> wait 1 second
Attempt 2: fails -> wait 2 seconds
Attempt 3: fails -> wait 4 seconds
Attempt 4: fails -> wait 8 seconds
...and so on
```

This is usually paired with a **maximum retry count** (for example, 10 attempts), so that a message doesn't get retried forever.

> **Definition:** A Dead Letter Queue (DLQ) is a separate, "backup" queue where messages are placed once they've exhausted their maximum retry attempts, instead of being silently discarded.

```
Message fails 10 times (with exponential backoff between attempts)
   |
   v
Message is moved to the Dead Letter Queue (DLQ)
   |
   |  a developer or internal team can manually inspect it later
   v
If the underlying issue is fixed, the message can be manually
re-enqueued into the main queue to be retried
```

The DLQ acts like a log or an audit trail for failed messages, giving your team the chance to investigate why something failed and decide whether to retry it, rather than losing that message entirely.

---

## 5. A New Requirement: Multiple Events, Multiple Channels

So far, this system only handles one event (sign-up) and one channel (email). Real products need much more: an email on login, an in-app notification when a friend request arrives, a push notification for something else, and so on.

**The naive approach:** go into every single route in your codebase and add the relevant queue-sending logic directly.

```
POST /signup   --> enqueue to Email Queue
POST /login    --> enqueue to Email Queue
POST /post     --> enqueue to In-App Queue
POST /friend-request --> enqueue to Push Notification Queue
```

This works, but it creates **tight coupling** between your application code and your notification channels. Every time a product manager asks for a new notification channel, or wants to change which channel an event uses, you have to go back into multiple different routes across your codebase and edit them directly. This gets messy fast, and it does not scale well from a maintainability point of view, even though the underlying queues themselves are scalable.

---

## 6. The Fix: Event-Driven Architecture and Fan-Out

Instead of your application code deciding exactly which queue to push into, you can have it simply **announce that something happened**, and let a separate system decide what to do about it.

> **Analogy:** Imagine a teacher making an announcement in a classroom, instead of going up to each individual student to tell them personally. Whoever the announcement is relevant to can act on it; everyone else can simply ignore it. This is different from a queue, where one specific consumer picks up one specific message. Here, one event can be heard, and acted on, by many different, independent listeners at once.

> **Definition:** In an event-driven architecture, your application raises ("publishes") events, such as `user.signup`, `user.login`, `user.sendRequest`, or `user.post`, to a central pub/sub (publish-subscribe) system. Any number of independent subscribers can listen for specific events and react to them however they see fit, without your original application code needing to know anything about those subscribers.

The AWS equivalent of this pub/sub system is **SNS (Simple Notification Service)**.

```
Application Servers
   |
   |  publish an event, e.g. "user.signup"
   v
SNS (Topic-based Pub/Sub)
   |--> bound to Email Queue         --> Email Worker --> SES
   |--> bound to In-App Queue        --> In-App Worker
   |--> bound to Push Notification Queue --> Push Worker --> Firebase
```

Now, your application code only ever needs one line: "publish this event to SNS." SNS topics are then **bound** ("subscribed") to whichever queues are relevant for that event, and this binding is just configuration, not application code.

> **Example:** If product requirements change (say, "friend requests should also trigger an email, not just a push notification"), you don't touch your application routes at all. You simply update the topic's subscriptions in SNS to also route to the Email Queue. Your code stays exactly the same.

> **Definition:** This overall pattern, where one published event gets distributed out to multiple independent queues or services at once, is called **fan-out architecture**. AWS itself documents this exact pattern (commonly demonstrated with an S3 upload event published to an SNS topic, which then fans out to multiple SQS queues and/or AWS Lambda functions).

An SNS topic isn't limited to feeding queues either. The same event can also directly trigger a Lambda function, or invoke a server directly, alongside (or instead of) pushing into a queue. This makes SNS act like a flexible gateway for distributing one event to many different kinds of downstream consumers.

---

## 7. The Hardest Problem: Deciding Whether to Send a Notification At All

Even with a solid delivery pipeline in place, there's a deeper, more subtle problem: **not every event should actually generate a notification.**

> **Example:** Imagine you're actively using a chat application like Slack, and you're online, actively reading a fast-moving conversation. If the system emailed you for every single incoming message while you were already reading them live, you'd be overwhelmed and frustrated. You clearly don't need an email notification for messages you're already seeing in real time.

This is fundamentally different from the delivery problem discussed so far. It's not about *how* to deliver a notification reliably, it's about *whether* a notification should be triggered in the first place. This decision-making layer is often referred to as **ingestion**.

**A simple mechanism for this:** track whether a user is currently active, for example by storing a "last active" timestamp per user in a fast store like Redis.

```
Redis: user:piyush:lastActive = "2 minutes ago"

New chat messages arrive for Piyush
   |
   |  check Piyush's last active timestamp
   v
If very recently active (e.g. within the last couple of minutes):
   -> skip sending a notification (he's likely already seeing the messages live)

If not recently active (e.g. hasn't been active in 2 days):
   -> trigger a notification: "You have several new messages, please check them"
```

**User notification preferences also matter.** Most platforms let users turn specific notification types on or off (for example, disabling emails while keeping push notifications on). Before triggering any notification, the system needs to check these stored preferences too, alongside things like whether the user is currently online, and whether the specific channel in question is even enabled for that event.

### A Real-World Example: Slack's Notification Decision Logic

The video references Slack's own publicly documented flowchart for deciding whether to send a notification, which considers a long chain of conditions, roughly along these lines:

```
Is the channel muted?              -> if yes, don't notify
Is the user in Do Not Disturb mode? -> if yes, don't notify
Was the user directly mentioned (@user, @channel, @everyone)?
   -> if yes (and mentions aren't suppressed in that channel's settings), notify
Is this a thread the user has subscribed to?
   -> if yes, notify
Otherwise, depending on more granular channel and notification
preferences, the system decides whether to notify or stay silent
```

On top of this, the delivery channel itself can be conditional: if a user is active on their mobile device, a push notification might be sent there instead of triggering an email, since an email may simply not be necessary if the user is already reachable through another, more immediate channel.

**Other factors that influence whether and how a notification is sent:**

- Whether the user is currently online or offline.
- The user's own notification channel preferences.
- Rate limits imposed by the delivery channel itself (as discussed earlier).
- How important the specific message is. For example, an OTP (one-time password) message is critical and should always be delivered promptly, whereas a promotional message is not, and can be treated with much lower priority or skipped entirely under certain conditions.

---

## 8. Existing Tools: Novu and Courier

Building all of this (queues, fan-out, retries, dead letter queues, ingestion logic, per-user preferences, multi-channel delivery) from scratch takes months of engineering effort. Because of this, many smaller companies and early-stage startups rely on existing notification infrastructure platforms rather than building their own from scratch. Two examples mentioned in the video:

- **Novu**: an open source notification infrastructure platform that provides much of this pipeline (ingestion, multi-channel delivery, queuing) out of the box.
- **Courier**: a closed-source, commercial notification API service that solves a similar problem.

Both exist precisely because designing and scaling a full notification system, one that reliably delivers across email, SMS, WhatsApp, in-app messages, and push notifications, while also correctly deciding when *not* to notify someone, is a substantial engineering undertaking on its own.

---

## Summary: Building Up a Notification System

| Stage | What Was Added | Problem It Solved | Problem That Remained |
|---|---|---|---|
| 1. Synchronous call | Direct call to an email API inside the sign-up route | Simplicity | Added latency, external downtime crashes sign-up, rate limits break sign-up |
| 2. Internal email server | Offload the email call to an internal server | Sign-up no longer crashes from external downtime | Rate-limited emails still silently fail to send |
| 3. Queue + Worker | Enqueue email requests; a worker processes them at a controlled pace | Fast enqueueing, controlled rate, retryable failures | Long outages could cause endless immediate retries |
| 4. Exponential Backoff + DLQ | Progressive retry delays, plus a dead letter queue for exhausted retries | Avoids wasted compute on doomed retries; failures are inspectable, not lost | Adding new event types still meant editing code everywhere |
| 5. Event-Driven Architecture (SNS) + Fan-Out | Publish events once; let subscriptions decide which queues/services react | Decouples application code from specific notification channels | Still doesn't decide *whether* a notification should be sent |
| 6. Ingestion Logic | Checks for online status, user preferences, mentions, DND, message importance, and so on | Prevents notification spam and irrelevant alerts | Genuinely complex to get right; often outsourced to platforms like Novu or Courier |

**Key takeaway:** a production-grade notification system is really two separate problems layered on top of each other: reliably **delivering** a notification once you've decided to send one (queues, retries, fan-out, rate limiting), and separately, correctly **deciding** whether a notification should be sent at all (online status, preferences, mentions, message importance). Both halves require real engineering effort, which is why dedicated notification infrastructure platforms exist.
