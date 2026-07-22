# System Design Notes

A set of study notes I've been keeping while working through Piyush Garg's system design videos on YouTube. The videos are in Hindi; these notes are written out in plain English, with the definitions, examples, and ASCII diagrams filled in so each topic stands on its own without needing to rewatch the video.

Every file lists the source video at the top, along with its length, so you can jump back to the original whenever you want the full explanation.

## Why these exist

Watching a video and understanding it are two different things. Writing the ideas down in my own words — and drawing out the request flows — is what actually makes them stick. I'm sharing them in case they're useful to anyone else preparing for interviews or just trying to get a feel for how large systems are put together.

The notes aren't a transcript. Where the video moved fast or skipped a detail, I've cross-checked the claim and added the missing context, so a few sections go beyond what's said on screen.

## Contents

### Getting started

| Topic | What it covers |
|-------|----------------|
| [Crash Course — Part 1](PiyushGarg/CrashCoursePart1.md) | Clients and servers, scaling, load balancing, databases, caching — the beginner walkthrough. |
| [Crash Course — Part 2](PiyushGarg/CrashCoursePart2.md) | The continuation: sharding, replication, message queues, and more of the core building blocks. |
| [Back-of-the-Envelope Calculation](PiyushGarg/BackofEnvelopeCalculation.md) | Estimating traffic, storage, and bandwidth so your design targets realistic numbers. |

### Scaling and infrastructure

| Topic | What it covers |
|-------|----------------|
| [Consistent Hashing](PiyushGarg/ConsistentHashing.md) | Distributing keys across nodes so adding or removing a server doesn't reshuffle everything. |
| [Bloom Filters](PiyushGarg/BloomFilters.md) | A space-efficient way to answer "have I seen this before?" — with username availability as the example. |
| [Rate Limiting](PiyushGarg/RateLimiting.md) | The common algorithms — token bucket, leaky bucket, fixed and sliding windows — and when each fits. |
| [Queues](PiyushGarg/Queues.md) | Message queues for decoupling services and smoothing out load spikes. |
| [Gossip Protocol](PiyushGarg/GossipProtocol.md) | How nodes in a cluster share state and detect failures without a central coordinator. |

### Architectural patterns

| Topic | What it covers |
|-------|----------------|
| [Important System Design Patterns](PiyushGarg/ImportantSystemDesignPatterns.md) | A tour of patterns worth knowing before an interview. |
| [Event Sourcing](PiyushGarg/EventSourcing.md) | Storing state as a log of events instead of overwriting rows in place. |
| [CQRS](PiyushGarg/CQRS.md) | Splitting reads from writes. Builds on the Event Sourcing notes, so read that one first. |

### Real-world case studies

| Topic | What it covers |
|-------|----------------|
| [UPI Payments](PiyushGarg/UPIPayments.md) | The architecture behind India's real-time payments system. |
| [Video Streaming at Scale](PiyushGarg/VideoStreaming.md) | Encoding, adaptive bitrate, and CDNs — how streaming platforms deliver video to millions. |
| [Multi-Conference Video Calls](PiyushGarg/Multi-ConferenceVideoCalls.md) | WebRTC vs. SFU vs. MCU, and the trade-offs between them. |

## How to read them

The notes are roughly ordered from fundamentals to specialized topics. If you're new to the subject, start with the two crash-course parts and the back-of-the-envelope note, then pick topics as they interest you. A few notes reference each other — CQRS builds on Event Sourcing, and Rate Limiting expands on a section from Crash Course Part 1 — and those links are called out inline.

Everything is plain Markdown, so it renders fine on GitHub or in any editor.

## Credit

All credit for the underlying explanations goes to [Piyush Garg](https://www.youtube.com/@piyushgargdev). These notes are my own write-up of his videos and aren't affiliated with or endorsed by him.
