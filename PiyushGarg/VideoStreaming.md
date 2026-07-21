# How Video Streaming Works on Scale - System Design

Source video: [How Video Streaming Works on Scale - System Design](https://www.youtube.com/watch?v=-JtjQ-OA7XE) (Hindi, about 31 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow. We live in an entertainment-driven era of TikTok, Instagram Reels, and YouTube, and delivering video smoothly to a client takes a genuinely large amount of engineering effort, software, and hardware working together behind the scenes.

---

## 1. What Is a Video, Really?

Before understanding video streaming, it helps to understand what a video actually is at a technical level.

> **Definition:** A video is simply a sequence of images (frames), played back one after another at a certain speed. Each individual image is a snapshot of a single frame. When these frames are played back in quick succession, the human brain perceives them as continuous motion, rather than as separate still images.

> **Example:** If a camera takes a photo of you once every second, and those photos are played back one after another, your brain will start to perceive it as a moving picture rather than a slideshow.

**Frame rate** controls how smooth this motion looks:

```
64 frames per second -> camera captures 64 images every second -> very smooth motion
20 frames per second -> camera captures 20 images every second -> noticeably less smooth
1 frame per second   -> camera captures 1 image every second   -> motion looks jerky and stuttering
```

Formats like **MP4** are simply a way of storing this sequence of images (along with audio) so that a computer can play them back in order, at the correct frame rate.

**The obvious consequence:** since a video is really just a large sequence of images, its file size grows quickly. Even a fairly short video today can easily reach 1GB, and a typical high-quality YouTube video is often in the range of 4 to 5GB. Making a file this large play back smoothly on a viewer's device is a genuine engineering challenge, and this is exactly the problem that video streaming exists to solve.

---

## 2. The Early Era: Progressive Download

In the early 2000s, when video platforms first emerged, there was no real concept of "streaming" yet.

> **Definition:** Progressive download means that a client must download the **entire** video file from the server before it can start playing it at all.

```
Server (stores a 100 MB video)
   |
   |  client requests the video
   v
Server sends the entire 100 MB file
   |
   |  client must wait for the full download to complete
   v
Only after the full file has arrived, playback can begin
```

This waiting period is what's known as **buffering**. Now imagine this same approach applied to a modern 4K or 8K video that's 4 to 5GB in size. Even at a solid connection speed of 150 to 200 Mbps, you could be waiting several minutes before playback even starts, all while wasting a large amount of bandwidth if you decide midway through that you don't actually want to watch it. This was a genuinely poor experience, especially on low-end devices or weak network connections, and it's exactly why buffering was such a widely complained-about issue in earlier internet eras.

---

## 3. The Next Step: Real-Time Streaming Protocols (RTMP and RTSP)

As technology evolved, dedicated streaming protocols were introduced to solve the progressive download problem.

> **Definition:** RTMP (Real-Time Messaging Protocol, originally developed by Macromedia in 2002, and later kept proprietary by Adobe after it acquired Macromedia in 2005, before Adobe eventually released it as an open specification in 2012) and RTSP (Real-Time Streaming Protocol, jointly developed by RealNetworks, Netscape, and Columbia University and standardized as RFC 2326) are specialized streaming protocols that allow video to be sent to a client in **chunks**, instead of requiring the entire file to be downloaded up front.

These protocols introduced two major benefits:

- **Low latency and live streaming support.** You no longer had to wait for an entire 100 MB file. You could start streaming a video in pieces almost immediately.
- **Efficient use of bandwidth.** Under progressive download, if you started watching a video and left partway through, you'd already wasted bandwidth downloading parts you never watched. With chunk-based streaming, you only ever download the parts you actually play, so leaving early wastes far less bandwidth and network capacity.

```
Server
  |
  |  sends the video in small chunks, one after another
  v
Client plays each chunk as it arrives, without waiting for the whole file
```

**But a problem still remained.** Consider a very high quality, say 4K, source video. Even a single chunk of a 4K video can be quite large. Today's world has an enormous variety of devices: mobile phones, laptops, large LED screens, smartwatches, even fridge displays, all with vastly different screen sizes and network conditions.

> **Example of the remaining problem:** A large, high-resolution 4K video chunk (say, 200 MB per chunk out of a 100GB source video) makes perfect sense on a big LED screen with a fast connection. But does a smartwatch, with its tiny screen, really need 4K resolution? Does a fridge display need it? And if a client's network connection is weak (say, only 10 to 30 Mbps), even downloading a single large chunk could take several seconds, causing buffering all over again, even though chunk-based streaming was supposed to solve exactly that.

The user experiencing buffering on a small screen or a weak connection will still blame the platform for a bad experience, even though the root cause is really about mismatched video quality versus their specific device and network. This is exactly the gap that **adaptive bitrate streaming** was built to close.

---

## 4. Adaptive Bitrate Streaming (ABR)

> **Definition:** Adaptive bitrate streaming lets the client itself decide what video quality (resolution/bitrate) is best for its own current screen size and network speed, and lets that choice change dynamically over time as network conditions change. The two major protocols used for this are **HLS (HTTP Live Streaming)**, created by Apple, and **MPEG-DASH (Dynamic Adaptive Streaming over HTTP)**.

The core idea: instead of forcing every device to receive the same fixed quality, let the client pick. A small screen or a weak connection can be served a lower quality (a bit blurry, a bit pixelated), while a large 4K screen with a fast connection can be served full 4K quality, all from the very same underlying source video.

> **Example:** If you're traveling with poor connectivity, you have two options: either you get full 4K streaming that constantly stutters and buffers, or you get a slightly lower, blurrier quality that plays smoothly without interruption. Adaptive bitrate streaming picks the second option automatically, and switches back to higher quality automatically once your connection improves. Something playing smoothly, even at lower quality, is a far better experience than something stuck buffering.

### How Adaptive Bitrate Streaming Actually Works

**Step 1: Pre-process (encode) the source video into multiple quality levels.**

You cannot hand the original 4K source file directly to the client, since that would just be progressive download again. Instead, before any client ever requests it, the video is processed through an encoding pipeline that creates multiple versions of it, at different resolutions:

```
Source Video (4K, e.g. 4GB)
   |
   |  encoding pipeline
   v
   |--> 240p version, split into small chunks ("segments")
   |--> 360p version, split into segments
   |--> 480p version, split into segments
   |--> 720p version, split into segments
   |--> 1080p version, split into segments
   |--> 4K version, split into segments
```

This is exactly why a freshly uploaded YouTube video doesn't publish instantly. It typically takes 20 to 30 minutes to finish this entire encoding pipeline (producing all these different resolution segments) before it actually goes live.

**Step 2: Create a manifest (index) file.**

Once you have dozens or hundreds of small segments across multiple resolutions, the client needs a way to know which segment belongs to which resolution, and in what order to play them.

> **Definition:** A manifest file (called an `.m3u8` file, or "index file," in HLS; an `.mpd` file in MPEG-DASH) acts like the index of a notebook. It doesn't contain the actual video data. Instead, it lists which resolutions are available, and where to find the corresponding sequence of segments for each one.

```
Manifest File (e.g. index.m3u8)
   |--> "For 240p, here are the segment locations, in order"
   |--> "For 360p, here are the segment locations, in order"
   |--> "For 480p, here are the segment locations, in order"
   |--> "For 720p, here are the segment locations, in order"
   |--> "For 1080p, here are the segment locations, in order"
   |--> "For 4K, here are the segment locations, in order"
```

**Step 3: The client reads the manifest and picks the best quality for itself.**

```
Client A: a 4K monitor with a 300 Mbps connection
   -> reads the manifest -> picks the 4K segment list -> streams in full 4K

Client B: a mobile phone with only a 30 Mbps connection
   -> reads the manifest -> picks the 480p segment list -> streams in 480p
```

Both clients are watching the exact same underlying video, each at the quality best suited to their own device and network, and this is precisely what "adaptive bitrate streaming" means.

### Adapting Dynamically, Mid-Playback

According to the ImageKit article referenced in the video, adaptive bitrate streaming results in very little buffering, a fast start time, and a good experience on both high-end and low-end connections. Typically, the client starts by loading the manifest file and initially chooses a low bitrate to begin playback quickly. From there:

```
If network throughput > bitrate of the currently downloading segment:
   -> client requests a higher bitrate segment next time (quality improves)

If network throughput drops (connection worsens):
   -> client requests a lower bitrate segment next time (quality decreases, to avoid buffering)
```

The exact algorithm used to decide which segment to fetch next can vary from client to client, and can factor in things like current network conditions and the device's screen size. This is what allows a video player to start at a lower quality, gradually increase quality as it confirms the network can handle it, and gracefully drop back down if the network gets worse, all automatically, without the user having to do anything.

---

## 5. A Practical Demo: Adaptive Bitrate Streaming With ImageKit

The video walks through a live demo using **ImageKit.io**, a service that handles video encoding and adaptive bitrate streaming as a managed product, so you don't have to build this entire pipeline yourself.

**What the demo showed:**

1. **Uploading a plain MP4 file** to ImageKit. On its own, this file only supports progressive download (the whole file has to download before it plays), since it hasn't been split into segments yet.
2. **Requesting the master manifest** by appending a specific transformation string to the file's URL (in ImageKit's case, something like `ik-master.m3u8`). This triggers ImageKit's encoding pipeline to generate multiple resolution renditions (240p, 360p, 480p, 720p, 1080p, and so on) and segment them, similar to the pipeline described above. This encoding step takes a little time to complete, just like YouTube's own processing delay.
3. **Inspecting the network tab** in the browser while playing the video showed exactly the mechanism described earlier in action:
   - A request for the master `.m3u8` manifest file, which listed the available resolutions and pointed to a separate manifest file for each one (240p, 360p, 480p, and so on).
   - Once the client picked a resolution (say, 360p), a request for that resolution's own manifest file, which in turn listed all of that resolution's individual video segments.
   - A series of requests for the actual segment files themselves, played back one after another.
