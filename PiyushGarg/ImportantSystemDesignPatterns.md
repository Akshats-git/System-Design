# System Design Patterns You Should Master Right Now

Source video: [System Design Patterns you should Master Right Now](https://www.youtube.com/watch?v=OdNpY3WQniQ) (Hindi, about 20 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow. This video is a quick-fire overview of several important system design patterns. Two of them, event sourcing and CQRS, already have their own dedicated, much more detailed notes in this folder: [EventSourcing.md](EventSourcing.md) and [CQRS.md](CQRS.md). This file summarizes those two briefly and covers the newer patterns from this video (microservices vs. monolith, database-per-service, and circuit breaker) in full detail.

---

## 1. Microservices vs. Monolith

This is usually the very first architectural decision you need to make in any system design: should you build a **microservice architecture** or a **monolith architecture**?

> **Definition:** In a microservice architecture, each feature of your application is broken out into its own separate, independent service. For example, you might have a dedicated Authentication Service, a separate Notification Service (perhaps written in a completely different language, like Rust, and deployed on a different cloud), a Post Service, and a Comment Service, each with a single, focused responsibility.

```
Authentication Service   (handles only auth)
Notification Service     (handles only notifications)
Post Service              (handles only posts)
Comment Service           (handles only comments)
```

**Benefit:** each service can be written, coded, deployed, and scaled independently, since they are isolated from one another.

**Trade-off:** you now have to manage many separate servers/services instead of just one, and you need to figure out how these services actually communicate with each other (for example, how does the Comment Service notify the Notification Service that a new comment was posted?).

> **Definition:** In a monolith architecture, there's a single, large server containing your entire codebase: authentication routes, notification handlers, comment routes, and everything else, all together in one place.

```
Single Server
   |--> Authentication routes
   |--> Notification handlers
   |--> Comment routes
   |--> ...everything else
```

You can still scale a monolith (by running multiple instances of the whole thing), but you can't scale just one piece of it. You can't say "I only want to scale authentication"; either the entire server scales, or it doesn't.

**The monolith's biggest risk:** since everything lives in one process, a bug in one part (say, the notification system) can crash the **entire** server, taking down authentication, comments, and everything else along with it. In a microservice architecture, if the Notification Service goes down, the Authentication Service and Comment Service can keep working completely unaffected.

> **Trade-off summary:** monoliths are simpler to manage but risk a single bug bringing down everything. Microservices isolate failures and allow independent scaling, but add real complexity in managing multiple services and the communication between them.

---

## 2. Database-per-Service Architecture

Once you've chosen microservices, a second, genuinely confusing decision follows: should each microservice have its **own** database, or should they all share a **single, common** database?

### Option A: A Shared Database

```
Auth Service          \
Notification Service   } ---> single shared Postgres database
File Upload Service    /
```

Here, each service might use its own table within the same database (a `users` table for Auth, an `uploads` table for File Upload with a `created_by` foreign key pointing back to `users`, a `notifications` table, and so on). Since it's a common database, the Notification Service could, for convenience, directly read the `users` table to get a user's name or email for sending a notification.

**The problem:** this is not true isolation. Even though your code is split into separate services, your database is still one shared, connected resource. Nothing technically stops the Notification Service from accidentally deleting a user, or a bug in the File Service from corrupting the `users` table. The database itself becomes a bottleneck and a hidden coupling point between services that are supposed to be independent.

### Option B: Database-per-Service

> **Definition:** In a database-per-service architecture, every microservice has its own dedicated database, completely separate from every other service's database. Each service is free to choose whichever database technology best fits its own specific needs.

```
Auth Service          --> its own MongoDB database
Notification Service  --> its own ClickHouse database (good for analytics: open rates, click tracking)
Post Service          --> its own Postgres database
```

> **Example of flexibility:** you might choose MongoDB for the Auth service if its schema needs to be flexible, ClickHouse for the Notification service if you're tracking a lot of analytics (open rates, click-through rates), and Postgres for the Post service if you need strong relational guarantees. Each choice is independent and based purely on that service's own needs.

**The trade-off:** since data now lives in genuinely separate databases, you lose the ability to do a database-level `JOIN` across services. If the Post Service needs a user's details alongside their posts, it cannot just join against a `users` table anymore, since that table lives in a completely different database. Instead, the Post Service has to make a request to the Authentication Service to fetch that user's details, and then combine ("join") the data itself, at the **application level**, rather than at the database level.

> **Benefit:** despite this trade-off, database-per-service is considered truly scalable, since every service's database can be independently optimized (different indexes, different scaling strategies, different database technology entirely) without affecting any other service.

---

## 3. Circuit Breaker Pattern

In a microservice architecture, services need to talk to each other to be useful at all. But this interdependency introduces a serious risk.

> **Example of the problem:** Imagine four services, where Service A depends on data from Service B, and other services further down the chain also depend on each other. If Service B suddenly goes down, Service A can't get the data it needs, so Service A can crash too. Since other services depend on Service A, they crash next, and so on. This chain reaction is called a **cascading failure**: one service going down ends up taking down the entire system, purely because other services were depending on it, directly or indirectly.

> **Definition:** The circuit breaker pattern helps handle faults that might take varying amounts of time to recover from, when a service depends on a remote service or resource. It works by placing a proxy layer between a service and the remote service it depends on. This proxy (the circuit breaker) temporarily blocks access to a failing service after detecting repeated failures, preventing repeated, doomed attempts, and giving the failing service room to recover.

**The analogy comes from electrical circuits:** a circuit can be either closed (electricity/data flows through) or open (the connection is broken, and nothing flows through).

```
Service A ----> Circuit Breaker (Proxy) ----> Service B

If Service B is healthy:
   Circuit Breaker stays CLOSED -> requests flow through normally

If Service B starts failing repeatedly:
   Circuit Breaker switches to OPEN -> requests are blocked immediately,
   and a fallback/error response is returned right away, without even
   trying to reach the failing Service B

After a set interval:
   Circuit Breaker moves to HALF-OPEN -> it cautiously allows a
   small number of requests through, to check if Service B has recovered

If Service B responds successfully:
   Circuit Breaker goes back to CLOSED -> normal traffic resumes

If Service B still fails:
   Circuit Breaker goes back to OPEN -> and waits before trying again
```

> **The three states of a circuit breaker:** **Closed** (requests flow through normally), **Open** (requests are blocked immediately, since the dependent service is known to be failing), and **Half-Open** (a cautious, periodic check to see if the dependent service has recovered).

**Why this matters:** without a circuit breaker, a failing service would keep getting hammered with repeated requests from every dependent service, making its recovery even harder. With a circuit breaker in place, once the circuit is "broken" (open), no more requests are sent to the failing service until it's had a chance to recover, at which point traffic is gradually restored.

> **Note from the video:** the creator mentions a dedicated, in-depth video specifically about the circuit breaker pattern, including a full implementation, was in the works at the time of this video.

---

## 4. Event Sourcing (Brief Recap)

> For the full, in-depth explanation with worked examples, see [EventSourcing.md](EventSourcing.md).

In high-throughput systems (like a banking system processing many transactions, or an e-commerce platform processing many orders), constantly mutating a single "current state" field (for example, an `orders` table where a `status` column keeps getting overwritten: `placed` → `out for delivery` → `shipped` → `delivered`) requires a lot of locking, and becomes a bottleneck at scale.

> **Definition:** Event sourcing solves this by using an immutable, append-only event log as the source of truth, instead of a single mutable state. Every meaningful change is recorded as a new, timestamped event, appended to the log, and never removed or overwritten.

```
12:00  OrderPlaced
12:10  OrderReadyToBeShipped
12:30  OrderShipping
...
```

To answer "what's the current status of my order?", the system reads through this event log and returns whatever the most recent event says. This is also exactly how many banking systems track balances: instead of storing and directly mutating a single balance number, they maintain a list of transactions (deposits and withdrawals), and compute the current balance on demand by replaying that list. This avoids heavy locking and produces a more consistent, scalable system, at the cost of needing to reconstruct state by replaying events rather than reading it directly.

---

## 5. CQRS (Brief Recap)

> For the full, in-depth explanation with an AWS-style architecture diagram, see [CQRS.md](CQRS.md).

Once you have an event stream in place (from event sourcing), you can take this further by also optimizing how **writes** and **reads** are handled, since at large scale (like Amazon), funneling every single read and write through one database becomes a bottleneck.

> **Definition:** CQRS (Command Query Responsibility Segregation) splits a system into two separate sides: a **command** side, responsible for mutations (create, update, delete), and a **query** side, responsible only for reads.

```
User wants to mutate something --> Command side --> generates an event
                                                          |
                                                          v
                                          event is used to update a separate,
                                          denormalized Read database
User wants to read something   --> Query side --> reads directly from
                                                     the Read database
```

Since the read side is a separate, pre-optimized (denormalized) database, reads become fast and don't compete with write traffic for the same resources. **The main trade-off:** this architecture works on **eventual consistency**, meaning there's a small delay between a write happening and it being reflected on the read side. If your system can tolerate that delay, CQRS is a strong pattern to use.

---

## Summary: Patterns Covered in This Video

| Pattern | Core Idea | Main Trade-off |
|---|---|---|
| Microservices vs. Monolith | Split an app into independent services vs. keeping it as one unified codebase | Microservices isolate failures and scale independently, but add coordination complexity |
| Database-per-Service | Give each microservice its own dedicated database | True scalability and flexibility, but joins must happen at the application level instead of the database level |
| Circuit Breaker | A proxy layer that blocks requests to a failing dependency, to prevent cascading failures | Adds a layer of indirection and monitoring logic between services |
| Event Sourcing | Use an immutable, append-only event log as the source of truth instead of mutable state | Avoids locking bottlenecks, but current state must be reconstructed by replaying events |
| CQRS | Separate the write (command) path from the read (query) path entirely | Enables independent scaling of reads and writes, at the cost of eventual consistency |

**Key takeaway:** these patterns aren't independent choices made in isolation, they build on each other. Choosing microservices leads directly to the database-per-service question. Depending on services on each other introduces the need for a circuit breaker. And once you're operating at high enough throughput, event sourcing and CQRS become natural next steps for keeping both writes and reads scalable and consistent.
