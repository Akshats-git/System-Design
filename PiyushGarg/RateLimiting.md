# Master Rate Limiting - System Design

Source video: [Master Rate Limiting - System Design](https://www.youtube.com/watch?v=CVItTb_jdkE) (Hindi, about 40 minutes)

These notes are written in simple English based on the transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow. Rate limiting was introduced briefly in [CrashCoursePart1.md](CrashCoursePart1.md); this video goes much deeper into the actual algorithms used to implement it.

---

## 1. Recap: The Request-Response Cycle

Imagine a simple system: some users, and a server.

```
User
  |
  |  sends a request
  v
Server
  |
  |  processes the request (application logic, database queries,
  |  in-memory calculations, validations, and so on)
  v
Response is sent back to the user
```

This is called a typical **request-response cycle**. The word "processing" here is just a broad umbrella term for whatever work the server actually does to handle that request.

---

## 2. Why Rate Limiting Is Needed

A server, no matter how well built, is ultimately just a physical machine, and every machine has limits.

> **Example:** Suppose a server has 2 vCPUs and 4GB of RAM. To keep things simple, let's say this particular machine can comfortably handle about 100 requests per minute. This becomes our "happy" number, the safe zone we want to stay within. These numbers are never exact hard limits. Going slightly over, say to 110 or 120 requests, usually won't instantly crash the server, but it does increase the risk of performance issues. If requests climb to something like 150, performance problems become likely, and the server may start to overheat. If requests reach something like 200, a crash becomes very likely. So 100 requests per minute isn't a hardcoded ceiling, it's a safe approximation of what the server can reliably handle.

Since we cannot directly control how many users show up, or how many requests a single user might try to send, we need a dedicated layer that enforces this limit. This is exactly what a **rate limiter** does.

> **Definition:** A rate limiter is a component placed in front of a server (or service) that enforces a configured limit on how many requests are allowed through in a given period of time. Requests within the limit are allowed to pass through to the server. Requests beyond the limit are rejected.

```
User
  |
  v
Rate Limiter (configured with a threshold, e.g. 100 requests/minute)
  |--> Within threshold     --> forward the request to the Server
  |--> Threshold exceeded   --> drop the request
```

When a request is dropped, the rate limiter typically responds with HTTP status code **429**.

> **Definition:** HTTP status code 429, "Too Many Requests," indicates that the client has sent too many requests in a given period of time, and the server is (temporarily) refusing to process any more from that client.

This is the high-level picture of what a rate limiter is: a layer that limits the rate of requests reaching your server, which is exactly why it's called a rate limiter. The more interesting question is: **how do you actually implement one?** This is where specific rate limiting algorithms come in.

> **Reference mentioned in the video:** the creator refers to a book by ByteByteGo (specifically its chapter on rate limiting) as a great resource that goes into more depth on this topic and these algorithms.

---

## 3. Algorithm 1: Token Bucket

> **Definition:** In the token bucket algorithm, a "bucket" holds a limited number of tokens. A refiller process adds new tokens to the bucket at a fixed rate, up to the bucket's maximum capacity. Every incoming request must consume one token before it is allowed through. If the bucket has no tokens left, the request is rejected.

**Example setup:** Let's say the bucket has a capacity of 5 tokens, and the refiller adds 3 tokens per second. Initially, the bucket starts empty.

```
Second 1: Refiller adds 3 tokens -> Bucket now has 3 tokens

Request 1 arrives -> a token is available -> take 1 token -> request is processed
                                                            -> Bucket now has 2 tokens
Request 2 arrives -> a token is available -> take 1 token -> request is processed
                                                            -> Bucket now has 1 token
Request 3 arrives -> a token is available -> take 1 token -> request is processed
                                                            -> Bucket now has 0 tokens
Request 4 arrives -> no tokens available -> request is REJECTED (429 Too Many Requests)

Second 2: Refiller adds 3 more tokens -> Bucket now has 3 tokens again
   ...and the cycle continues
```

If, at some point, no requests come in for a while, the refiller keeps adding tokens each second, but the bucket never exceeds its maximum capacity (5, in this example). Any tokens beyond the capacity are simply not added; they overflow.

```
      +------------------+
      |     Refiller      |  adds tokens at a fixed rate (e.g. 3/second)
      +---------+--------+
                |
                v
         +-------------+
         |   Bucket    |  <-- capacity limit (e.g. 5 tokens max)
         +------+------+
                |
        request consumes 1 token
                |
      token available? ----No----> reject request (429)
                |
               Yes
                |
                v
         forward request to server
```

### Pros and Cons of Token Bucket

**Pros:**
- Simple and easy to implement.
- Memory efficient.
- Allows short bursts of traffic, since a request can go through as long as tokens are still available in the bucket.
- Widely used in production by companies like Amazon and Stripe to throttle their API requests.

**Cons:**
- There are two parameters to tune: the bucket's capacity, and the token refill rate. Choosing the right values for these in a real production system can be genuinely difficult, and often requires trial and error.

---

## 4. Algorithm 2: Leaky Bucket

> **Definition:** The leaky bucket algorithm is similar to the token bucket, except that instead of tracking tokens, incoming requests are placed into a queue (the "bucket"), and they are processed, or "leaked" out, at a fixed rate. If the queue is full, new requests are dropped.

> **Analogy used in the video:** think of an overhead water tank on top of a house, holding maybe 1000 to 1500 liters. If you opened a large pipe directly underneath it, all that water would rush out at once, more than you could handle. Instead, a shower fixture controls the flow, letting water out at a slow, steady, manageable rate. A leaky bucket rate limiter works the same way: requests can arrive quickly and in large numbers, but they are only ever processed, or let through, at a fixed, steady rate.

```
Users
  |
  |  submit requests very quickly (e.g. 20,000 requests/minute)
  v
Bucket (a queue, with a maximum capacity, e.g. 10,000)
  |
  |  requests "leak" out one at a time, at a fixed rate (e.g. 5 requests/minute)
  v
Server processes each request as it leaks out

If the bucket (queue) is already full when a new request arrives:
   -> the request is dropped (429 Too Many Requests)
```

### Pros and Cons of Leaky Bucket

**Pros:**
- Memory efficient, given a limited queue size.
- Requests are processed at a fixed, predictable rate, which is well suited to use cases that need a stable, steady outflow rate.

**Cons:**
- A burst of traffic can fill up the entire queue with old requests. If a huge number of requests suddenly arrive from a single user, the queue can fill up entirely, and new, possibly more important, requests may never get a chance to be processed.
- Just like the token bucket, there are two parameters to tune here too: the queue's capacity, and the fixed leak (processing) rate. Getting these right in practice often takes trial and error.

---

## 5. Algorithm 3: Fixed Window Counter

> **Definition:** The fixed window counter algorithm divides time into fixed-size windows (for example, one-second windows, or one-minute windows). Each incoming request increments a counter for the current window. Once that counter reaches a predefined threshold, any further requests within that same window are rejected. As soon as a new window begins, the counter resets back to zero.

**Example:** Suppose the threshold is 3 requests per second.

```
Window 1 (0s to 1s):
   Request arrives at 0.0s  -> counter = 1 -> allowed
   2 requests arrive at 0.5s -> counter = 3 -> allowed
   (window ends, counter resets)

Window 2 (1s to 2s):
   2 requests arrive -> counter = 2 -> allowed
   3 requests arrive -> counter would become 5
      -> first request allowed (counter = 3, right at the threshold)
      -> remaining 2 requests rejected (429 Too Many Requests)
   (window ends, counter resets)

Window 3 (2s to 3s):
   3 requests arrive -> counter = 3 -> allowed
   any further requests in this window -> rejected
```

```
Requests
   ^
   |  ###           ###           ###
   |  ###   x x     ###   x x     ###
   +--+---+---+---+---+---+---+---+---> Time (seconds)
      |Window1|  |Window2|  |Window3|
```

### The Core Problem: Traffic Bursts at Window Edges

This algorithm has a genuine bug, which becomes clear when you look closely at what happens right at the boundary between two windows.

> **Example of the edge problem:** Suppose Window 1 covers 0s to 1s, and Window 2 covers 1s to 2s. Now imagine 2 requests arrive at 0.5s (near the end of Window 1) and 2 more requests arrive at 1.5s (near the start of Window 2). Each window individually only counted 2 requests, well within the threshold of 3. But if you look at any actual, continuous 1-second slice that straddles the boundary, for example from 0.5s to 1.5s, you'll find that 4 requests were processed within that single second, even though the threshold was only supposed to allow 3 requests per second.

```
        Window 1 (0s-1s)        Window 2 (1s-2s)
      +-------------------+-------------------+
      |             xx    |xx                 |
      +-------------------+-------------------+
                    ^-------- 1 second --------^
                 (0.5s to 1.5s: 4 requests actually got through)
```

> **The core problem:** a major flaw of the fixed window counter algorithm is that it can allow a burst of traffic right at the edges of two adjacent windows, resulting in more requests going through in some real one-second span than your threshold was ever meant to allow.

### Pros and Cons of Fixed Window Counter

**Pros:**
- Simple to reason about and implement.
- Resetting the available quota cleanly at the end of each fixed time unit works fine for many use cases, since even a small, occasional overshoot (say, one or two extra requests) is often still within a server's actual safety margin.

**Cons:**
- The edge-burst problem described above is a fundamental flaw: traffic spikes right at window boundaries can let through noticeably more requests than intended.

---

## 6. Algorithm 4: Sliding Window Log

> **Definition:** The sliding window log algorithm fixes the edge-burst problem of the fixed window counter. Instead of a fixed counter that resets at fixed intervals, it keeps a log (a list) of the exact timestamp of every request. Whenever a new request arrives, all timestamps older than the start of the current sliding window (that is, older than "now minus the window size") are removed from the log first. Only after that cleanup does it check whether the log size is still within the allowed threshold.

**Example:** Suppose the threshold is 3 requests per 5 seconds.

```
Request at 1s  -> log: [1]                          -> size 1, within threshold -> allowed
Request at 3s  -> log: [1, 3]                       -> size 2, within threshold -> allowed
Request at 4.5s -> log: [1, 3, 4.5]                 -> size 3, within threshold -> allowed

Request at 6s:
   Sliding window is now [1s, 6s] (last 5 seconds)
   Timestamp 1 has expired (older than 1s ago) -> remove it
   Log becomes: [3, 4.5]                      -> size 2, within threshold
   Add new timestamp 6 -> log: [3, 4.5, 6]    -> allowed

Requests at 7s (two of them):
   Sliding window is now [2s, 7s]
   Timestamp 3 has expired -> remove it
   Log becomes: [4.5, 6]                      -> size 2
   First request would make it size 3 -> still allowed? No: threshold already
   effectively reached once both are considered, so both extra requests here
   are rejected since the log would exceed the threshold of 3.

Request at 8s:
   Sliding window is now [3s, 8s]
   Timestamp 4.5 has expired -> remove it
   Log becomes: [6]                            -> size 1, within threshold
   Add new timestamp 8 -> log: [6, 8]          -> allowed
```

```
Log of timestamps: [ t1, t2, t3, ... ]
   |
   |  new request arrives at time "now"
   v
Remove every timestamp older than (now - window size)
   |
   v
Is the remaining log size < threshold?
   |--> Yes --> add "now" to the log -> allow the request
   |--> No  --> reject the request (429 Too Many Requests)
```

This is called a "sliding" window because the window is always measured relative to the current moment ("the last 5 seconds," continuously), rather than being anchored to fixed clock boundaries like 0-5s, 5-10s, and so on. Because of this, it correctly avoids the edge-burst problem seen in the fixed window counter.

### The Trade-off

This approach is more accurate, but it requires storing every individual request's timestamp (typically in a cache), rather than just a single counter, which uses more memory as traffic grows.

---

## 7. Algorithm 5: Sliding Window Counter

> **Definition:** The sliding window counter algorithm is a hybrid approach that combines the fixed window counter and the sliding window log. It keeps the simplicity of fixed-size windows and counters (from the fixed window counter), but adjusts the count using a weighted portion of the previous window, based on how far into the current window a new request falls (similar in spirit to the sliding window log).

**How it works:** Suppose you're using 1-second fixed windows, and a new request arrives at, say, 10% into the current window (meaning it's very close to the boundary with the previous window). Instead of only counting requests in the current window so far, this algorithm also considers a proportional share, in this case roughly 90%, of the previous window's request count, since so much of the "recent past" actually still belongs to that earlier window from a real-time perspective.

