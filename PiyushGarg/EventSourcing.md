# System Design - Event Sourcing

Source video: [System Design - Event Sourcing](https://www.youtube.com/watch?v=JTmgi0vO5Ug) (Hindi, about 33 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow. Event sourcing is a pattern used by large companies such as Uber and Netflix, and it behaves quite differently from the traditional CRUD approach most developers learn first.

---

## 1. A Quick Recap: What Is an Event?

In most full-stack applications, we are used to **CRUD** operations: Create, Read, Update, and Delete. You create a row in a database, you read it, you update it, or you delete it.

Event sourcing takes a fundamentally different approach. To understand it, we first need to understand what an **event** actually is.

> **Definition:** An event is simply a record of something that happened in the system, usually triggered by a user action.

> **Example:** On a platform like Amazon, examples of events include "an item was added to the cart," "an item was checked out," "a seller uploaded a new product," or "a seller updated a product's price." Every one of these user actions is an event.

Large systems process huge numbers of these events every second. The question event sourcing answers is: how should we actually store and process them?

---

## 2. The Traditional Approach: Updating the Database Directly

Traditionally, whenever an event happens, we perform a direct action on the database. Let's walk through an example: a seller updating a product's price.

```
User
  |
  |  sends a PATCH request: "update product ID 1's price to 100"
  v
API Gateway
  |
  v
Reverse Proxy (for example, NGINX)
  |
  v
Server (EC2 instance)
  |
  v
Database (for example, Postgres)
  |
  |  finds the row for product ID 1 and updates its price column to 100
  v
Done. The next time anyone reads product ID 1, they will see the price as 100.
```

This works perfectly well at a small scale. Whenever someone reads that product afterward, they get the updated price. But problems start to appear once the system scales up.

---

## 3. Problem 1: The Database Becomes a Bottleneck

As the number of updates and reads on the database grows, the database itself becomes a **bottleneck**. Two specific issues show up:

**Locking during frequent updates.** To update a row safely, the database has to acquire a lock on it. If updates happen very frequently, rows get locked often, and this can create race conditions.

> **Example:** Suppose the price of a product is being updated to 100 at almost the exact same moment that another user tries to read that product's price. If the read happens a few milliseconds before the update is fully committed, that user might see the old price (say, 90) instead of the new one (100), even though the update was technically "just" applied. If updates are extremely frequent, the constant row-locking required to keep data consistent can seriously slow the system down, because nobody else can read a row while it is locked for an update.

---

## 4. Problem 2: State Management Is Genuinely Hard

The second, deeper problem is **state management**. Let's understand this with a video processing example, similar to how YouTube handles uploaded videos.

Processing a video (encoding it into multiple resolutions like 360p, 480p, 720p, 1080p, and 4K, transcribing it, and optimizing it for low-end devices) takes real time, often many minutes, depending on available resources. To build a video processing pipeline, we need to carefully track the state of each video as it moves through the system.

**A typical (traditional) design might look like this:**

```
Step 1: User uploads a raw video (stored in S3)
Step 2: A database entry is created for this video, with state = "uploaded"
Step 3: A free worker (from a worker pool) picks up the video
         -> state is updated to "processing"
Step 4: Processing either succeeds or fails
         -> on success, state is updated to "success"
         -> on failure, state is updated to "failed"
```

The pseudocode for this looks simple enough:

```
When user uploads video:      update DB status -> "uploaded"
When worker picks up video:   update DB status -> "processing"
When worker finishes:
   if successful:              update DB status -> "success"
   if failed:                  update DB status -> "failed"
```

**Here is where it breaks down.** Each of these steps is a separate database update, and each one can independently fail.

> **Example of what can go wrong:** Suppose the video is actually uploaded successfully, but at that exact moment the database is too busy to accept the "uploaded" status update, and that particular database write fails. The real-world state says "uploaded," but the database still shows nothing, or an earlier, stale state. Later, a worker picks up the video and successfully updates the state to "processing." But then the final "success" update fails for the same reason (a busy database, a bad query, or some other glitch). Now the video has actually finished processing successfully in the real world, but the database will show it stuck at "processing" forever. This is exactly the kind of situation where a user uploads a video and it appears permanently "stuck" in an earlier state, even though the real work behind the scenes actually completed fine.

**Why this happens:** the real world and the database can silently fall out of sync, because each state update is an independent operation that might fail on its own, and there is no way to trace back afterward exactly what really happened and where things went wrong. If a user complains, "I uploaded my video, why does it still say processing?", there is no record to investigate. In other words, there are no real **events** being tracked here, only a single, constantly overwritten status field.

---

## 5. Event Sourcing: Making Events the Source of Truth

> **Definition:** Event sourcing is a software design pattern where every change to an application's state is stored as a sequence of events, instead of just storing the current state. Each state-changing action is recorded as an event in an append-only log. This makes it possible to reconstruct the application's state at any point in time, simply by replaying those events in order.

The key difference from the traditional approach is this: **we never directly update the database.** A direct update is an atomic operation that simply overwrites whatever was there before, permanently losing the history of how that state was reached. Event sourcing instead says: always append a new **event** to an **event log**.

> **Definition:** An event log is an append-only log, meaning new events are always added to the end of it, never inserted in the middle or reordered. This matters because events must always stay in the exact order they actually happened. For instance, "video uploaded" must always come before "processing started," and this order can never be shuffled.

**The video processing example, redone with event sourcing:**

```
Event Log for one video (append-only, always in order):

1. Event: "VideoUploaded"          data: { path: "...", id: "..." }   timestamp: T1
2. Event: "VideoProcessingInit"    data: { ... }                     timestamp: T2
3. Event: "VideoProcessingSuccess" data: { ... }                     timestamp: T3
   (or "VideoProcessingFailed" if something went wrong)
```

Every event also carries a timestamp, recording exactly when it was emitted. These events are stored permanently in some storage layer (for example, S3), and this stored, ordered sequence of events becomes the actual source of truth, not a single "status" column in a database.

---

## 6. Hydration: Reconstructing State From Events

> **Definition:** Hydration is the process of reconstructing an object's current state by reading and replaying its events in order, from the very first one to the most recent one.

> **Example:** If a user asks, "What is the current status of the video I uploaded?", the system does not look up a status column. Instead, it replays that video's events in order: first "VideoUploaded" (so the state was "uploaded"), then "VideoProcessingInit" (so the state moved to "processing"), then "VideoProcessingSuccess" (so the final state is "success"). The system then returns "success" to the user. If the user wants to know exactly how it reached that state, the full event history can be shown too: uploaded at this time, picked up by a worker at that time, and completed successfully after that.

### A Second Example: Bank Account Balances

Consider a banking system that needs to track a user's current balance.

**The traditional way:** keep a single "balance" column on the user's row, and mutate it directly on every transaction.

```
Initial balance: 500
Deposit 200  ->  balance becomes 700
Deposit 300  ->  balance becomes 1000
Withdraw 500 ->  balance becomes 500
```

Here, the system is trying to maintain state itself, through a series of atomic overwrite operations, and each previous value is lost the moment it changes.

**The event-sourced way:** never touch a "balance" column directly. Instead, append events describing exactly what happened.

```
Event Log (append-only):

1. Event: "Deposit"   data: { amount: 200 }
2. Event: "Deposit"   data: { amount: 300 }
3. Event: "Withdraw"  data: { amount: 500 }
```

When the current balance needs to be calculated, hydration replays these events on top of the known initial balance:

```
Start: initial balance = 500
Apply Event 1 (Deposit 200):   500 + 200  = 700
Apply Event 2 (Deposit 300):   700 + 300  = 1000
Apply Event 3 (Withdraw 500):  1000 - 500 = 500

Final reconstructed balance: 500
```

The final balance is calculated purely by replaying every event, and the system can also explain to the user exactly how that final number was reached, by showing them the full sequence of events.

---

## 7. Why Event Sourcing Is Powerful: Audit Trails, Reconciliation, and Time Travel

Because every single event is preserved, event sourcing gives you several very useful properties almost for free:

- **Audit trail.** If a user disputes a result (for example, "My balance should be 300, not 500, something is wrong"), you have a complete, permanent history of every event that led to the current state. There is nothing hidden or overwritten.
- **Reconciliation.** If something does go wrong (for example, a bug caused an incorrect state), you can simply replay all the stored events again from scratch to recompute the correct current state. This process is called reconciliation.
- **Time travel.** Since every event is timestamped and preserved, a user can ask, "What was my balance a month ago?" You can answer this by replaying only the events up to that point in time. This would be impossible with a traditional system that only stores the single, current, overwritten value.

> **Key idea:** With event sourcing, the event stream itself is the source of truth, not a single mutable field in a database.

---

## 8. A Practical Problem: Reading Every Event, Every Time, Is Inefficient

You might reasonably think that replaying every single event, every single time someone wants to read the current state, sounds inefficient, and you would be right. As a system runs in production, an append-only log naturally keeps growing forever, since events are only ever added, never removed or rewritten.

**The solution: cache the hydration result.** Instead of replaying the entire event log from scratch on every read, you can maintain a separate, continuously updated store (sometimes called a materialized view, or a "read model") that already holds the most recently computed state.

```
Event Stream (append-only, source of truth)
   |
   |  a background process continuously reads new events in order
   v
Read Model / Cached State Store (for example, Postgres)
   |
   |  always reflects the latest known state
   v
User queries -> read directly from this fast, up-to-date store
```

In this setup, you never update state synchronously in direct response to a user action. Instead, a hydration process continuously runs in the background, processing events in sequence, and keeps this cached state store up to date. Regular user queries can then read quickly from this cached store, instead of replaying the full event log every time. And if a user ever complains that the cached state looks wrong, you can still fall back to reconciliation: replay the actual events again and correct the cached store, while also being able to show the user exactly how that state was reached.

> **Note:** This pattern, keeping a separate read-optimized store built from a stream of write events, is closely related to a pattern called **CQRS (Command Query Responsibility Segregation)**, which separates the logic that writes/changes data (commands) from the logic that reads data (queries). The video mentions a dedicated future video specifically about CQRS.

---

## 9. A Second Practical Problem: Keeping Events In Order Across Multiple Workers

Suppose you have an event stream, and to process it at scale, you spin up multiple worker processes (for example, three separate Node.js servers) that all consume from the same stream.

**Here is a subtle problem that can occur:**

```
Event Stream for one video (in true order):
   1. VideoUploaded
   2. VideoProcessingInit
   3. VideoProcessingSuccess

If these three events get picked up by three different, independently busy workers:
   Worker A picks up Event 1, but is slow / under heavy load
   Worker B picks up Event 2 quickly -> updates state to "processing"
   Worker C picks up Event 3 quickly -> updates state to "success"
   Worker A finally finishes late -> updates state back to "uploaded"

Result: the final recorded state is "uploaded", even though the real
sequence of events said the video succeeded. The state is now wrong,
purely because of the order in which workers happened to finish.
```

Even though the underlying events were correct, processing them out of order across independent workers corrupted the final state. This is exactly the kind of bug that would make a user complain that their video's status looks wrong. You could still fix it afterward with reconciliation, but it is much better to avoid the problem entirely.

**The solution: consumer groups and partitions (as used in Apache Kafka).**

> **Definition:** A partition is essentially an index or a bucket used to divide a stream of events, so that all events belonging to the same entity are guaranteed to always go to the same partition, and therefore always get processed by the same worker, strictly in order.

```
Video A's events -> always sent to Partition 0 -> always processed by Worker 1
Video B's events -> always sent to Partition 1 -> always processed by Worker 2
```

By guaranteeing that every event for a specific object (for example, one specific video, or one specific bank account) always lands on the same partition, and is therefore always handled by the same worker, you guarantee that events for that object are always processed strictly in the order they actually happened, even if that worker is sometimes slow. The order stays correct; only the timing of when it finishes might be delayed.

A **consumer group** is a set of worker processes that split the full set of partitions between themselves.

```
Consumer Group (3 worker processes)
   |--> Worker 1 subscribed to Partition 1 and Partition 4
   |--> Worker 2 subscribed to Partition 2 and Partition 5
   |--> Worker 3 subscribed to Partition 3
```

Since a specific object's events always route to the same partition, and that partition is always handled by the same worker within the group, correct ordering per object is guaranteed system-wide, even while many different objects are being processed in parallel across multiple workers.

> **Note:** This concept of consumer groups and topic partitions comes from **Apache Kafka**, a tool very commonly used to implement event sourcing at scale. The video references a separate, dedicated video on Kafka for more depth.

---

## 10. Real-World Example: Orders on Amazon as a Sequence of Events

The next time you order something on Amazon, remember that everything happening behind the scenes is a sequence of events, not a single overwritten status field.

```
Event Log for one order (append-only, in true order):

1. ItemAddedToCart
2. PaymentMethodAdded
3. OrderPlaced
4. OrderDispatched
5. OrderShipped
6. OrderDelivered   (or OrderReturned, if the customer returns it)
```

The system never simply overwrites a single "status" field with "Delivered." It stores this entire sequence of events. Whenever you check your order status, the system replays these events and finds that the most recent one was "OrderDelivered," so it reports your order as successfully delivered. If you ever complain, "My order was never delivered, why does it say delivered?", the full event history can be examined to find out exactly where something might have gone wrong, rather than just trusting one single, possibly incorrectly updated, field.

---

## Summary: Key Concepts in Event Sourcing

| Concept | Purpose |
|---|---|
| Event | A record of something that happened (for example, "item added to cart") |
| Traditional (CRUD) Update | Directly overwrites the current state in the database, losing prior history |
| Event Log | An append-only, ordered list of events that acts as the true source of truth |
| Hydration | Reconstructing current state by replaying events in order |
| Reconciliation | Replaying events again to fix or verify a state that seems incorrect |
| Audit Trail | The full, permanent history of events, useful for debugging and disputes |
| Time Travel | Reconstructing what the state looked like at any past point in time |
| Read Model / Cached State | A continuously updated store built from events, so reads don't need to replay everything every time |
| CQRS | A related pattern that separates the write (command) path from the read (query) path |
| Partitions and Consumer Groups | A mechanism (from Kafka) that guarantees all events for one object are processed in order, by the same worker |

**What comes next, per the video:** a dedicated video on **CQRS (Command Query Responsibility Segregation)**, which is closely related to event sourcing.
