# Master Queues | System Design Interview

Source video: [Master Queues | System Design Interview](https://www.youtube.com/watch?v=2tCfITBVKjA) (Hindi, about 31 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow. There's a common misconception in system design that queues always guarantee First In, First Out (FIFO) ordering. This is not actually true, and this video works through why, alongside the deeper mechanics of how queues really work.

---

## 1. Starting Point: A Fully Synchronous Sign-Up Flow

Consider a very simple sign-up flow.

```
User sends: POST /signup
   |
   v
Server:
   1. Validates the request body (email, password)
   2. Inserts the new user into the database (e.g. Postgres)
   3. Sends a welcome email (e.g. via AWS SES)
   4. Creates an auth token
   5. Returns the response
```

This is called a **synchronous architecture**, because every step happens one after another, in sequence, before the response is finally returned to the user.

### Why This Breaks Down at Scale

This works fine for a handful of users, but problems appear once traffic grows (say, to 10,000 or 100,000 sign-ups).

**Problem 1: External dependency.** Sending an email always involves calling an external service. If that email service is slow or temporarily down, the entire sign-up process slows down along with it, even though sending a welcome email has nothing fundamentally to do with whether an account was created successfully.

**Problem 2: Rate limiting.** Email services typically enforce a rate limit (say, 25 emails per minute). If 10,000 users try to sign up around the same time, and your code tries to synchronously send 10,000 emails in that same window, the rate limit will be hit almost immediately. This causes the "send welcome email" step to start failing, which can crash the entire signup route.

> **The core issue:** a fundamentally non-critical, external step (sending a welcome email) is blocking a critical, core step (creating a user account), and introducing fragility that has nothing to do with the actual sign-up logic itself.

---

## 2. Moving to an Asynchronous Architecture

The fix is to identify which steps are truly critical (must happen synchronously) and which steps can be deferred (processed asynchronously, later).

- **Inserting the user into the database** is critical. The entire point of "signing up" is creating that database record, so this step must remain synchronous.
- **Sending the welcome email** is not critical to the sign-up itself. It's fine if it happens a little later.

```
Server:
   1. Insert user into DB           (synchronous, critical)
   2. Enqueue a "send welcome email" message onto a message queue
      (NOT sending the email directly here)
   3. Create an auth token           (synchronous)
   4. Return the response
```

Notice the key change: the server no longer sends the email itself. It simply places ("enqueues") a message describing that task onto a **message queue**, and moves on immediately.

> **Definition:** A queue, in this context, works like a real-world line (think of a line at a ticket counter or a temple). You place items ("messages" or "payloads") into it, and they wait there until something picks them up and processes them.

```
Message Queue:  [ msg1 ]  [ msg2 ]  [ msg3 ]  ...
```

A queue always has two ends:

- **Producer side:** whoever puts ("produces") messages into the queue. This side is always simple, you just need a way to enqueue data.
- **Consumer side:** whoever picks up ("consumes") messages from the queue and actually processes them (for example, a separate worker/server that talks to AWS SES to send the actual email).

```
Producer -----> [ Message Queue ] -----> Consumer (Worker)
(server)                                  (processes and discards each message)
```

### Why This Is Called "Asynchronous"

Since the email is no longer sent immediately, a user might get their welcome email instantly, or it might take a few minutes, depending on how many messages are already waiting ahead of it in the queue. This delay is completely acceptable, since sending a welcome email a few minutes late causes no real harm.

> **Example:** When you upload a video to YouTube, it doesn't become available instantly. YouTube's servers pick up the upload asynchronously and process it, often taking 10 to 15 minutes, before it finally becomes available to viewers. This is exactly the same asynchronous processing pattern.

Everything covered so far is the **High-Level Design (HLD)**: a bird's-eye view of what a queue is and why it's useful. Queues are actually much more nuanced underneath, which is where the **Low-Level Design (LLD)** comes in.

---

## 3. Two Fundamental Queue Mechanisms: Push vs. Pull

When it comes to how consumers actually receive messages from a queue, there are two broad mechanisms: **push-based** and **pull-based**.

> **A general rule to remember:** the producer side is always easy in either mechanism, you just need an API to enqueue data. The real differences between push and pull show up entirely on the **consumer** side.

---

## 4. Push-Based Queues (e.g. RabbitMQ)

> **Definition:** In a push-based queue, worker functions register themselves with a central broker, announcing that they're available to process messages. The broker itself is then responsible for deciding which worker gets which message, and it actively pushes the message to that worker.

**How it works, step by step (using RabbitMQ as an example):**

```
Producer --> calls RabbitMQ's API (e.g. "/send") with the message data
   |
   v
RabbitMQ inserts the message into its internally managed queue
   and sends an acknowledgement back to the producer
```

```
Worker function
   |
   |  registers itself with RabbitMQ: "I'm available, send me work"
   v
RabbitMQ stores this registration (like a factory owner noting
   that a particular laborer is free and ready for the next task)
   |
   |  when a message arrives, RabbitMQ decides which registered
   |  worker should receive it, and pushes it directly to them
   v
Worker receives and processes the message
```

Multiple workers can register themselves the same way, and RabbitMQ automatically load-balances incoming messages across all of them.

**Heartbeat mechanism:** since RabbitMQ needs to know which workers are actually still alive, every registered worker periodically sends a heartbeat (for example, every 30 seconds) saying "I'm still alive." If a worker's heartbeat doesn't arrive within the expected window (say, within a minute), RabbitMQ declares it dead and stops sending it new messages.

### Pros of Push-Based Queues

- **Simple architecture on the consumer side.** You don't need to write any code for how to fetch messages from the queue; you only need to write the business logic for what to do once a message arrives.
- **Deduplication is built in.** Since a single, central broker is responsible for assigning each message to exactly one worker, there's no risk of the same message accidentally being processed twice by two different workers.
- **Retry mechanisms are built in.** If a worker doesn't acknowledge that it successfully processed a message within a certain time, RabbitMQ can automatically give that same message to a different worker instead.

### Cons of Push-Based Queues

- **The single broker can become a bottleneck.** Since the broker is responsible for managing every registration, every heartbeat, and every message assignment, this centralized control can make push-based queues behave somewhat slower under very heavy load.

---

## 5. Pull-Based Queues (e.g. AWS SQS)

> **Definition:** In a pull-based queue, there is no central broker actively assigning work. Instead, the queue simply exposes two APIs: one to push (enqueue) messages, and one to pull (fetch) messages. Consumers are entirely responsible for repeatedly asking the queue whether new messages are available.

```
Queue (e.g. AWS SQS)
   |--> Push API: producer calls this to enqueue a message
   |--> Pull API: consumer calls this to fetch a message, if one is available
```

Since there's no worker registration and no broker actively pushing anything, consumers must implement **polling**: repeatedly calling the pull API on some interval to check for new messages.

```
setInterval(() => {
    call("/pull")   // ask the queue: "do you have any data for me?"
}, 60000)   // e.g., every 1 minute
```

Each worker independently polls the queue, processes whatever it finds, and repeats.

### Pros of Pull-Based Queues

- **You have full control.** You decide exactly when to poll, how frequently, and how to handle every part of the process.

### Cons of Pull-Based Queues (Things You Must Handle Yourself)

- **Deduplication is your responsibility.** If multiple workers happen to poll at nearly the same moment and all see the same available message, more than one of them might end up processing it. A common fix is to use something like a Redis-based lock: a worker acquires a lock on a specific message ID before processing it, so other workers skip it if they see it's already locked.
- **Retry mechanisms are your responsibility.** If a worker pulls a message but fails to process it, it's up to that worker's code to put the message back into the queue for a retry, or route it to a separate **dead letter queue** for failed messages.
- **Backoff strategy is your responsibility.** If the queue has been empty for a while, continuously polling every minute wastes API calls (and pull-based services like SQS charge per API call). A good approach is to implement a backoff: if you've polled, say, 20 times with nothing found, increase your polling interval (from 1 minute to 5 minutes, then to 10 minutes, and so on), and reset it back down once messages start arriving again.

> **Personal preference shared in the video:** the creator generally prefers pull-based queues, specifically tools like **BullMQ** and **AWS SQS**. BullMQ (built on top of Redis) provides a nice abstraction that handles a lot of the polling internally, so you don't have to worry about it as much yourself. AWS SQS, on the other hand, is much more "raw": you're expected to implement everything yourself (an endless polling loop, backoff strategies, retry mechanisms, rate limiting, and so on) which the video's creator describes as "a fun and genuinely interesting engineering problem to solve."

---

## 6. The Big Myth: Are Queues Always FIFO?

You might assume: of course a queue processes messages First In, First Out. If message 1 was enqueued before message 2, surely message 1 is always processed first.

**This is not guaranteed once you have multiple workers processing in parallel.**

> **Example:** Imagine you're building an SMS service, where message order genuinely matters. You want a user to first receive a welcome message, then a greeting message a bit later, then a follow-up asking about their experience a few days after that. Now suppose these messages (1, 2, 3, 4, 5, 6...) are enqueued in the correct order, but you have multiple parallel workers pulling and processing them. If one worker happens to be faster than another, it might finish processing message 3 before a different, slower worker finishes message 1. The result: message 3 might get delivered to the user before message 1 does, completely scrambling the intended order. The user ends up receiving a feedback request before their welcome message, which is a confusing, broken experience.

> **Key takeaway:** enqueueing messages happens in sequence, but **processing** them across multiple parallel workers does not guarantee that same sequence. Queues do not inherently guarantee FIFO delivery in a system with multiple concurrent consumers.

---

## 7. How Kafka Solves the Ordering Problem

This ordering problem is exactly what **Apache Kafka** was designed to solve.

> **Definition:** Kafka uses **partitions**, and each message can be assigned a **key**. All messages sharing the same key are guaranteed to always be routed to the same partition, and are therefore always processed in the correct order relative to each other, since they end up being handled by the same consumer.

> **Example:** Suppose User 1's messages and User 2's messages are interleaved in the incoming stream. By assigning each user's messages a key based on their user ID, Kafka guarantees that all of User 1's messages always go to the same partition (and are processed in order), while all of User 2's messages always go to a different, consistent partition (also processed in order). Different users' messages can still be processed in parallel across different partitions, but within any single user's sequence, order is always preserved.

This gives you the best of both worlds: **parallelism** across different keys/entities, while still guaranteeing **strict ordering** within any single key/entity's own sequence of messages. This specific guarantee (ordering per key, combined with parallel processing across keys) is something Kafka handles especially well, better than most simpler queue systems. Kafka also has a related concept called **consumer groups**, which is worth studying separately in more depth (the video references a dedicated Kafka video for this).

---

## Summary: Push vs. Pull, and the FIFO Myth

| Concept | Push-Based Queues (e.g. RabbitMQ) | Pull-Based Queues (e.g. AWS SQS, BullMQ) |
|---|---|---|
| Who decides message delivery? | The central broker decides and pushes to registered workers | The consumer repeatedly polls to check for new messages |
| Consumer-side complexity | Simple; just write business logic | You must write polling logic yourself |
| Deduplication | Built in (handled by the single broker) | You must implement it yourself (e.g. via distributed locks) |
| Retry mechanism | Built in (broker reassigns unacknowledged messages) | You must implement it yourself |
| Backoff strategy | Not needed (broker pushes only when work exists) | You must implement it yourself, to avoid wasting API calls |
| Control | Less control (broker manages everything) | Full control over timing and behavior |
| Main drawback | The single broker can become a bottleneck | Requires significantly more custom code |

**On the FIFO myth:** queues do not automatically guarantee First In, First Out processing once multiple parallel workers are involved, since faster workers can finish later messages before slower workers finish earlier ones. Systems like Kafka solve this with partitions and keys, guaranteeing strict order within each key while still processing different keys in parallel.

**Final takeaway:** there's no universally "better" choice between push-based and pull-based queues, it depends entirely on your system's specific requirements, and there's nothing stopping you from using both patterns in the same application (for example, push-based for real-time notifications, and pull-based for a heavier video-processing pipeline). System design is fundamentally about understanding these trade-offs and picking the right tool for each specific job.
