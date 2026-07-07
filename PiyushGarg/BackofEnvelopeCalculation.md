# Back of the Envelope Calculation - System Design Concept

Source video: [Back of Envelope Calculation - System Design Concept](https://www.youtube.com/watch?v=DwqTon7ZS_s) (Hindi, about 17 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow. This is a short video, but the concept it covers is genuinely important and often overlooked, especially for anyone working at a senior level where the job includes designing and architecting systems. It is also something many freshers miss, and it is meant to be done before you actually start designing a system.

**Further reading, added alongside the video's own references:** [ByteByteGo: Back-of-the-Envelope Estimation](https://bytebytego.com/courses/system-design-interview/back-of-the-envelope-estimation)

---

## 1. Why Do We Need Back-of-the-Envelope Calculations?

Whenever we design a system, we need certain basic physical requirements. At a bare minimum, we need a compute machine (a server), a database, and some storage. These are the absolute minimum requirements just to get a system up and running: storage to store media, a database to store data, and a server to actually serve requests.

**The real problem:** we don't actually know how much of each of these we need. How much storage do I need? How large or "strong" does my database need to be? How many servers do I need? Nobody knows these numbers upfront.

> **Example from the video:** While deploying a recent project, the DevOps team asked a series of very specific questions: how many physical servers do you need, and what configuration (how much CPU and how much memory)? If you need a managed database service, how much CPU and RAM does that need? How much storage or disk space do you need, and how many GB of SSD?

If you are a fresher, or if you simply don't have this experience yet, these questions can feel overwhelming. You don't necessarily know how your system performs, what scale you are operating at, how many users you have, or how the load actually behaves. So you end up guessing randomly, for example asking for an 8-CPU machine with no real basis for that number. This can go wrong in two different ways: you might end up **wasting resources** (paying for an 8-CPU machine you never fully use), or you might **under-provision** (choosing a 2-CPU machine to cut costs, which then crashes under real load).

**This is exactly the gap that back-of-the-envelope calculations fill:** a structured way to arrive at reasonable, defensible resource estimates, before you actually commit to a specific infrastructure setup.

> **Where the name comes from:** This concept is credited to a presentation by Jeff Dean, a well-known Google scientist and engineer. The name "back of the envelope" refers to the idea of doing a rough calculation on the back of a physical envelope (the kind you'd mail a letter in), rather than a fully structured, precise calculation. Everything here is a rough estimate: if we expect this many users, storing this much data, with this many daily active users, then roughly this much infrastructure should be enough to get by.

> **Definition:** A back-of-the-envelope calculation is a deliberately rough, approximate estimate of a system's expected scale (traffic, storage, throughput, and so on), used to make reasonable infrastructure decisions when exact numbers are not available.

---

## 2. Foundational Math You Should Know

Before you can do these calculations comfortably, there are a few basic numbers and unit conversions you should have ready in your head.

### Seconds in a Day

```
24 hours  x  60 minutes  x  60 seconds  =  86,400 seconds in a day
```

**Rule number one of back-of-the-envelope calculations: approximate everything.** Round numbers off. Since we are only ever dealing with estimates here, 86,400 can comfortably be rounded to approximately **100,000 (one lakh) seconds per day**, just to make the surrounding math simpler. Nothing here needs to be exact.

### Byte, KB, MB, GB, TB, and PB

You should be comfortable converting between these units:

```
1 Byte      =  base unit (think of it as 10^0)
1 Kilobyte  =  1,000 Bytes            (10^3, a thousand)
1 Megabyte  =  1,000 Kilobytes        (10^6, a million)
1 Gigabyte  =  1,000 Megabytes        (10^9, a billion)
1 Terabyte  =  1,000 Gigabytes        (10^12, a trillion)
1 Petabyte  =  1,000 Terabytes        (10^15, a quadrillion)
```

Each step simply multiplies by roughly 1000. Once you're comfortable with seconds-per-day and these unit conversions, you're basically ready to start doing back-of-the-envelope calculations.

---

## 3. Latency Numbers Every Programmer Should Know

Before doing any calculation, it also helps to have a rough sense of how long different operations actually take. This is commonly referred to as **"Latency Numbers Every Programmer Should Know,"** and it's covered well in the ByteByteGo article linked at the top of these notes.

A few reference points mentioned in the video:

```
L1 cache reference               ~ 0.5 nanoseconds
L2 cache reference                 ~ a few nanoseconds
Main memory (RAM) reference        ~ 100 nanoseconds
Disk seek                          ~ 10 milliseconds
```

**The key intuition to take away from this:**

- **Memory (RAM) is fast, but volatile.** Data in memory disappears if power is lost or the process restarts.
- **Disk is slow, but persistent.** Data on disk is retained permanently, which is exactly why querying a database (which reads from disk) takes noticeably longer than querying something cached in memory (like Redis).
- **Avoid disk seeks if possible**, since they meaningfully increase latency.
- **Simple compression algorithms are fast; more complex compression algorithms take more time.** Choose accordingly, based on whether you need speed or a smaller size more.
- **Compress data before sending it over the network, if possible.**
- **Data centers are often in different geographic regions, and sending data between them takes time.** For example, if you are in India and talking to a server hosted in the US, that physical distance adds real latency.

Keeping these rough numbers in mind is part of what makes your back-of-the-envelope estimates realistic, rather than pure guesswork.

---

## 4. A Worked Example: Estimating Twitter's QPS and Storage

Let's actually work through an example: estimating the **QPS (Queries Per Second)** and **storage requirements** for something like Twitter.

### Step 1: Write Down Your Assumptions

Since we don't have real data, we start with reasonable assumptions, based on general knowledge of the platform:

```
- 300 million Monthly Active Users (MAU)
- 50% of users are active daily (Daily Active Users)
- On average, each active user posts 2 tweets per day
- 10% of tweets contain photos or other media
- Data must be stored for at least 5 years
```

None of these numbers need to be exactly correct. They are reasonable, round averages. The 50% could just as easily have been 40% or 60%, and the "2 tweets per day" could have been 1, 3, or 4. That's the nature of an estimate.

### Step 2: Calculate Daily Active Users (DAU)

```
DAU = 300 million (MAU) x 50%
    = 150 million daily active users
```

### Step 3: Calculate Tweets QPS (Queries Per Second)

```
150 million daily active users x 2 tweets/day = 300 million tweets/day

300 million tweets/day  /  86,400 seconds/day  ≈ 3,500 tweets per second
```

This tells you that your database needs to be able to handle roughly **3,500 insert operations per second**, just for the act of posting tweets. (This is only for writes/inserts; reads are a separate calculation.)

### Step 4: Calculate Peak QPS

Real traffic isn't perfectly steady. Sometimes a major event happens (say, a globally significant news event) and suddenly far more people start tweeting than usual. To account for this, we estimate a **peak QPS**, often simply by doubling the average:

```
Peak QPS ≈ 3,500 x 2 ≈ 7,000 queries per second
```

This tells you that your system needs to be highly available and scalable enough to comfortably handle bursts of up to roughly 7,000 queries per second, not just the average load.

### Step 5: Calculate Storage Requirements

Now let's estimate storage. Assume each tweet consists of:

```
- A tweet ID:      about 64 bytes (a UUID-like identifier)
- Tweet text:       about 140 bytes on average
- Media (if any):   about 1 MB on average, when present
```

Since 10% of the 300 million daily tweets contain media:

```
150 million daily active users x 2 tweets/day x 10% (with media) x 1 MB
   ≈ 30 million tweets/day with media, at ~1 MB each
   ≈ 30 terabytes of media storage needed per day
```

Now extend this over the required 5-year retention period:

```
30 TB/day x 365 days/year x 5 years ≈ 55 petabytes of storage (approximately)
```

So, this system would need to budget for roughly **55 petabytes of storage** over the next 5 years, just for media attached to tweets.

> **This is a complete back-of-the-envelope calculation:** starting from a handful of reasonable assumptions, and working step by step toward concrete infrastructure numbers (QPS, peak QPS, and storage) that can actually guide real decisions, like how many database write nodes you need, or how much storage capacity to provision.

---

## 5. Useful Tips for Doing These Calculations

- **Round off and approximate everything.** You never need to work with the exact real number. For example, a number like 99,987 divided by 9.1 can comfortably be simplified to something like 100,000 divided by 10. Everything here runs on approximation, not precision.
- **Write down your assumptions clearly.** As shown in the Twitter example above, always note down exactly what you're assuming: how many monthly active users, what percentage are daily active, how many tweets per user per day, and so on.
- **Always label your units.** When doing these calculations, it's important to always write down whether you're talking about megabytes or terabytes, seconds or hours, and so on. Writing just a bare number like "5" without a unit attached is meaningless. Always attach the unit explicitly.
- **Know the commonly asked estimations.** In system design contexts, you'll commonly be asked to estimate things like QPS, peak QPS, and storage requirements, much like the Twitter example above.

> **Important nuance:** these calculations don't follow one single fixed formula. You genuinely get better at them with experience. If you make an estimate, implement your system based on it, and later discover that your real-world resource usage is actually higher than what you calculated, that's valuable feedback. Next time, you'll naturally calculate a bit more accurately, because you now have real experience to draw from. Since everything here is an approximation to begin with, this kind of learning from experience is expected and normal.

---

## 6. A Practical, Personal Approach (From the Video Creator)

When there's no real usage data available yet, a practical strategy is to **start with the bare minimum** rather than over-provisioning from day one.

```
Starting point in a dev environment:
   - 2 CPUs
   - 4 GB RAM
   - 256 GB of storage
   - Roughly 8 to 10 users actively using it
```

There are two good reasons to start this small: first, **cost**, since bigger machines cost more regardless of whether you actually need them. Second, this approach lets you observe how your actual code and logic perform under genuinely limited resources, rather than masking inefficiencies with oversized infrastructure.

Once this is deployed, **monitor it closely**: average CPU utilization, average memory usage, queries per second, and network in/out. This gives you a concrete number: "for 8 to 10 users, this is what our resource usage looks like."

From there, you can extrapolate toward production scale. For example, if you expect 100 users to sign up in production, and you assume that only about 30% of them will be active, returning users (so about 30 daily active users), you can scale your earlier dev-environment numbers up proportionally from the 8-to-10-user baseline to estimate what a 30-user load will actually require, and provision accordingly.

**This entire process, estimating scale and resource needs from a handful of reasonable assumptions, is what is formally known as a back-of-the-envelope calculation.**

---

## Summary: Key Ideas in Back-of-the-Envelope Calculations

| Concept | Purpose |
|---|---|
| Seconds in a day (~86,400, roughly 100,000) | A foundational number used in almost every throughput calculation |
| Byte/KB/MB/GB/TB/PB conversions | Needed to reason about storage requirements at any scale |
| Latency numbers (memory vs. disk, network, compression) | Helps you reason about where real-world slowness comes from |
| Assumptions | The starting inputs (user counts, averages, percentages) that drive every estimate |
| QPS (Queries Per Second) | The average rate of operations (e.g., writes) your system must sustain |
| Peak QPS | A safety margin estimate (often 2x average) for sudden traffic spikes |
| Storage Estimation | Projected data volume over time, based on per-item size and retention period |
| Rounding and Approximation | The core principle: exact precision is never the goal here |
| Labeling Units | Always attach units (MB, TB, seconds, etc.) to avoid meaningless numbers |
| Starting Small and Monitoring | A practical, iterative way to arrive at real infrastructure numbers when no data exists yet |

**Key takeaway:** back-of-the-envelope calculations are a simple but genuinely important system design skill: using rough, clearly labeled assumptions and basic math to arrive at defensible infrastructure decisions (servers, database sizing, storage), instead of guessing randomly or over-provisioning out of caution.