```
Previous Window (e.g. 3s-4s)         Current Window (e.g. 4s-5s)
      [ N requests ]                     [ request arrives at 4.1s, i.e. 10% in ]
                                                |
                          estimated count = (90% x N from previous window)
                                              + (requests so far in current window)
                                                |
                       is this estimated count within the threshold?
                          |--> Yes --> allow the request
                          |--> No  --> reject the request
```

> **Example:** If a request arrives 30% of the way into the current window, the algorithm looks at 70% of the previous window's count, adds it to however many requests have occurred so far in the current window, and checks that combined estimate against the threshold. This gives a smoother, more accurate approximation of "requests in the last full window's worth of time," without needing to store every single timestamp the way the sliding window log does.

This is why it's called a sliding window **counter**: you still keep simple, fixed-size window counters (cheap to store), but you slide a weighted estimate across the boundary between windows, rather than resetting abruptly.

---

## Summary: All Rate Limiting Algorithms Covered

| Algorithm | How It Works | Main Strength | Main Weakness |
|---|---|---|---|
| Token Bucket | Tokens refill at a fixed rate into a capped bucket; each request consumes one token | Simple, memory efficient, allows short bursts | Bucket size and refill rate can be hard to tune correctly |
| Leaky Bucket | Requests queue up and are processed ("leaked") at a fixed rate | Smooths traffic into a steady, predictable outflow | A burst can fill the queue with old requests, blocking new ones |
| Fixed Window Counter | Counts requests within fixed time windows (e.g. per second), resetting each window | Simple to implement and reason about | Can allow traffic bursts right at window edges, exceeding the intended threshold |
| Sliding Window Log | Keeps a timestamp log of every request; removes expired timestamps as the window slides forward | Accurate; fixes the edge-burst problem | Uses more memory, since every request's timestamp must be stored |
| Sliding Window Counter | A hybrid: fixed-size window counters, weighted by how far into the window a request falls | Good accuracy with much less memory than the sliding window log | Slightly more complex to implement than a plain fixed window counter |

**Key takeaway:** all rate limiting strategies exist to answer the same underlying question, how many requests should be allowed through in a given period of time, but they differ in how precisely they track that rate, and how much memory and complexity they're willing to trade for that precision. Token bucket and leaky bucket are simple and widely used in production; sliding window log and sliding window counter exist specifically to fix the boundary-related inaccuracies of the fixed window counter approach.
