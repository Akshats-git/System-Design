# System Design Crash Course - Part 1 (Beginner Friendly)

Source video: [System Design for Beginners](https://www.youtube.com/watch?v=lFeYU31TnQ8) (Hindi, about 47 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow.

---

## 1. The Big Picture: Client and Server

At the highest level, every system is made up of just two parts. There is a **client**, which is anything that uses the application, such as a mobile app, a laptop browser, or an IoT device. And there is a **server**, which is a machine that serves the application to the client.

A server is nothing special. It is simply a computer that has two properties. First, it runs 24 hours a day, 7 days a week. Second, it has a public IP address, so that it can be reached from anywhere on the internet.

> **Example:** Your own laptop could technically act as a server if you kept it switched on all the time and gave it a public IP address. In practice, nobody does this, because of power costs, reliability, and security concerns. This is exactly why we rent servers from cloud providers such as AWS or DigitalOcean instead. These providers simply run machines 24/7 on our behalf and give them public IP addresses.

```
Client (mobile / laptop / IoT)
   |
   |  sends request
   v
Server (always-on machine, has a public IP)
   |
   |  sends response back
   v
Client
```

---

## 2. IP Address and DNS

An **IP address** is a numeric address, such as `10.2.3.4`, that uniquely identifies a machine on the internet. It works a bit like a postal address: a house number, a street, a city, a pin code, a state, and finally a country, all combine to identify one exact location.

The problem is that IP addresses are hard to remember, in the same way most people cannot recall their friends' phone numbers from memory. This is where **DNS (Domain Name System)** comes in.

> **Definition:** DNS is a global, decentralized directory that maps human-friendly domain names, such as `amazon.com`, to the actual machine IP addresses behind them.

This lookup process is called **DNS Resolution**, and it works in three simple steps:

```
Step 1: The browser asks DNS, "What is the IP address for amazon.com?"
Step 2: DNS replies, "amazon.com is at IP 10.2.3.4"
Step 3: The browser now sends the real request directly to 10.2.3.4
```

Whenever you buy a domain and point it to a server's IP address, that mapping gets stored on DNS servers around the world. This is also why DNS is described as decentralized: no single company owns or controls the whole system.

---

## 3. Vertical Scaling (Scaling Up)

As the number of users grows, the request load on a server also grows. A small machine, for example one with 2 CPUs and 4GB of RAM, will eventually become a **bottleneck**. It can run out of memory or CPU capacity, which causes the server to crash. This is exactly why websites tend to "go down" during periods of very high traffic, such as when exam results are released on an education board's website.

> **Definition:** Vertical scaling, also known as scaling up, means increasing the capacity of a single machine by adding more CPU, RAM, or disk space to it.

```
Before upgrade:  1 machine with  2 CPU  and   4 GB RAM
After upgrade:   1 machine with 64 CPU  and 128 GB RAM
```

The machine is still a single server. It has simply been made more powerful.

There are two important problems with vertical scaling.

**It can be wasteful.** If you upgrade your server to handle the worst-case, peak traffic scenario, you end up paying for a lot of unused capacity most of the time. Amazon's traffic, for instance, is not always high. It spikes mainly during festival seasons and big sales, so a server sized for that peak sits mostly idle the rest of the year.

**It causes downtime.** You cannot add CPU or RAM to a machine while it is running. The server has to be shut down, upgraded, and then restarted, and this restart takes time. Now imagine a flash sale, such as "Big Billion Days," that is scheduled to start at exactly 12:00 PM. If the server needs to restart right around that time to scale up, even one minute of downtime at that critical moment is unacceptable. Large systems need zero downtime, and vertical scaling alone cannot guarantee that.

---

## 4. Horizontal Scaling (Scaling Out)

Instead of making one server bigger, we can add more servers that each run a copy of the same application. This is called **horizontal scaling**.

> **Definition:** Horizontal scaling, also known as scaling out, means adding more servers (replicas) to share the load, rather than increasing the resources of a single server.

```
Client
  |--> Server 1 (IP: 10.2.3.5)
  |--> Server 2 (IP: 10.2.3.6)
  |--> Server 3 (IP: 10.2.3.7)
```

The big advantage here is that there is no downtime. New machines can be started up in parallel while the existing ones continue to serve traffic as normal.

This does introduce a new problem, though. DNS can only point a domain name to one IP address, but now we have three different servers with three different IP addresses. So how should incoming traffic be distributed among them?

---

## 5. Load Balancer

> **Definition:** A load balancer is a server placed in front of your replica servers. It receives all incoming traffic first, and then distributes, or "balances," that traffic across the available servers.

```
Client
  |
  v
Load Balancer
  |--> Server 1
  |--> Server 2
  |--> Server 3
```

The DNS record now points to the load balancer's IP address, instead of pointing to any individual server. The load balancer's job is simply to decide which incoming request should go to which server.

**A simple load balancing algorithm: Round Robin.** Requests are distributed in a repeating cycle. The first request goes to Server 1, the second to Server 2, the third to Server 3, the fourth back to Server 1, and so on, cycling through the servers equally.

**Health checks.** When a new server starts up, it gets registered with the load balancer. However, the load balancer only starts sending it traffic once that server is confirmed to be "healthy," meaning it has been up and stable for a short period of time.

**Auto scaling.** When traffic increases, new servers can be started up and registered with the load balancer automatically. When traffic decreases, some servers can be removed, which is called scaling in. This is how cloud auto-scaling policies work in practice.

In AWS terminology, this component is called an **ELB (Elastic Load Balancer)**. Internally, an ELB is itself a robust, distributed system with many workers behind it, so it does not easily become a bottleneck on its own.

**Key distinction to remember:** with vertical scaling, there is only one server and one IP address, so no load balancer is needed. With horizontal scaling, there are multiple servers and multiple IP addresses, so a load balancer becomes necessary.

---

## 6. Microservice Architecture and API Gateway

In a real system like Amazon, there is not just one application. There are many independent **microservices**, each handling one specific responsibility. For example, an **Auth Service** handles login and authentication, an **Orders Service** manages orders, a **Payments Service** handles payments, and a general **API Service** handles most other API traffic.

Each of these microservices can scale independently, using horizontal scaling, based on its own load. For example, the Auth service might run on 4 servers, the Orders service on 3 servers, and the Payments service on just 2 servers, since payment traffic is usually much lower than general browsing or API traffic.

This raises a new question. If a request comes in for `amazon.com/auth/...`, how does the system know to send it to the Auth service, while a request for `amazon.com/orders/...` should go to the Orders service instead?

> **Definition:** An API Gateway is a centralized entry point for all incoming API calls. It acts as a reverse proxy, meaning it routes each incoming request to the correct backend service (and that service's load balancer), based on rules such as the request's path, and then sends the response back to the client.

```
Client
  |
  v
API Gateway (single public IP, registered in DNS)
  |
  |--> /auth/*      --> Load Balancer (Auth)     --> Auth service instances
  |--> /orders/*    --> Load Balancer (Orders)   --> Orders service instances
  |--> /payments/*  --> Load Balancer (Payments) --> Payments service instances
```

The API Gateway has its own public IP address, and this is the address registered in DNS. Internally, it looks at the incoming request, matches it against a routing rule, and forwards it to the load balancer of the correct microservice. From there, the load balancer of that specific microservice picks one of its own EC2 instances (Amazon's term for a virtual machine) to actually handle the request.

You can also attach an authenticator, such as Auth0, at the API Gateway level, so that users are verified before their request is routed any further.

---

## 7. Background Workers and Batch Processing

Some tasks are simply too heavy to perform inside a normal, live request-response cycle. For example, sending an email to one million users, or processing a large, bulk-uploaded CSV file, cannot reasonably be done by running a million-iteration loop inside a single web request.

> **Definition:** A worker, sometimes called a background worker, is a service that runs independently of the live request-response cycle, in order to process such bulk or long-running tasks.

**A naive, synchronous example:**

```
Payment Service
   |
   |  makes an HTTP call and waits for a response
   v
Email Worker
   |
   |  calls an external API
   v
Gmail API --> email is sent
```

This approach is called **synchronous**, because the Payment Service has to wait until the email is actually sent before it can continue. This does not scale well if payments are happening every second, since the payment flow itself is now blocked by a slow, external dependency, in this case Gmail's API.

---

## 8. Asynchronous Communication: Message Queues

To avoid this kind of blocking, we can decouple services from each other using a **queue**, which is an asynchronous way of passing information between services.

> **Definition:** A queue is a First-In-First-Out (FIFO) data structure used to pass messages or events between services, without either side having to wait for the other to finish.

```
Payment Service
   |
   |  pushes an event (for example, an order ID)
   v
Queue: [order1, order2, order3, ...]
   |
   |  Email Worker pulls one event at a time
   v
Email Worker --> Gmail API --> email sent
```

The Payment Service simply pushes an event, such as an order ID, onto the queue, and then moves on immediately without waiting. The Email Worker pulls one event at a time from the queue, sends the corresponding email, discards that event, and then moves on to the next one. This is exactly why the confirmation email after placing an order usually arrives a couple of seconds late: the worker is working through the queue one item at a time.

You can also scale the email workers horizontally, running several of them at once, to increase parallelism. This means that instead of processing one email at a time, you might process two, three, or more at the same time.

**Using a queue to enforce rate limits.** Suppose Gmail's API only allows 10 emails to be sent per second. You can configure the worker to pull at most 10 messages per second from the queue. This way, the queue naturally creates a controlled bottleneck, ensuring you never exceed the external API's rate limit.

The equivalent AWS service for this pattern is **SQS (Simple Queue Service)**.

**Pull versus push mechanisms.** In a pull mechanism, also called polling, the worker repeatedly asks the queue whether it has any new messages. There are two common styles of polling. **Short polling** means asking every second or so, which results in more API calls and therefore a higher cost. **Long polling** means asking once and then waiting, for example for 10 seconds, collecting whatever messages arrive during that window before processing them together. This is generally more efficient. In a push mechanism, the queue itself proactively invokes or notifies the worker whenever a new message arrives. SQS specifically uses the pull, or polling, model.

---

## 9. Pub/Sub Model and Event-Driven Architecture

A plain queue has one important limitation: it is strictly one-to-one. Once a single consumer picks up a message, no other consumer will ever see that same message. But sometimes, one single event, such as "a payment was made," needs to trigger several independent actions at once: sending an email, sending a WhatsApp message, notifying the vendor, and sending an SMS.

> **Definition:** Pub/Sub, short for Publish-Subscribe, is a messaging pattern where a publisher emits, or "publishes," an event just once, and multiple independent subscribers can each receive and react to that same event.

The equivalent AWS service for this pattern is **SNS (Simple Notification Service)**.

```
Payment Service
   |
   |  publishes a "payment made" event
   v
SNS (notification topic)
   |--> Email Worker
   |--> WhatsApp Service
   |--> SMS Service
```

This general style, where something happens, an event is announced, and anyone interested can react to it, is called **event-driven architecture**.

There is an important limitation of a pure Pub/Sub system like SNS: there is no acknowledgement. If a subscriber fails to process the event, for example because the WhatsApp API happened to be down at that moment, there is no built-in retry mechanism. The message is simply lost, unless you build your own custom retry logic on top.

Compare this to a queue-based system like SQS, where failure handling is easier. You can re-enqueue a failed message so it gets processed later, or you can push it to a **Dead Letter Queue (DLQ)**, which is a special queue that holds messages that failed processing, so that you can inspect or retry them afterward.

---

## 10. Fan-Out Architecture

To get the benefits of both approaches, multiple independent subscribers, plus reliable acknowledgement and retries, we can combine Pub/Sub with queues. This combined pattern is called **fan-out architecture**.

> **Definition:** Fan-out architecture publishes one event to a notification service, such as SNS, which then pushes a copy of that message into several separate queues, such as SQS queues, with one queue dedicated to each downstream service.

```
Payment Service
   |
   |  publishes a "payment made" event
   v
SNS (notification topic)
   |--> WhatsApp Queue --> WhatsApp Worker(s)
   |--> Email Queue    --> Email Worker(s)
   |--> SMS Queue      --> SMS Worker(s)
```

**A real-world example: video upload processing, similar to YouTube.**

```
Video uploaded to S3
   |
   v
SNS publishes an "upload complete" event, which fans out to:
   - a queue for transcoding to 480p
   - a queue for transcoding to 360p
   - a queue for extracting audio only
   - a queue for generating a thumbnail
```

Each of these queues has its own dedicated worker or workers processing that specific job. Because each branch is backed by its own queue, every branch also gets proper retries and acknowledgements if something fails.

---

## 11. Rate Limiting

> **Definition:** Rate limiting restricts how many requests a client, or the system as a whole, can send or process within a given window of time. Its main purpose is to prevent abuse, such as fake requests or DDoS attacks, and to protect backend resources from being overwhelmed.

> **Example:** A system might allow only 5 requests per second from a single client. Any request beyond that limit gets rejected with a `429 Too Many Requests` error.

Rate limiting can be applied at more than one level. It can be applied at the client or user level, to stop one particular user from spamming requests. It can also be applied at the load balancer or system level, in order to control the overall rate of requests being forwarded onward to internal servers or to external third-party APIs.

**Common rate-limiting strategies:**

- **Token Bucket:** A "bucket" holds a limited number of tokens, and new tokens are added back at a fixed rate over time. Every incoming request consumes one token. If the bucket runs out of tokens, further requests are rejected until it refills.
- **Leaky Bucket:** Requests are processed at a constant, fixed rate, similar to water leaking out of a bucket at a steady pace. This smooths out sudden bursts of traffic into a steady stream.

**Further reading:** [API Rate Limiting Strategies: Token Bucket vs Leaky Bucket](https://www.eraser.io/decision-node/api-rate-limiting-strategies-token-bucket-vs-leaky-bucket)

The video also references a companion article titled "API Rate Limiting Strategies," written by the same creator, which covers these strategies in more depth with code examples.

---

## 12. Database Scaling: Read Replicas

As more and more microservices grow, they all end up querying a single shared **database**, and this database can itself become a bottleneck.

> **Definition:** A read replica is a copy of the primary, or master, database that is used only for read operations. This reduces the load placed on the primary node.

```
App / Microservices
   |--> writes                          --> Primary (Master) Node
   |--> real-time, critical reads       --> Primary (Master) Node
   |--> analytics and reporting reads   --> Read Replica 1, Read Replica 2
```

All write operations, such as inserts and updates, must always go to the primary or master node. Real-time, critical reads should also go to the primary node, because read replicas can have a slight replication delay, meaning the data inside them may be a little bit out of date. Reads that are not time-critical, such as analytics dashboards, logs, or reports, can instead be redirected to a read replica. This frees up capacity on the primary node for the operations that truly need it.

---

## 13. Caching

> **Definition:** A cache is a fast, in-memory data store used to hold frequently accessed data, so that repeated requests do not need to hit the slower database every single time.

A commonly used tool for this is **Redis**, which is an in-memory database often used purely as a cache.

```
Request
   |
   v
Check Cache (Redis)
   |--> Found     --> return the cached result directly
   |--> Not found --> query the Database
                        --> store the result in the Cache
                        --> return the result to the client
```

The main benefit of a caching layer is that it significantly reduces the number of direct calls made to the database, which leads to faster response times and a more robust system overall.

---

## 14. CDN (Content Delivery Network)

If a load balancer is exposed directly to users all over the world, then every single request, no matter where the user is physically located, has to travel all the way to wherever the origin servers happen to be. For example, a user in India making a request to a server located in the United States would experience significant latency.

> **Definition:** A CDN, or Content Delivery Network, is a globally distributed network of edge servers, meaning small servers placed in many regions around the world, that cache content closer to users and route each user's request to their nearest available edge location.

In AWS, this service is called **CloudFront**.

```
Client in North India --> nearest --> Edge Server (North India)
Client in Canada       --> nearest --> Edge Server (Canada)
Client in the US       --> nearest --> Edge Server (US)
```

**How a user gets routed to the nearest edge server: Anycast.**

> **Definition:** Anycast is a networking technique where the same IP address is announced from many different physical locations at once. The network automatically routes each user's request to whichever location advertising that IP address is geographically or topologically nearest to them.

**How caching at the edge actually works, step by step:**

```
Each Edge Server:
   |--> Cache hit  --> return the content directly from the edge (very fast)
   |--> Cache miss --> forward the request to the Load Balancer
                         --> Origin Server processes the request
                         --> response is cached at this edge server
                         --> response is returned to the user
```

The first time a user in North India requests a particular photo, that photo is not yet cached at the nearby edge server. So the request is forwarded onward to the load balancer, and then to the origin server, which returns the photo. That photo is then cached at the North India edge server as well as being returned to the user. The next time any user near that same edge server requests the same photo, it is served directly from the edge cache, with no need to contact the origin server at all.

This approach has several clear benefits. It reduces latency, since content is served from a nearby location rather than a distant origin server. It reduces load on the origin server, since many requests are served entirely from edge caches and never even reach the backend. It also saves overall network bandwidth.

> **Example:** On Amazon, product photos and videos are cached at the CDN edge location nearest to a particular region. Once a single user in that region has viewed a product, every other nearby user gets the same images and videos served instantly from the local cache, instead of from the origin server.

---

## Summary: All Components Covered

| Component | Purpose |
|---|---|
| Client / Server | The basic request and response participants in any system |
| IP Address / DNS | Locating and naming servers on the internet |
| Vertical Scaling | Increasing a single server's resources (has downtime) |
| Horizontal Scaling | Adding more server replicas (no downtime) |
| Load Balancer | Distributing traffic across replica servers |
| Microservices | Independent services for each domain, such as auth, orders, and payments |
| API Gateway | A central entry point that routes requests to the correct microservice |
| Background Workers | Handling long-running or bulk tasks outside the live request-response cycle |
| Message Queue (SQS) | Asynchronous, one-to-one, ordered communication between services |
| Pub/Sub (SNS) | Broadcasting one event to many subscribers (with no acknowledgement) |
| Fan-Out Architecture | Combining Pub/Sub and queues for reliable, multi-service notification |
| Rate Limiting | Controlling request rate to prevent abuse and overload |
| Read Replicas | Offloading read traffic away from the primary database |
| Caching (Redis) | Serving frequently used data quickly, without hitting the database |
| CDN (CloudFront) | Serving and caching content from the nearest geographic edge location |

**What comes next in this series, according to the video:** containers, Docker, containerization, and container orchestration.
