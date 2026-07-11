# What are Bloom Filters? | System Design

Source video: [What are Bloom Filters? | System Design](https://www.youtube.com/watch?v=vz0QUa4CS3o) (Hindi, about 21 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow.

---

## 1. The Problem: Checking Username Availability at Scale

Imagine you're designing a platform like Twitter. The very first feature you'd build is sign-up, and on Twitter, signing up requires choosing a unique **username**.

```
User enters a username, e.g. "piyush"
   |
   v
Client calls the backend API: "Is the username 'piyush' available?"
   |
   v
Backend checks the database
```

**The naive approach:** query the users table directly.

```sql
SELECT * FROM users WHERE username = 'piyush' LIMIT 1;
```

If this query returns a result, the username is already taken. If it returns nothing, the username is available. This works fine at a small scale, but consider what happens as the platform grows.

> **Example of the problem at scale:** Suppose the platform has 20 million users. Every time someone types a username, the system has to search through 20 million records to check if it exists, even with indexes in place, this is still a heavy, costly operation at that scale. Worse, popular usernames are very often already taken, so a new user might try "piyush," find it taken, then try "piyush_gaming," find that taken too, then try a few more variations before finally finding one that's free. On average, a single signing-up user might end up triggering **two to three** of these expensive 20-million-record searches, just to land on one available username. Multiply that by many users signing up at once, and this single feature becomes a major bottleneck on the database.

---

## 2. First Attempt at a Fix: A Hash Map

A natural first idea is to avoid hitting the database directly, and instead keep an in-memory hash map of every username that's already taken.

```
HashMap:
   "piyush" -> true
   "john"   -> true
   ...
```

Whenever a new username is registered, you simply add it to the hash map with a value of `true`. To check availability, you just look it up in the hash map.

> **Benefit:** A hash map lookup takes O(1) time (constant time), regardless of how many usernames exist, which is far faster than scanning millions of database rows.

**The catch:** this trades lookup speed for **space**. If a platform is adding even a modest 1,000 new users every day, that hash map keeps growing forever, and storing every single username that has ever been registered, in memory, becomes extremely space-inefficient at scale.

```
Database scan approach:  lookup time O(N)   |  space: minimal extra space
Hash map approach:       lookup time O(1)   |  space: grows without bound
```

**What's needed is something that strikes a balance between these two extremes:** fast lookups, without needing to store the full, ever-growing set of usernames in memory. This is exactly the problem that a **Bloom filter** solves.

---

## 3. What Is a Bloom Filter?

> **Definition:** A Bloom filter is a space-efficient, probabilistic data structure that uses a bit array and multiple hash functions to test whether an element is possibly a member of a set, or definitely not a member of it.

The key idea behind a Bloom filter involves a deliberate trade-off, and understanding this trade-off is essential before using one.

### The Core Trade-off: False Positives

A Bloom filter never gives you a simple "yes" or "no" the way a database or a hash map does. Instead, it gives you one of two answers:

```
"Definitely NOT in the set"        -> you can trust this completely
"Possibly in the set" (maybe)      -> this could be wrong
```

> **The critical guarantee:** if something genuinely **is** in the set (e.g., a username that really has been registered), a Bloom filter will **always** correctly say it's taken. It will never incorrectly report a truly-registered username as available. This is essential, since incorrectly telling a user their taken username is free would be a serious bug.
>
> However, if something is **not** actually in the set, a Bloom filter can sometimes still say "yes, this looks taken," even though it's actually still free. This incorrect "yes" is called a **false positive**.

> **Example:** Suppose a username like "Apple" was genuinely never registered by anyone. A Bloom filter might still respond "this is taken" for "Apple," purely due to how its internal hashing works (explained in detail below). The practical consequence is just that this particular username becomes permanently unavailable to anyone, even though no one ever actually took it. That's the cost of using a Bloom filter: an acceptable trade-off in systems that can tolerate it, since the alternative (scanning millions of rows on every keystroke) is far more expensive.

If your system can tolerate occasionally, and incorrectly, telling a user that a genuinely free username is unavailable, then a Bloom filter is a great fit. If it can't, a Bloom filter isn't the right tool.

---

## 4. How a Bloom Filter Actually Works

A Bloom filter is built from two main components: a **hash function**, and a set of **buckets** (a bit array).

> **Definition:** A hash function takes some input (in this case, a username) and returns some form of numeric output, its hash. For a Bloom filter, this hash is used to point to one or more positions in a bit array.

> **Definition:** Buckets refer to a bit array: a simple sequence of bits, each of which starts at 0, and gets flipped to 1 as usernames are inserted.

**Example setup:** let's use a small bit array of 10 buckets, indexed 0 through 9, all starting at 0.

```
Index:   0  1  2  3  4  5  6  7  8  9
Bits:    0  0  0  0  0  0  0  0  0  0
```

### Inserting "piyush"

Suppose hashing "piyush" returns the indices **0, 2, 4, 8**. To insert it, you set each of those bit positions to 1.

```
Index:   0  1  2  3  4  5  6  7  8  9
Bits:    1  0  1  0  1  0  0  0  1  0
```

### Checking if "piyush" is available

Hash "piyush" again. Since hashing is deterministic, it returns the same indices: 0, 2, 4, 8. Check each of these positions:

```
Index 0: is it 1? Yes.
Index 2: is it 1? Yes.
Index 4: is it 1? Yes.
Index 8: is it 1? Yes.

All positions are 1, so "piyush" is DEFINITELY taken.
```

### Inserting "john"

Suppose hashing "john" returns indices **1, 4, 6, 8**. Set those bits to 1 (index 4 and 8 were already 1 from "piyush", that's fine, they simply stay 1).

```
Index:   0  1  2  3  4  5  6  7  8  9
Bits:    1  1  1  0  1  0  1  0  1  0
```

### Checking a brand-new username: "lemon"

Suppose hashing "lemon" returns indices **1, 2, 4, 7**.

```
Index 1: is it 1? Yes.
Index 2: is it 1? Yes.
Index 4: is it 1? Yes.
Index 7: is it 1? No! It's still 0.

Since at least one required position is 0, "lemon" is DEFINITELY available.
```

Because even one of "lemon's" positions (index 7) was never set by anything, we know for certain that "lemon" was never actually inserted. This is how a Bloom filter can confidently say something is available: if any single required bit is still 0, the item cannot possibly have been inserted before.

### The False Positive in Action: "Apple"

Now suppose hashing "Apple" (which was never actually registered by anyone) returns indices **1, 2, 6, 8**.

```
Index 1: is it 1? Yes (set by "john").
Index 2: is it 1? Yes (set by "piyush").
Index 6: is it 1? Yes (set by "john").
Index 8: is it 1? Yes (set by both "piyush" and "john").

All positions happen to already be 1, so the Bloom filter reports:
"Apple" is TAKEN.
```

But "Apple" was never actually inserted by anyone. It just so happened that every single bit position its hash pointed to had already been set to 1 by the combination of "piyush" and "john." **This is a false positive**: the Bloom filter incorrectly reports an available username as taken, purely due to overlapping bit positions from other, unrelated entries.

---

## 5. Reducing False Positives: Bigger Buckets, Better Hashing

At this point, it might seem like Bloom filters are a bad idea. With only 10 buckets, it wouldn't take many users before every single bit is set to 1, at which point literally every possible username would incorrectly show up as "taken."

**The fix is straightforward: use a much larger bit array.**

> **Example:** Instead of 10 buckets, use 1,000 buckets. Now, a hash function returns a number somewhere between 0 and 999, spreading entries out far more widely. With more possible positions available, the odds of two different usernames' hashes overlapping on all their positions (a collision) drop significantly. This means far fewer false positives, and the ability to comfortably store far more usernames before the bit array starts filling up.

As the number of users on your platform grows, you can proportionally increase your bucket count (say, to 5,000 or more), and use a good hash function that spreads values widely and evenly across the array. Both of these reduce the collision rate, and therefore reduce how often false positives occur.

**Using multiple hash functions (or multiple "sub-filters") further reduces collisions.** Instead of relying on a single hash function, you can use two (or more) separate hash functions, and only treat an item as "possibly taken" if **all** of them agree that its corresponding bits are set. This makes accidental overlaps (like the "Apple" example above) considerably less likely.

---

## 6. Real-World Uses of Bloom Filters

Bloom filters are used well beyond just checking username availability. A few real-world examples mentioned in the video:

- **Google Chrome** uses a Bloom filter to quickly check whether a URL might be malicious or has been flagged as unsafe, before doing a more expensive, definitive check.
- **Spam filters** use Bloom filters to quickly flag whether an incoming email might be spam.
- **Email availability checks** (for example, on Gmail, when you're choosing a new email address) work in a very similar way to the username example above.

---

## 7. When Should You Actually Use a Bloom Filter?

> **Important guidance from the video:** Bloom filters are meant for **high-scale systems**, like Twitter or LinkedIn, where checking something as simple as username availability would otherwise place heavy, repeated load on the main database. If your system only has a modest number of users, your database is more than capable of handling a direct search itself, and you don't need the added complexity (and the false-positive trade-off) of a Bloom filter at all.

---

## Summary: Key Concepts in Bloom Filters

| Concept | Purpose |
|---|---|
| The Problem | Checking uniqueness (e.g., username availability) against millions of records is a heavy, repeated database load at scale |
| Hash Map Approach | Fast O(1) lookups, but space usage grows without bound as more items are added |
| Bloom Filter | A space-efficient, probabilistic alternative: a bit array plus one or more hash functions |
| "Definitely Not" | If any required bit is 0, the item was definitely never inserted |
| "Possibly Yes" (False Positive) | If all required bits happen to be 1, the item is reported as taken, even if it was never actually inserted |
| Bucket (Bit Array) Size | A larger bit array spreads hash values out more, reducing collisions and false positives |
| Multiple Hash Functions | Using more than one hash function (checking that all agree) further reduces false positives |
| When to Use | Only in high-scale systems where a false positive is an acceptable trade-off for avoiding expensive, repeated full-scale lookups |

**Key takeaway:** a Bloom filter trades perfect accuracy for space efficiency and speed. It can tell you with full certainty that something is *not* in a set, but it can only tell you that something is *probably* in a set, occasionally being wrong in a specific, predictable direction (false positives, never false negatives). This makes it an excellent fit for high-scale existence checks, like username or email availability, where an occasional overly cautious "this is taken" is a perfectly acceptable cost for avoiding constant, expensive database scans.