4. **Manually switching quality** (say, from 480p up to 1080p, or back down) in the demo caused the player to immediately start requesting a different set of segments at the new resolution, visibly changing the picture quality within moments.
5. **Simulating a slow network** (throttling to a slower connection type in the browser's dev tools) caused the player to automatically fall back to lower-resolution segments on its own, exactly as adaptive bitrate streaming is supposed to behave.

**HLS versus MPEG-DASH, in practice:** both are supported by services like ImageKit. The only real difference in usage is the file extension you request: append something like `ik-master.m3u8` for **HLS**, or the equivalent `.mpd`-style request for **MPEG-DASH**. Both formats are fundamentally doing the same job: providing a manifest that points to multiple resolution-specific segment sets.

---

## 6. Why Most Teams Don't Build This Pipeline Themselves

Building your own video streaming pipeline from scratch involves several genuinely hard engineering problems:

- **A large amount of storage**, since you're storing the same video multiple times over, once per resolution, each split into many small segments.
- **A large, complex encoding pipeline**, since every uploaded video needs to be transcoded into multiple resolutions and packaged with the correct manifest files.
- **Significant ongoing maintenance and cost**, since running and scaling this kind of pipeline reliably, especially at high upload volumes, is expensive and operationally demanding.

This is exactly why many startups and smaller platforms choose to offload this part of their system to specialized managed services (like ImageKit, in this video's example), rather than building and maintaining a custom video encoding and adaptive streaming pipeline themselves.

---

## Summary: How Video Streaming Evolved

| Stage | How It Worked | Main Limitation |
|---|---|---|
| Progressive Download | Client downloads the entire video file before playback can start | Long wait times (buffering), especially for large or high-quality videos |
| RTMP / RTSP (Real-Time Streaming Protocols) | Video sent to the client in chunks; enabled low latency and live streaming | A single, fixed quality still didn't suit every device or network condition |
| Adaptive Bitrate Streaming (HLS / MPEG-DASH) | Source video pre-encoded into multiple resolutions and segments; a manifest file lets the client choose and dynamically switch quality | Requires a genuinely complex encoding pipeline, storage, and infrastructure to build and maintain |

**Key takeaway:** modern video streaming works by pre-processing a source video into multiple quality levels split into small segments, publishing a manifest file that describes where each segment lives, and letting the client itself decide (and continuously re-decide) which quality level best matches its current screen size and network conditions, rather than forcing every viewer to receive the exact same video quality regardless of their device or connection.
