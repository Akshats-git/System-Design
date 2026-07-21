# System Design Crash Course - Part 2 (Beginner Friendly)

Source video: [System Design Crash Course - Part 2](https://www.youtube.com/watch?v=YuB3OuF3MUE) (Hindi, about 50 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow. This part builds on [CrashCoursePart1.md](CrashCoursePart1.md), which covered the basics: client-server, DNS, scaling, load balancers, microservices, API gateways, queues, pub/sub, rate limiting, caching, and CDNs.

---

## 1. System Design Is Not One-Size-Fits-All

Before going further, there is an important idea to understand. System design is not something you can learn once and then apply in exactly the same way everywhere. Every company has its own use case, its own unique selling point, its own working pattern, and its own traffic pattern. This is why system design keeps evolving over time for every company, rather than being a fixed recipe.

The main goals of system design are usually to make a system fully scalable, fault tolerant, and cost optimized at the same time. Scaling something is easy if you simply throw a huge amount of resources at it. The real challenge is making sure the system does not crash under load, while also not wasting money on resources that are rarely used. To strike that balance, every company first needs to deeply understand its own traffic pattern.

> **Example:** YouTube, Netflix, and Hotstar (JioHotstar) are all, at the end of the day, video streaming platforms. But you cannot take Netflix's system design and apply it directly to YouTube, or take YouTube's system design and apply it to Hotstar. Each of them would crash, or become extremely expensive to run, if it used another platform's exact architecture. This is not because one of these designs is bad. It is because each platform has a fundamentally different traffic pattern.

---

## 2. Case Study: Netflix (Predictable Traffic)

Netflix's traffic is comparatively easier to manage, because most of it is **predictable**.

Imagine Netflix has, say, 3 million active users on an average day, and its system is scaled to comfortably handle that load.

**The problem with sudden spikes:** If traffic increases very suddenly (a "burst"), most systems rely on auto-scaling policies, such as "add a new server if requests in the last 15 minutes go above 1000" or "add a new server if average CPU usage crosses 70%." Scaling always happens based on some policy, and it is never magical. It has to be told when to react.

If traffic grows slowly and gradually, auto-scaling works fine, because the system has time to react and add servers step by step as the average load rises. But if traffic suddenly spikes far faster than the average can catch up (for example, a 10x jump in seconds), auto-scaling based on average CPU usage reacts too slowly, and the system can crash before enough new servers are added.

```
Slow, gradual traffic growth:
  3 servers -> 4 servers -> 5 servers -> 6 servers -> 7 servers -> 8 servers
  (Auto-scaling keeps up easily because the average has time to update)

Sudden spike:
  3 servers -> (traffic jumps 10x almost instantly) -> system crashes
  (Auto-scaling based on averages reacts too slowly)
```

**Why Netflix can avoid this problem:** Netflix looks at its own historical data. A traffic spike on Netflix almost always happens for one specific reason: a new movie or show is being released. Movie releases are never random. They are planned weeks in advance, with trailers released ahead of time. So Netflix already knows the exact date and time a spike is likely to happen.

> **Example:** If Netflix knows a new movie will launch in 4 days, it can pre-warm its servers ahead of time. If 10 servers are normally running, Netflix can scale up to 30 servers, say 24 hours in advance, based on statistics from previous launches. If the predicted spike happens, the system easily absorbs it. If the spike does not happen (say the movie is less popular than expected), Netflix simply scales back down to 10 servers afterward. Either way, nothing crashes.

Netflix can also pre-cache the first 10 minutes of a new release across all of its CDN edge servers worldwide, before the movie is even officially launched. This further reduces load on the origin servers once the movie goes live.

**Key takeaway:** Because Netflix controls exactly when new content is published, it can predict spikes and pre-scale for them. This makes handling Netflix's traffic pattern comparatively easier.

---

## 3. Case Study: YouTube (Unpredictable Traffic)

Now imagine applying Netflix's exact system design to YouTube. It would not work, because YouTube's traffic pattern is fundamentally different, and much harder to predict.

On YouTube, publishing is not controlled by YouTube itself. It is controlled by millions of independent creators and users. A large creator could go live at any random moment, with no warning to YouTube in advance. A sudden real-world event (for example, breaking political news) can cause many news channels to go live all at once. In both cases, traffic spikes suddenly and without warning.

> **Example:** Suppose YouTube's average traffic is around 10 million concurrent users. If a very large creator suddenly starts a livestream, or a major unpredictable news event breaks, traffic can spike far above that average almost instantly, with no advance notice at all.

Some traffic prediction is still possible on YouTube using machine learning and historical patterns. For example, traffic tends to rise during exam seasons (February and March, when students consume more educational content), and tends to dip during major festivals like Diwali in India, when people are less engaged with online content. This kind of month-level prediction is possible. However, sudden, unplanned spikes cannot be predicted this way.

Because of this, YouTube's system design has to be advanced enough to handle sudden, unpredictable spikes gracefully, generally by keeping some servers pre-warmed at all times and relying on extremely fast auto-scaling, even though this comes at an extra cost.

**Key takeaway:** YouTube's traffic pattern is largely unpredictable because publishing is controlled by anyone in the world, not by YouTube itself. This forces YouTube's system design to prioritize handling sudden spikes over pure cost efficiency.

---

## 4. Case Study: Hotstar (Mixed Predictability, and a Subtle Trap)

Hotstar sits somewhere in between. Like Netflix, it streams movies and web series, so that part of its traffic is somewhat predictable. But Hotstar also does **live streaming**, most notably of cricket matches, and this creates a more interesting and subtle problem.

Hotstar effectively runs (at least) two separate services:

- A **movies and web series service**, which shows the catalog you see when you log in.
- A **live streaming service**, used for live sports and events.

On a normal day with no live match happening, the live streaming service can be scaled down heavily, since most people are just watching movies or shows. But if Hotstar knows in advance that a big match, say India versus a major rival, is scheduled for tomorrow, it can predict a large traffic spike and pre-scale specifically for that.

> **Example:** Suppose the live streaming service normally runs on just 10 servers. A few hours before a big match, Hotstar can scale it up to 200 servers, while scaling the movies service down, since very few people watch movies during a live match.

**The traffic pattern during a live cricket match looks something like this:**

```
Traffic over time during a match:

Low traffic
   |
   |  Toss happens        -> sudden spike (everyone tunes in)
   |  Waiting for play    -> traffic dips a little
   |  Boring over bowled   -> slight rise, then dips again
   |  Big wicket falls     -> sudden spike (everyone reacts)
   |  Star batsman comes in -> sudden spike
   |  Star batsman out early -> another spike, then people get upset and leave
   v
```

Because these in-match spikes and dips happen so quickly and unpredictably (a big wicket, a star player batting, and so on), **auto-scaling based on average usage is dangerous here**. If traffic dips and auto-scaling scales the servers down, and then traffic spikes again moments later, the system can crash before it scales back up. The safer strategy during a known live event is to disable auto-scaling entirely, and simply keep the system scaled up (say, at 200 servers) for the full duration of the match, even if that costs more. Paying for a few extra hours of overprovisioned servers is far cheaper than an outage during a major live event.

**Now here is the subtle trap.** Even if the live streaming service itself is scaled well enough to handle, say, 240 million people watching, there is another risk. When a match gets boring, or a viewer's favorite player gets out, what do many people do? They press the **back button** on their screen. Pressing back takes them from the live stream back to the home screen, which lists all available movies and shows. Loading that home screen triggers calls to the **movies API**, not the live streaming API.

```
User watching live stream
   |
   |  presses "back" button
   v
Home screen (movies catalog)
   |
   |  calls Movies API to load the catalog
   v
Movies API receives a sudden spike in traffic
```

So, every dip in live-stream engagement (a boring moment, a wicket falling) can translate into a **spike on the completely different movies service**, simply because frustrated or bored viewers go "back" to browse something else. If 240 million people are watching a live match and a large fraction of them press back at the same moment, the movies API can suddenly be flooded with traffic it was never scaled for, even though the live streaming service itself is working perfectly fine.

**Key takeaway:** This is exactly why system design is never something you get perfectly right on the first attempt. You start, you monitor real traffic, you learn from actual crashes and incidents, and then you optimize for the next time. Hotstar likely only learned about this movies-API spike after experiencing it in a previous live event, and used that learning to prepare better for future ones.

---

## 5. Case Study: Amazon (Partially Predictable)

Amazon's traffic is also partly predictable. Amazon knows in advance about planned sale events (such as "Big Billion Days"), so it can pre-scale its systems ahead of the sale's start time, in a similar way to how Netflix pre-scales for a movie release.

The genuinely hard case, as discussed above, remains platforms like YouTube, where anyone can trigger a spike at any moment, and no advance forecasting is possible.

---

## 6. The Overhead of Managing Traditional (Server-ful) Machines

Stepping back from these traffic-pattern examples, there is a deeper, more general problem worth understanding: managing traditional physical or virtual machines comes with a lot of operational overhead.

When you scale out using traditional servers, you have to think about many small details every time: how much CPU and RAM to assign, how to allocate an IP address, and how to register the new machine with the load balancer. This is a genuinely difficult and time-consuming process to repeat every single time you scale.

> **Question worth asking:** If you are a software developer or a startup founder, do you actually want to spend your time thinking about CPU allocation and IP addresses, or would you rather focus purely on building your application?

This exact problem is what led cloud providers like AWS to create a new kind of service: **serverless computing**, starting with AWS Lambda.

---

## 7. Serverless Computing (AWS Lambda)

> **Definition:** Serverless does not mean there is no server involved. It means that you, as a developer, do not manage the server at all. The cloud provider manages the underlying machine, operating system, CPU, RAM, and scaling entirely on your behalf.

**How it works, step by step:**

```
Step 1: You write your code (for example, a single JavaScript file)
Step 2: You upload just that code file to AWS Lambda
Step 3: AWS gives you back a public URL
Step 4: You share or use that URL

When a request hits that URL:
   AWS Lambda automatically loads your code
   AWS Lambda executes it to handle that one request
   AWS Lambda then destroys that instance of the function

If 10 requests arrive at once:
   AWS Lambda spins up 10 separate function instances
   Each one handles one request, then is destroyed
```

You never decide how much CPU or RAM the underlying machine has. You never write an auto-scaling policy. You simply hand over your code, and AWS handles invocation and scaling automatically, one lightweight function execution per request.

### Advantages of Serverless

1. **Very low cost for typical workloads.** AWS Lambda's free tier includes 1 million free requests per month, plus 400,000 GB-seconds of free compute time (compute time is billed as memory allocated multiplied by execution time; 400,000 GB-seconds works out to about 3.2 million seconds of runtime only if a function is configured with the smallest possible memory allocation of 128MB). Beyond that, pricing is very cheap (a small fraction of a dollar per additional million requests), which is a tiny cost for most profitable applications.
2. **You only pay for what you actually use.** With a traditional server, you pay per hour that the machine is running, even if zero users show up. With Lambda, if there are no requests, you pay nothing at all.
3. **No infrastructure management overhead.** No CPU sizing, no RAM sizing, no manual auto-scaling group configuration.

### Disadvantages of Serverless

1. **Cold start.** If a Lambda function has not been used in a while (zero active users), the very first request after that gap has to "wake up" a fresh Lambda instance, which involves pulling your code and starting it. This first request pays a latency penalty (in the video's example, roughly a couple of seconds). Once the function is "warm," subsequent requests resolve much faster (in the video's example, around half a second). If your application has continuous, steady traffic, this problem barely matters, since the function rarely goes fully cold.
2. **Fixed maximum execution duration.** Because an API Gateway typically sits in front of Lambda, a single invocation cannot take longer than a fixed limit (by default, up to 29 seconds for an API Gateway to Lambda integration, though this can be raised via a service quota increase for regional or private APIs). Long-running tasks do not fit this model well.
3. **No real protection against being scaled infinitely under attack.** If a DDoS attack sends a huge number of requests, Lambda will simply keep invoking more and more function instances to try to keep up, which can also mean a runaway bill.
4. **Vendor lock-in.** Once your code and architecture are built specifically around Lambda's structure, switching to a different cloud provider later becomes very difficult, since you would need to rewrite your code and architecture from scratch. In practice, using Lambda also tends to pull in a whole ecosystem of other AWS-specific services around it over time, for example SQS for queuing, API Gateway for invocation and routing, Route 53 for domain management, S3 for storage, CloudWatch for logging and monitoring, and Step Functions for coordinating longer workflows. Each of these is individually convenient, but together they make it progressively harder to ever leave the AWS ecosystem, and they can add up in cost even though each individual Lambda invocation felt very cheap.
5. **No low-level configuration control.** You cannot configure things at the operating system level, since you never have access to the operating system in the first place.
6. **Always stateless.** A Lambda function instance is destroyed right after handling its request, so you cannot rely on it to hold onto any data or state between requests.
7. **Database connection issues at scale.** If many Lambda instances all try to connect directly to a database like MongoDB at the same time, this can create a huge number of simultaneous database connections, which can overwhelm the database and cause connections to drop. Solving this typically requires adding a separate, stateful connection-pooling layer (for example, running on a persistent EC2 instance) in front of the database, which itself becomes an added cost and a potential single point of failure.

**Key takeaway:** Serverless looks very attractive on paper, cheap, and hands-off, but real production experience often reveals significant hidden complexity and cost once you dig deeper, especially around cold starts, vendor lock-in, and statefulness.

The term "serverless" essentially describes the opposite of the traditional model, which is called **server-ful architecture**, meaning you (or your team) are responsible for managing the server yourself.

---

## 8. Back to Server-ful Machines: The "Works on My Machine" Problem

Let's return to a traditional, server-ful setup: a single machine with a static IP, given 2 CPUs and 4GB of RAM, running as part of an auto-scaling group behind a load balancer.

**The scaling problem:** Suppose your code depends on external libraries or tools. For example, imagine your code depends on **FFmpeg**, a command-line tool used for video transcoding and manipulation. On your local laptop (say, running Windows or Mac), FFmpeg is already installed, and your code works fine locally.

When you deploy to a new server, that server does not have FFmpeg installed yet. So before your code can even run, you first need to run a setup script to install FFmpeg and any other dependencies. This setup process takes real time (running system updates, installing packages, installing drivers, and so on).

> **The core problem:** If a sudden traffic spike hits your system exactly while a new server is still being set up, users could already be bombarding your existing servers before the new one is even ready to help. By the time the new server finishes installing its dependencies, the existing servers may have already crashed under the load.

This points to two separate, deeper problems with traditional machines:

1. **Spinning up a brand-new machine is a heavy, slow process.**
2. **The classic "it works on my machine" problem.** Your code might depend on a specific version of a library (say, FFmpeg version 0.4.2) and a specific operating system (say, Windows or Mac locally). When you deploy to a fresh Ubuntu server in the cloud, a different version of that dependency might get installed, or some other configuration might differ, causing code that worked perfectly on your laptop to fail once deployed. For any reasonably sized codebase using 30 to 40 packages, keeping every single dependency version in sync between your local machine and every server becomes a real headache.

---

## 9. Virtualization: Solving "Works on My Machine"

> **Definition:** Virtualization lets you create a virtual machine (VM), a self-contained software-based computer, running its own full operating system, inside your physical machine (or inside a cloud server).

**How it works:**

```
Physical/Host Machine (Windows, Mac, or Linux, doesn't matter)
   |
   v
Virtual Machine (VM)
   |--> Its own full Operating System (for example, Ubuntu)
   |--> Your installed dependencies (for example, FFmpeg)
   |--> Your application code
```

You build and test everything inside this VM. Once you are sure it works, you deploy this entire VM image to your server. Since the VM carries its own complete operating system and dependencies bundled together, it no longer matters what the underlying host machine is. This solves the "works on my machine" problem: your code now genuinely works the same way on any machine, because it is really running inside the exact same VM environment every time.

**The downside of virtualization:** A full VM is heavy, because it bundles an entire second operating system on top of the host machine's own operating system. This means:

- It requires significantly more resources (2 CPUs and 4GB of RAM is no longer enough).
- Starting up a new VM takes real time, since a multi-gigabyte operating system image has to be pulled and booted before your code can even run.

So virtualization solves the consistency problem, but it does not solve the scaling-speed problem, and it adds noticeably more cost and complexity.

---

## 10. Containerization: A Lightweight Alternative to VMs

Large companies like Google faced this exact problem years ago while scaling massively: full VMs consumed too many resources and did not scale quickly enough. Their solution was **containerization**.

> **Definition:** A container is essentially a lightweight virtual machine. Instead of bundling a full separate operating system, a container shares the host machine's operating system kernel, and only bundles your application's dependencies and code on top of it.

```
A traditional VM might look like:
   Operating System (about 4 GB)
   + Packages/Dependencies (about 200 MB)
   + Your Code (about 50 MB)
   = Several GB in total

A container, by removing the separate OS layer, looks like:
   Packages/Dependencies (about 200 MB)
   + Your Code (about 50 MB)
   = Only a few hundred MB in total
```

Because containers are so much smaller and lighter than full VMs, you can run many more of them on the very same physical machine, and they start up far faster. When one physical machine's capacity is used up, you simply spin up another physical machine and keep launching containers on it too. Each container runs its own isolated code, and if a container crashes, you can simply destroy it and spin up a fresh one in its place, quickly.

Because a container is tested and built consistently (the same way a VM image is), the "works on my machine" guarantee still holds: if it works in the container on your laptop, it will work in that same container anywhere else too.

---

## 11. Container Orchestration: Why We Need a "Brain"

Once you start running many containers (the video gives an example of running 15 to 16 containers, or more, across a couple of physical machines), a new problem appears: **someone has to manage all of these containers.**

Consider what needs to happen automatically:

- When traffic increases, new containers need to be created.
- When traffic decreases, containers need to be safely removed.
- If a container crashes or enters an error state, it needs to be deleted and replaced with a fresh one.
- When you deploy a new version of your code, old containers need to be gradually replaced with new ones, ideally without any downtime. This pattern is called a **rolling update**. Another common deployment strategy for this is called **blue-green deployment**.

> **Definition:** Container orchestration is the process of automating the deployment, management, and scaling of containerized applications across a cluster of servers.

```
Cluster of Servers
   |--> Server 1: running several containers
   |--> Server 2: running several containers
   |--> Server 3: running several containers

The Orchestrator ("brain") automatically:
   |--> creates new containers when traffic increases
   |--> removes containers when traffic decreases
   |--> replaces crashed or errored containers
   |--> performs rolling updates when new code is deployed
```

Manually managing which container is being created, which one is being destroyed, and which one needs to be restarted, across many machines, is simply too much for a person to track by hand. This need for an automated "brain" is exactly why container orchestration tools exist.

---

## 12. The History of Kubernetes

Google faced this exact container-management challenge internally, years before most other companies. Their internal solution was a system called **Borg**.

> **Definition:** Google Borg was a cluster management system, developed and used internally by Google, to manage its massive data center infrastructure and containerized workloads.

Google did not open-source Borg itself. However, the same team that built Borg was later asked to take everything they had learned and rewrite it as a brand-new project, built from scratch rather than reusing Borg's original code. This new project became a full container orchestrator, one capable of handling auto-scaling, auto load balancing, automatic creation, and automatic destruction of containers, for any kind of use case.

Google then open-sourced this new project and donated it to the **CNCF (Cloud Native Computing Foundation)**. This project was named **Kubernetes**.

```
Google Borg (internal, never open-sourced)
        |
        |  same team rewrites the learnings from Borg into a brand-new project
        v
New project ("Project X")
        |
        |  open-sourced and donated
        v
CNCF (Cloud Native Computing Foundation)
        |
        v
Kubernetes (publicly available, open source)
```

> **Definition:** Kubernetes (often abbreviated as K8s) is an open source container orchestration system that automates the deployment, scaling, and management of containerized applications.

Kubernetes was directly inspired by the learnings from Google's Borg project, and although Google created it, it is now maintained by a worldwide community of contributors, with its trademark held by the CNCF.

Kubernetes manages containers across a cluster of servers. It decides how to roll out updates, how to replace containers safely, and how to keep deployments running with as close to zero downtime as possible. Kubernetes also comes with its own built-in reverse proxy and load balancing support, and it has strong support for ingress and egress controllers, meaning it can manage a great deal of networking and traffic routing on its own.

> **Note from the video:** The creator mentions a separate, dedicated video that explains Kubernetes from scratch in much more depth, covering components such as the Control Plane, the API Server, etcd, the Scheduler, Worker Nodes, kubelet, kube-proxy, and how Pods are created and scheduled.

**Why containers plus Kubernetes are so widely used in modern system design:** this combination offers a strong balance. Scaling out quickly and easily, replacing unhealthy instances easily, performing rolling updates and blue-green deployments easily, and achieving close to zero-downtime deployments, all while relying on a mature, open source orchestrator instead of having to build your own custom "brain" to manage everything.

---

## 13. Load Testing and Stress Testing

Behind a smooth, large-scale live streaming experience (like watching a cricket match with tens of millions of concurrent viewers), there is a huge amount of engineering, trial and error, and past crashes that shaped the current system.

> **Definition:** Load testing and stress testing involve deliberately simulating heavy, realistic (or even higher-than-realistic) traffic on a system ahead of a known high-traffic event, in order to check whether the system is truly fault tolerant enough to handle it.

> **Example:** Companies like Hotstar, ahead of a big cricket match they know is coming, will often simulate a large volume of fake traffic against their own systems a day in advance, deliberately running their servers under full stress, in order to discover the exact limits of what their system can currently handle before the real event happens.

Even something as ordinary as smoothly streaming a 4K video on YouTube today relies on a large amount of underlying system design work: CDNs, edge caching, load balancers, and everything else covered in these two parts, all working together behind the scenes.

---

## Summary: All Components Covered in Part 2

| Concept | Purpose |
|---|---|
| Traffic Pattern Analysis | Understanding your own system's specific load behavior before choosing an architecture |
| Predictable Traffic (Netflix) | Pre-scaling and pre-caching ahead of planned events, like movie releases |
| Unpredictable Traffic (YouTube) | Designing for sudden, unplanned spikes since publishing is not controlled centrally |
| Mixed Traffic (Hotstar) | Balancing predictable live-event scaling with unexpected side effects, like a movies-API spike caused by users pressing "back" |
| Auto-Scaling Pitfalls | Average-based auto-scaling can fail against sudden spikes and dips; sometimes it is safer to disable it and stay scaled up |
| Serverless (AWS Lambda) | Fully managed, pay-per-use compute with no server management, at the cost of cold starts, execution limits, and vendor lock-in |
| Server-ful Architecture | Traditional self-managed servers with full control but far more operational overhead |
| Virtualization (VMs) | Solves "works on my machine" by bundling a full OS, at the cost of being heavy and slow to scale |
| Containerization | A lightweight alternative to VMs that shares the host OS kernel, making scaling much faster and cheaper |
| Container Orchestration | Automating the deployment, scaling, and healing of containers across a cluster |
| Kubernetes | The open source container orchestrator, born from Google's internal Borg system and donated to the CNCF |
| Load and Stress Testing | Proactively simulating heavy traffic ahead of known events to validate fault tolerance |

**What comes next, per the video:** a dedicated, from-scratch explanation of Kubernetes internals (Control Plane, API Server, etcd, Scheduler, Worker Nodes, kubelet, kube-proxy, and Pods) is referenced as a separate video.
