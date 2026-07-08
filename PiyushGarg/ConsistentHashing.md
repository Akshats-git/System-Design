# Consistent Hashing - System Design

Source video: [Consistent Hashing - System Design](https://www.youtube.com/watch?v=IC5Y1EE-aj4) (Hindi, about 31 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow. Before understanding consistent hashing, it helps to first understand plain hashing, and the specific problem that makes consistent hashing necessary in the first place.

---

## 1. The Problem: Why Do We Need Hashing At All?

Imagine an application with a large number of users and a large amount of data. At some point, a single database becomes a bottleneck, since it simply cannot handle that much data or traffic on its own.

The natural fix is **horizontal scaling**: instead of one database, you deploy several (say, three databases instead of one).

```
Before:  [ Single Database ]  <-- handles 100% of the workload, becomes a bottleneck

After:   [ Database 0 ]  [ Database 1 ]  [ Database 2 ]  <-- workload is split across all three
```

**The first real problem shows up immediately: how do you decide which user's data goes into which database?**

> **Example:** Suppose you have five users: Piyush (ID 1), John (ID 2), Jen (ID 3), Alex (ID 4), and Tiger (ID 5). If you just randomly decide to place Piyush's data into Database 0, that's fine, as long as you can reliably find it there again later. If your lookup logic ever looks somewhere else (say, Database 2) for Piyush's data next time, you will simply never find it, even though it does exist somewhere in the system.

This means you need an algorithm that reliably places a given piece of data into the same partition, every single time, both when writing and when reading it back later.

### A Naive (Broken) Approach

A tempting first idea might be a function that returns a random index every time you insert data:

```
function getPartition(userId):
    return random_number()
```

This is broken, because it isn't **deterministic**. The same user ID could return a different index every time it's called, meaning that data you wrote once could become permanently unreachable the next time you look for it.

### A Better Approach: The Modulus Hash Function

A very simple, working approach is:

```
function getPartition(userId):
    return userId % number_of_servers
```

> **Example (with 3 servers):**
> ```
> Piyush (ID 1): 1 % 3 = 1   -> stored in Database 1
> John   (ID 2): 2 % 3 = 2   -> stored in Database 2
> Jen    (ID 3): 3 % 3 = 0   -> stored in Database 0
> Alex   (ID 4): 4 % 3 = 1   -> stored in Database 1
> Tiger  (ID 5): 5 % 3 = 2   -> stored in Database 2
> ```

Because this function always returns the exact same output for the exact same input, you can reliably find any user's data again later, simply by recomputing the same formula.

> **Definition:** A hash function takes some input (such as a user ID) and produces a deterministic, fixed-size output. "Deterministic" means the same input will always produce the same output, every single time. A hash function is also generally **one-way**, meaning you cannot reverse the output back into the original input. In this context, the hash function's output tells you exactly which partition (server) a piece of data belongs to, and it effectively acts as your load balancer, since it's responsible for spreading data evenly across all the available servers.

Note that a hash function that always returns the same fixed value (say, always `1`) would technically still be "deterministic," but it would send all data to a single server, defeating the whole purpose. A good hash function needs to both be deterministic **and** distribute load evenly across all available servers.

---

## 2. The Real Problem: What Happens When You Add or Remove a Server?

The modulus approach works fine, until your scale changes and you need to add (or remove) a server.

> **Example continued:** With 3 servers, we had: Piyush → 1, John → 2, Jen → 0, Alex → 1, Tiger → 2.
>
> Now suppose traffic grows and you add a fourth server. Your hash function's formula must change too, since it now needs to divide by 4 instead of 3:
>
> ```
> Piyush (ID 1): 1 % 4 = 1   -> still Database 1  (unchanged, lucky)
> John   (ID 2): 2 % 4 = 2   -> still Database 2  (unchanged, lucky)
> Jen    (ID 3): 3 % 4 = 3   -> NOW Database 3  (previously was in Database 0!)
> Alex   (ID 4): 4 % 4 = 0   -> NOW Database 0  (previously was in Database 1!)
> Tiger  (ID 5): 5 % 4 = 1   -> NOW Database 1  (previously was in Database 2!)
> ```

Look closely at what happened: simply by adding **one** new server, 3 out of 5 existing users' data now maps to a completely different partition than where it actually still physically resides. If Jen tries to fetch her data, the hash function will point to Database 3, but her data is still sitting in Database 0, so the lookup will fail entirely.

> **Observation from the video:** scaling from 3 servers to 4 servers required moving the majority of existing keys to different partitions, even though only one new server was added. Now imagine this isn't 5 users, but 5 million. Adding just one new server could require physically moving around 3 million keys to new locations. The same exact problem happens in reverse when a server is removed: most of the remaining keys need to be reshuffled across the servers that are left.

**This is the core problem that consistent hashing exists to solve:** with a plain modulus-based hash function, adding or removing even a single server forces a huge, disruptive amount of data movement across the system.

---

## 3. Consistent Hashing: The Ring (Clock) Mechanism

The basic idea behind consistent hashing is to imagine a circular ring of numbers, like a clock face, instead of a simple modulus formula.

> **Definition:** Consistent hashing arranges a fixed range of hash values (say, 1 through 12, just like the numbers on a clock) into a circular ring. Both servers and data are placed onto specific points on this ring, based on a hash function. To find which server a piece of data belongs to, you start at the data's position on the ring and move **clockwise** until you reach the first server you find. That server is responsible for storing that data.

**Step 1: Place your servers on the ring.**

```
Imagine a clock face numbered 1 to 12.

Hash(Server A's IP) = 9   -> place Server A at position 9
Hash(Server B's IP) = 2   -> place Server B at position 2
Hash(Server C's IP) = 5   -> place Server C at position 5
```

**Step 2: Place your data on the same ring, and walk clockwise to find its server.**

```
Hash(some data) = 1
   -> starting at position 1, walk clockwise...
   -> the first server found is Server B (at position 2)
   -> this data is stored on Server B

Hash(some other data) = 6
   -> starting at position 6, walk clockwise...
   -> the first server found is Server A (at position 9)
   -> this data is stored on Server A

Hash(another piece of data) = 10
   -> starting at position 10, walk clockwise...
   -> the first server found (wrapping around past 12, back to 1) is Server B (at position 2)
   -> this data is stored on Server B
```

This is exactly how data gets distributed around the ring: hash it, find its position, then walk clockwise until a server is found, and store the data there.

---

## 4. Why This Solves the Rehashing Problem

Now let's see what happens when a new server is added to this ring-based system.

> **Example:** Suppose Server A is at position 9, and you deploy a new Server D, which hashes to position 12. Before Server D existed, any data hashing to a position between 9 and 12 (say, position 10) would have walked clockwise and landed on... whichever server came next after 12, wrapping around the ring. Now that Server D sits at position 12, all data that hashes to somewhere between 9 (exclusive) and 12 (inclusive) should now belong to Server D instead.

```
Before adding Server D:
   Server A at 9 -----------------------> (next server further along the ring)
                  positions 10, 11, 12 all belonged to whichever server came next

After adding Server D (at position 12):
   Server A at 9 --------> Server D at 12 -----> (next server further along the ring)
                  positions 10, 11, 12 now belong to Server D
```

**The key insight: only the keys that fall between the previous server (position 9) and the newly added server (position 12) need to move.** Every other key, anywhere else on the ring, is completely unaffected and stays exactly where it was.

```
To find affected keys when a server is added:
   Start at the newly added server's position
   Walk counter-clockwise around the ring
   Stop as soon as you reach the previous server
   Only the keys found in that range need to be moved
```

The same logic applies in reverse when a server is **removed**: only the keys that were stored on the removed server need to move, and they simply move to the very next server found by walking clockwise from the removed server's old position. Every other key on the ring is unaffected.

### Benefits of Consistent Hashing

- **Minimal data movement.** Only a small fraction of keys need to be redistributed when a server is added or removed, unlike the plain modulus approach, where adding or removing a single server could reshuffle the majority of all keys.
- **Easy to compute which keys are affected.** You only need to look at the range between the changed server's new position and its nearest neighbor on the ring.
- **Easy to scale horizontally**, since data remains fairly evenly distributed as servers are added or removed.

---

## 5. A Remaining Problem: Uneven Distribution and Hotspots

Consistent hashing isn't perfect on its own. If the hash function happens to place two servers very close together on the ring, by pure chance, the distribution of data across servers can become badly uneven.

> **Example:** Suppose two servers end up hashed to positions that are very close to each other on the ring. The server that comes right after this "close pair," going clockwise, will end up responsible for an unfairly large stretch of the ring, and therefore for a disproportionately large share of the data, while the two nearby servers only handle a tiny sliver each.

```
Ring positions (not to scale):

  ...-----[Server X]--[Server Y]---------------------------[Server Z]-----...
            (very close together)         (huge gap: lots of data lands here)
```

This creates a **hotspot**: one server ends up overloaded simply because of where it happened to land on the ring, not because of anything about the actual data.

### The Fix: Virtual Nodes

> **Definition:** A virtual node is an additional marker placed on the hash ring that points back to a real, physical server. Instead of representing each physical server with just a single point on the ring, each server is represented by **multiple** virtual nodes, scattered at different positions around the ring.

```
Physical Server X is represented on the ring by multiple virtual nodes:
   Virtual Node at position 10  -----> points to Server X
   Virtual Node at position 45  -----> points to Server X
   Virtual Node at position 78  -----> points to Server X
```

Since each real server now has several scattered positions on the ring instead of just one, the overall distribution of data becomes far more even and much less sensitive to any single unlucky hash collision. If any one of Server X's virtual node positions happens to be unlucky (too close to a neighbor), its other virtual node positions elsewhere on the ring help balance things back out.

---

## 6. Real-World Usage

Consistent hashing is a widely used technique in real-world, large-scale distributed systems. The video specifically mentions:

- **Amazon DynamoDB**
- **Apache Cassandra**
- **Discord's chat application infrastructure**

> **A related problem worth knowing about (mentioned as a topic for a future video):** the "celebrity problem," also known as a hotspot problem in a different form, where a large volume of data or traffic all belongs to one especially popular entity (like a celebrity user), and ends up concentrated on a single partition no matter how well the hashing is designed, simply because that one entity's data is disproportionately accessed.

---

## Summary: Key Concepts in Consistent Hashing

| Concept | Purpose |
|---|---|
| Hash Function | A deterministic, one-way function that decides which partition a piece of data belongs to |
| Modulus Hashing (`id % servers`) | A simple hashing approach that works, but breaks badly when the number of servers changes |
| Rehashing Problem | Adding or removing even one server can force the majority of existing keys to move, under plain modulus hashing |
| Hash Ring | A circular arrangement of hash values, used to place both servers and data consistently |
| Clockwise Lookup | The rule used to find which server a piece of data belongs to: walk clockwise from the data's hash position until a server is found |
| Minimal Key Movement | Adding or removing a server under consistent hashing only affects keys between that server and its nearest neighbor on the ring |
| Hotspot Problem | Uneven placement of servers on the ring can overload one server with a disproportionate share of data |
| Virtual Nodes | Representing each physical server with multiple scattered points on the ring, to even out data distribution and reduce hotspots |

**Key takeaway:** consistent hashing solves a very concrete, practical problem: how to add or remove servers in a distributed system without having to move almost all of your existing data around every single time. It does this by mapping both servers and data onto a shared circular ring and using a simple clockwise lookup rule, with virtual nodes added on top to keep the load evenly balanced across all real servers.
