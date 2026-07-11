# System Design Behind Multi-Conference Video Calls - WebRTC vs SFU vs MCU

Source video: [System Design Behind Multi-Conference Video Calls - WebRTC vs SFU vs MCU](https://www.youtube.com/watch?v=Zaz6hYVm-WE) (Hindi, about 24 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow.

---

## 1. What Is a Multi-Conference Application?

Think of apps like Zoom or Google Meet, where you can have a video call with more than just one other person.

> **Definition:** "Multi-conference" simply means a video call involving more than two participants at once. Each participant in such a call is referred to as a **peer**.

Designing a multi-conference application is genuinely difficult for two reasons: it's a real-time application (audio and video have to be transmitted with minimal delay), and there are several different architectural patterns you can choose from, each with its own trade-offs. This video walks through those patterns, from simplest to most production-ready.

---

## 2. The Simplest Case: WebRTC Peer-to-Peer (Just Two Peers)

> **Definition:** WebRTC (Web Real-Time Communication) enables direct **peer-to-peer (P2P)** communication between exactly two participants, without any server sitting in between. It works over the **UDP** protocol (User Datagram Protocol), chosen because it's fast and supports this kind of direct connection well.

```
Peer 1  <-------- direct peer-to-peer connection -------->  Peer 2
        (audio + video streams sent directly, no server involved)
```

Since there's no server involved at all, this kind of connection is essentially free, you only pay for your own internet bandwidth. This is the simplest and cheapest possible setup, but it only works for exactly **two** participants.

---

## 3. Trying to Extend This: Mesh Peer-to-Peer

What if a third peer wants to join? Since WebRTC only supports direct peer-to-peer connections, one workaround is to have **every peer connect directly to every other peer**. This is called a **mesh** topology.

> **Definition:** In a mesh peer-to-peer architecture, every single peer establishes a direct WebRTC connection with every other peer in the call.

```
3 peers:
Peer 1 <---> Peer 2
Peer 1 <---> Peer 3
Peer 2 <---> Peer 3

(3 total direct connections)
```

This technically works, since WebRTC does allow direct peer-to-peer connections, and each pair here really is just a two-person connection. But watch what happens as more peers join.

**The problem: this doesn't scale.** Every time a new peer joins, that new peer must establish a direct connection with **every existing peer**, and every existing peer must also establish a connection back with the new one.

```
4 peers = 6 total connections
5 peers = 10 total connections
12 peers = 66 total connections
```

> **Why this breaks down:** Each individual peer ends up needing to maintain a separate direct connection with every other participant. If there are 12 people in a call, each single peer's device has to manage 11 simultaneous peer-to-peer connections, sending and receiving audio/video streams over every single one. This puts a huge load on each individual device, is very difficult to debug, and tends to crash frequently as the number of participants grows. Mesh peer-to-peer is fine for a handful of people, but it is not a scalable or fault-tolerant architecture for real multi-conference applications like Zoom or Google Meet.

---

## 4. Introducing a Server: The Forwarding Unit Concept

The industry quickly realized that peer-to-peer alone works fine for two people, but doesn't scale for many people. The fix: introduce a **server** into the middle of the conversation, acting as a **central forwarding unit**.

> **Key insight:** WebRTC's rule ("only direct peer-to-peer connections") is still respected here. It's just that now, every peer forms a peer-to-peer connection with **the server**, instead of with every other peer directly.

```
Peer 1 <---> Server
Peer 2 <---> Server
Peer 3 <---> Server
Peer 4 <---> Server
Peer 5 <---> Server

(Each peer makes exactly ONE connection, to the server)
```

This is a huge improvement: no matter how many peers join, **each individual peer only ever needs to maintain a single connection**, to the server. The server itself is built to handle many simultaneous connections. There are two major ways this central server can actually behave once it has everyone's streams: **MCU** and **SFU**.

---

## 5. Option A: MCU (Multipoint Control Unit)

> **Definition:** An MCU (Multipoint Control Unit) is a communication server that receives audio and video streams from every peer in a call, **combines (mixes) them together into a single, unified stream**, and sends that one combined stream back out to everyone.

```
Peer 1 --stream--> \
Peer 2 --stream--> |
Peer 3 --stream-->  }---> MCU Server ---> combines all streams into ONE ---> sent back to every peer
Peer 4 --stream--> |
Peer 5 --stream--> /
```

> **Example:** Think of a podcast or live-streamed group discussion (the video mentions the creator's own past collaborative livestreams with other creators) that ends up as a single, combined video on YouTube, where you can't separate out any one person's feed. That combined output is exactly what an MCU produces: one single mixed stream containing everyone's audio and video together.

**The problem with MCU:** combining multiple live audio and video streams into a single output stream, in real time, is an extremely **CPU-intensive** task. This introduces:

- **Noticeable lag**, since mixing multiple real-time streams on the fly takes real processing time.
- **High server cost**, since this kind of continuous, real-time stream mixing requires significant CPU resources, and cost scales up quickly as more participants and more calls are added.

---

## 6. Option B: SFU (Selective Forwarding Unit) — The Better Solution

> **Definition:** An SFU (Selective Forwarding Unit) is a communication server that also receives audio and video streams from every peer, but instead of mixing them into a single combined stream, it forwards each peer's stream **separately** to every other peer, without any combining or processing.

```
Peer 1 --stream--> \
Peer 2 --stream--> |
Peer 3 --stream-->  }---> SFU Server (no mixing, just forwards)
Peer 4 --stream--> /

Peer 1 receives: Peer 2's stream, Peer 3's stream, Peer 4's stream (separately)
Peer 2 receives: Peer 1's stream, Peer 3's stream, Peer 4's stream (separately)
   ...and so on
```

Every peer sends exactly one stream to the server, and in return, receives everyone else's streams as **separate, individual streams**, not merged into one.

### Why SFU Is Better Than MCU

**1. It's far less CPU-intensive.** The server here acts like a simple pass-through, a "tunnel" or "pipe." It isn't doing any real processing or mixing, just receiving a stream and routing it back out to the right recipients. This is dramatically cheaper and faster than MCU's real-time stream-mixing approach.

**2. The client gets control over rendering.** Since each participant's stream arrives separately (instead of pre-mixed into one), the receiving client's app can decide, on its own, how to display or handle each individual stream.

> **Example:** Suppose Peer 1 doesn't want to hear or see Peer 3 anymore. With an SFU, Peer 1's app can simply choose to stop consuming (ignore) Peer 3's specific stream, effectively muting just that one person, locally, for themselves only. This is impossible with MCU, since by the time the stream reaches Peer 1, all the other peers' audio and video are already permanently mixed into one combined stream, there's no way to separate out and mute just one person's contribution afterward (except by muting them for everyone, at the server level).

> **Another example:** If you've ever pinned, enlarged, or resized a specific person's video tile in Google Meet just for your own view, that's only possible because you're receiving that person's video as its own separate stream, exactly what an SFU provides.

**3. Selective forwarding across multiple rooms.** The "selective" part of the name also refers to the server's ability to decide **which streams should go to which peers** in the first place. For example, if you're in one Google Meet room and someone else is in a completely separate meeting, none of your meeting's content should ever reach theirs. An SFU server keeps track of which peers belong to which call/room, and only forwards the appropriate streams to the appropriate participants.

**This is exactly the architecture used by production applications like Google Meet and Zoom.**

---

## 7. Building Your Own SFU: mediasoup

If you wanted to implement your own SFU, there's a well-known open source library for this called **mediasoup**.

> **Definition:** mediasoup is a low-level Selective Forwarding Unit library that can be integrated into a Node.js server, giving you the building blocks to implement your own SFU-based video conferencing backend.

The video's creator shares some personal history here: they've worked with mediasoup on and off for the past five to six years, including contributing to its open source repository (and recalls once getting a rather blunt response from a maintainer after filing an incomplete issue, a reminder that open source contributions need to include proper detail and effort). Despite that, mediasoup is described as a genuinely well-designed library, with a well thought out architecture built around concepts like **workers**, **transports**, and **routers** that handle the actual stream routing internally. The creator mentions they were revisiting mediasoup recently and considered making a dedicated, in-depth video specifically about how it works, if there's enough interest.

---

## Summary: Comparing the Architectures

| Architecture | How It Works | Scales to Many Peers? | Main Trade-off |
|---|---|---|---|
| WebRTC Peer-to-Peer | Two peers connect and exchange streams directly, no server | No (only 2 participants) | Free and simple, but limited to exactly two people |
| Mesh Peer-to-Peer | Every peer connects directly to every other peer | Poorly (connections grow quadratically) | Works for a handful of people, but becomes unmanageable and crash-prone as participants grow |
| MCU (Multipoint Control Unit) | A central server combines all streams into a single mixed stream and broadcasts it | Yes, but at high server cost | Very CPU-intensive; introduces lag; clients can't control individual streams |
| SFU (Selective Forwarding Unit) | A central server forwards each peer's stream separately to every other peer, without mixing | Yes, efficiently | Low CPU cost (just a pass-through); clients get full control over rendering/muting/pinning individual streams |

**Key takeaway:** modern multi-conference video applications like Zoom and Google Meet are built on the SFU architecture, since it avoids the heavy, real-time CPU cost of mixing streams (MCU's core weakness), while still letting a single central server handle any number of participants (something plain peer-to-peer or mesh networking cannot do). The trade-off SFU makes is pushing the responsibility of "what to do with each incoming stream" onto the client, which turns out to be both cheaper for the server and more flexible for the user.
