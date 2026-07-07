# CQRS System Design Pattern

Source video: [CQRS System Design Pattern](https://www.youtube.com/watch?v=vNplj9LwQSw) (Hindi, about 33 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow. This video builds directly on [EventSourcing.md](EventSourcing.md), so it helps to read that one first.

> **A disclaimer from the video, worth repeating up front:** CQRS is used only in very complex, large-scale systems. It does not mean every basic application needs this pattern. Most small or medium applications should not implement CQRS. It is still useful to understand the intention behind it, so you recognize when it is genuinely the right tool.

---

## 1. How a Traditional Application Works

Before looking at CQRS, it helps to see how a typical application is usually structured.

```
User (Client)
   |
   v
Server Layer
   |--> REST API Endpoints (built with, for example, Django, FastAPI, or Express.js)
   |--> Controller Layer (decides what to do based on the HTTP method)
   |--> Database (for example, Postgres)
```

A user's requests, whether GET, POST, or PATCH, all hit these REST API endpoints. Based on the type of request, the appropriate controller is called:

- A **GET** request triggers a controller that performs a **read** operation on the database and returns the data.
- A **PATCH** request (an update) triggers a different controller that performs an **update** operation on the database.
- A **POST** request (a create) triggers yet another controller that performs an **insert** (create) operation on the database.

When this kind of application needs to scale, we typically scale the server layer itself, using the two familiar approaches: **vertical scaling** (adding more CPU or RAM to the same server) and **horizontal scaling** (adding more copies of the server).

---

## 2. The Core Problem: One Database Handling Everything

The basic operations performed on any piece of data are called **CRUD**: Create, Read, Update, and Delete. In a traditional application, all of these operations, reads and writes together, hit the very same database.

As traffic grows, this single database becomes a **bottleneck**. The reason is that reads and writes can conflict with each other.

> **Example:** Imagine a pair of earphones on Amazon, currently priced at $50. At any given moment, a huge number of users might be reading that price at once. At the very same time, a seller might trigger an update to change the price to $40 for a sale. That update operation needs to acquire a lock on that row while it makes the change. If updates like this happen frequently, and a huge number of reads are also happening at the same time, the database ends up spending a lot of effort managing locks and contention. At the scale of a company like Amazon, this makes queries noticeably slower and turns the single database into a serious bottleneck.

This exact problem is what the **CQRS** pattern is designed to solve.

---

## 3. What Is CQRS?

> **Definition:** CQRS stands for Command Query Responsibility Segregation. It is a design pattern that separates the part of a system responsible for changing data (called the "command" side) from the part responsible for reading data (called the "query" side). You can use CQRS to separate updates and queries whenever they have different requirements for throughput, latency, or consistency.

- **Throughput** here refers to how frequently data is being read or changed, in other words, the number of input/output operations happening.
- **Command** refers to any operation that mutates data: creating, updating, or deleting it.
- **Query** refers only to reading data. Nothing else.

CQRS essentially splits CRUD into two separate halves:

```
CRUD
  |--> Command side: Create, Update, Delete (anything that mutates data)
  |--> Query side:   Read (anything that only reads data)
```

---

## 4. Routing Requests: Command Side vs Query Side

The first step in implementing CQRS is routing incoming requests to the correct side, based on their HTTP method. This is typically done at the **API Gateway** level, which acts as the single entry point into the system.

```
User
  |
  v
API Gateway
  |--> GET requests                          --> Query side
  |--> POST / PUT / PATCH / DELETE requests  --> Command side
```

All users interact only with the API Gateway, and the gateway automatically routes each request to the correct side, purely based on its HTTP method.

---

## 5. Splitting the Database Too: Write DB and Read DB

The next, more interesting step is that CQRS doesn't just split the application logic. It also splits the **database** itself into two separate databases: a **write database** and a **read database**.

```
Command side  --> Write Database  (optimized purely for mutations)
Query side    --> Read Database   (optimized purely for reads)
```

**A natural question: how do these two databases stay in sync with each other?**

Keeping two separate databases in sync is a real challenge. Whenever a change happens in the write database, that change needs to eventually be reflected in the read database. This is typically done using an **event system**: whenever a write happens, an event is emitted (for example, onto a queue), and a separate process consumes that event and updates the read database accordingly.

```
Command side
   |
   v
Write Database
   |
   |  a write triggers an event, for example "ProductUpdated"
   v
Queue / Event System
   |
   v
Read Database gets updated based on that event
```

**This introduces eventual consistency.** Since the read database is updated asynchronously, based on events, there can be a short delay (say, one or two seconds) before a change made on the write side becomes visible on the read side. This is called **eventual consistency**, meaning the two databases will match eventually, just not necessarily instantly.

> **The trade-off to understand:** At large scale, you generally have two choices. Either you keep everything in one tightly consistent database and risk it becoming overwhelmed and crashing under heavy load, or you accept a small amount of eventual consistency (a short delay between write and read) in exchange for a much more scalable, fault-tolerant architecture. For most read-heavy systems, this trade-off is well worth it.

---

## 6. The Full CQRS Architecture, Piece by Piece

Let's build up the complete picture, layer by layer.

```
                     Presentation Layer
                     (the UI / REST API endpoints shown to the user)
                              |
             +----------------+----------------+
             |                                 |
             v                                 v
        Command Side                      Query Side
```

### The Command Side

On the command side, a very important rule applies: **you never directly mutate the database.** Instead, every single action generates a **command** first.

> **Definition:** A command is a structured representation of an intended change, for example "update this product," carrying whatever data is needed to perform that change (such as the product's ID and the new values).

```
Presentation Layer
   |
   |  user wants to update a product
   v
Command Handler
   |
   |  generates a command, e.g.:
   |  { type: "ProductUpdateCommand", productId: "...", payload: {...} }
   v
Write Model
   |
   |  interprets the command and executes the appropriate mutation
   v
Write Database
```

The **write model** is the piece of logic responsible for looking at a command, figuring out what it means, and mutating the write database accordingly (updating a row for an update command, inserting a row for a create command, and so on). Since this write model only ever deals with mutations, it can be continuously optimized purely for that purpose.

### The Query Side

```
Presentation Layer
   |
   |  user wants to read some data
   v
Query Handler
   |
   v
Read Model
   |
   v
Read Database
```

The **read model** is optimized purely for reading data quickly, and is completely separate from anything related to mutations.

---

## 7. Choosing Different Database Types for Write vs Read

One major benefit of CQRS is that, because the write database and read database are now separate, you are free to choose a completely different type of database for each one, based on what each side actually needs.

**Write database: normalized, relational data.** For the write side, you might use a SQL database with data kept in **normalized** form: multiple related tables, connected using foreign keys, with minimal data duplication. For example, one table might hold just the product ID and its slug, another table might hold its extended properties, another might hold its orders, and so on.

**Read database: denormalized, pre-joined data.** For the read side, you might instead use a NoSQL database, such as MongoDB, and store data in a **denormalized** form: a single document that already contains everything needed to answer a typical read query, without requiring any joins.

```
Write DB (normalized, SQL):
   products table       -->  id, slug
   product_details table -->  extended properties
   orders table          -->  order history
   sales table           -->  sales counts

Read DB (denormalized, NoSQL):
   One single document per product, already containing:
   {
     productId: "...",
     details: {...},
     salesCount: ...,
     purchaseCount: ...
   }
```

**Why this matters:** if you need to query data spread across multiple normalized tables, you typically need joins and lookups, and these require the database to perform a lot of extra, often recursive, work. On the read side, since the data is already stored in its final, pre-joined shape, a query becomes as simple as a single `WHERE id = ...` clause against one table or collection, with no joins needed at all. This is the main benefit: fast, simple reads, at the cost of some duplicated data.

And if this denormalized read data ever becomes stale or corrupted, that's actually fine, because you can always regenerate it again from the write side.

---

## 8. A Full Example Architecture (in an AWS-Style Context)

This is a faithful recreation of the actual whiteboard diagram drawn in the video, showing how every piece introduced above fits together into one complete system.

```
                                   Client
                                     ^  |
                                     |  v
                                  API Gateway
                       HttpMethod { Get, Post, Put, Patch, Delete }
                                     |
                    +----------------+----------------+
                    |                                 |
        PUT/POST/PATCH/DELETE                        GET
                    |                                 |
                    v                                 v
           CommandServiceELB                   QueryServiceELB
                    |                                 |
         (target group, 4 EC2s)               (target group, 4 EC2s)
          "Authorization, Validation"             "Query Handlers"
                    |                                 |
                    v                                 v
          Event Message Broker                  Query Handler (service)
        (Kafka / AWS Kinesis Streams)                  ^
                    |                                  |  reads from
                    v                                  |
           (4 consumer EC2 instances)                 ReadDB  <----------------------+
                    |                                  ^                             |
                    v                                  |                             |
        Write DB (Storing All logs)  - - - - "Recreate the Read DB" - - - - - - - - -+
           (e.g. ClickHouse, append-only)              ^
                    |                                   \
                    v                                    \  Lambda updates ReadDB
        Event Notification (e.g. SNS)                     \
        "something got updated"                            \
                    |                                        \
        +-----------+-----------+                             \
        |                       |                               \
        v                       v                                \
  UpdateReadDBQueue        EmailQueue                              |
        |                       |                                  |
        v                       v                                  |
      Lambda  --------(updates ReadDB)--------------------------- (arrow up into ReadDB)
                       |
                       v
              Simple Email Service (SES)
                       |
                       |  email delivered
                       v
                    Client (loops back to the user)
```

A few important details from this flow, matching the instructor's own diagram:

- **The command side never has a separate "command handler" box before the event broker.** The four EC2 instances behind `CommandServiceELB` directly perform **authorization and validation**, and then emit the resulting event straight onto the **Event Message Broker** (Kafka or AWS Kinesis Streams). There is no intermediate aggregator service.
- A **second, separate group of consumer EC2 instances** sits behind the broker, consuming these logs and persisting them into the **Write DB**. This write database (the video uses ClickHouse as an example) stores **all logs**, meaning the raw, append-only history of every command, not a single mutable "final state."
- **Kafka's consumer groups and partitions** (covered in the event sourcing notes) are used here too, to guarantee that all events for a particular object are always processed in the correct order, even across multiple consumer instances.
- On the query side, the EC2 instances behind `QueryServiceELB` are the horizontally scaled **Query Handlers**. They call into a separate **Query Handler service** (for example, an ORM layer like Prisma), which is the only thing that actually reads from the **ReadDB**.
- The **ReadDB is updated in two different ways**, and both are visible in the diagram:
  1. **The normal, incremental path:** the Write DB emits a "something got updated" notification (an SNS-style event), which fans out to multiple queues, including an `UpdateReadDBQueue`. A Lambda function consumes that queue and updates the ReadDB directly.
  2. **The full-rebuild path:** a separate, direct, dashed arrow goes straight from the Write DB to the ReadDB, labeled **"Recreate the Read DB."** This represents replaying the entire append-only log from scratch whenever the ReadDB needs to be fully regenerated (for example, if it becomes corrupted, stale, or you want to migrate it to a different database technology entirely).
- The same fan-out also feeds a **second queue** (an email queue), which is consumed by a **Simple Email Service (SES)** style component, for example to send a notification or promotional email back to the user. In the diagram, this is drawn as a long connector looping all the way back up to the client, representing the email actually reaching the user.

---

## 9. Combining CQRS With Event Sourcing

If you've read the [event sourcing notes](EventSourcing.md), you'll notice these two patterns fit together very naturally.

Instead of the command handler directly writing a final mutated value into the write database, it can instead simply **append an event describing what happened** (for example, "ProductUpdated", with the relevant details) into an append-only log, exactly as described in event sourcing. The read database is then built up, or rebuilt, by replaying these logs.

```
Command generated (e.g. "update this product")
   |
   v
Instead of overwriting a final state directly...
   |
   v
...append an event to an append-only log (the real write database)
   |
   v
Replay these logs (hydration) to build or rebuild the Read Database
```

This means that whenever the read database becomes stale, corrupted, or you want to switch to a different database technology entirely, you don't lose anything. You can always iterate over every stored event again and regenerate the read database completely from scratch.

You can also take this further: since every mutation is emitted as an event on a message broker (like Kafka), you are not limited to only updating the read database from that event. You can trigger **other actions entirely** from the very same event.

> **Example:** Suppose a product's price is updated and the new price is lower than before. Besides updating the read database, that same "ProductUpdated" event could also be picked up by a separate email service, which then sends a promotional email to interested users, letting them know the price dropped.

---

## 10. Adding a CDN for Further Optimization

You can optimize the query side even further by placing a CDN (like AWS CloudFront) in front of it.

```
User
  |
  v
CloudFront (CDN)
  |--> Cache hit  --> return cached response directly (fast)
  |--> Cache miss --> forward to Query Service --> Read Database
```

Whenever an update happens on the write side, you can also trigger a **cache invalidation**, for example through a dedicated Lambda function that clears the relevant cached entries in CloudFront. This ensures users get fresh data soon after an update, while still benefiting from caching most of the time.

---

## 11. When Should You Actually Use CQRS?

CQRS adds real complexity to a system, and this complexity is not free. It is not suitable, and not even really designed, for small applications. It becomes genuinely useful in situations like these:

- Your system already follows a **database-per-service** pattern (each microservice owns its own database), and you need to join or combine data across multiple services for reading purposes.
- Your **read** workload and your **write** workload have meaningfully different requirements for scaling, latency, or consistency. For example, reads might need to be extremely fast and heavily cached, while writes need to be strongly validated and consistent.
- **Eventual consistency is acceptable** for your read queries. This is true for many typical applications, but it is generally not acceptable for something like a real-time stock trading system, where reads must always reflect the absolute latest state instantly.

---

## Summary: Key Concepts in CQRS

| Concept | Purpose |
|---|---|
| Command | A structured request to change data (create, update, or delete) |
| Query | A request to only read data, never to change it |
| Command Side | Handles all mutations; can be optimized purely for writes |
| Query Side | Handles all reads; can be optimized purely for reads |
| Write Database | Stores data optimized for accurate, validated mutations (often normalized) |
| Read Database | Stores data optimized for fast reads (often denormalized, pre-joined) |
| Eventual Consistency | The read side may lag slightly behind the write side, in exchange for scalability |
| Event System (Queue / Kafka) | Keeps the read database in sync with the write database asynchronously |
| Event Sourcing + CQRS | Storing commands as append-only event logs, so the read database can always be rebuilt from scratch |
| CDN + Cache Invalidation | Further speeds up reads, while still refreshing data after writes |

**Key takeaway:** CQRS is a pattern for large, complex systems where reads and writes genuinely need to be scaled, optimized, and modeled independently of each other, at the cost of added architectural complexity and eventual (rather than immediate) consistency between the two sides.
