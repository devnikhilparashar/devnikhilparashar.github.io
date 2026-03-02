# Live Streaming Systems — Notes & Glossary (RTMP, HLS, LL‑HLS)

RTMP vs HLS vs LL‑HLS • Why “just WebSockets” doesn’t scale • Latency math • Bitrate/ABR basics • Core terminology

**Quick mental model**
- **Ingest (publisher → server):** often **RTMP** (or SRT)
- **Delivery (server/CDN → viewers):** often **HLS / DASH / LL‑HLS**
- **Primary tradeoff:** **Latency ↔ scale ↔ stability**

---

## Contents

1. [High-level systems: RTMP vs HLS, and HLS vs LL‑HLS](#1-high-level-systems-rtmp-vs-hls-and-hls-vs-llhls)
2. [Why we can’t do a “simple” publisher → server → clients setup](#2-why-we-cant-do-a-simple-publisher--server--clients-websockets-setup)
3. [Latency calculations + bandwidth math](#3-latency-calculations-rtmp-hls-llhls--bandwidth-math-for-resolution)
4. [Glossary of key terms](#4-definitions-of-key-terms)
5. [Appendix: commands](#appendix-commands-we-used-in-the-lab)

> Suggested local lab tools: **OBS** (publisher), **nginx‑rtmp** (ingest), **VLC** (player), **FFmpeg/FFprobe** (inspect frames, timestamps, keyframes).

---

## 1) High-level systems: RTMP vs HLS, and HLS vs LL‑HLS

### RTMP (primarily used for ingest)
- **What it is:** A long-lived connection from a broadcaster (e.g., OBS) to an ingest server.
- **Where it fits:** **Publisher → ingest** side.
- **Why it’s used:** Stable persistent streaming from a single sender into the platform.

### HLS (primarily used for delivery)
- **What it is:** HTTP-based segmented streaming: manifests (`.m3u8`) + media segments (`.ts` or `.m4s`).
- **Where it fits:** **Origin/CDN → viewer** side.
- **Why it’s used:** Scales via CDNs because segments are cacheable HTTP objects; supports **ABR** (adaptive bitrate).

### LL‑HLS (Low-Latency HLS)
- **What it is:** An HLS variant that reduces latency by delivering **partial segments** (chunks) and reducing playlist update delay.
- **Key mechanisms:** partial segments (`#EXT-X-PART`), preload hints, and blocking playlist reload (server control).
- **Result:** Can reduce end-to-end latency from ~15–30s (traditional HLS) to ~2–5s (LL‑HLS), while still using HTTP/CDNs.

### Pros & Cons (quick comparison)

| System | Best for | Pros | Cons |
|---|---|---|---|
| **RTMP** (ingest) | Broadcaster → platform ingest | Persistent connection, simple broadcaster tooling (OBS), good ingest stability | Not ideal for large-scale viewer delivery directly; not CDN-friendly for massive fan-out |
| **HLS** (delivery) | Massive scale to viewers | CDN caching, ABR support, works well over HTTP, resilient to many network issues | Higher latency (segment + buffer), more components (packaging, manifest updates) |
| **LL‑HLS** (delivery) | Lower-latency at large scale | 2–5s latency possible, still HTTP/CDN compatible | More requests/overhead, more operational complexity than standard HLS |

**Rule of thumb:** Use **RTMP/SRT** for ingest, and **HLS/LL‑HLS** for delivery to many viewers. Delivery choice depends on latency requirements.

---

## 2) Why we can’t do a “simple” publisher → server → clients (WebSockets) setup

### The simple idea
“Broadcaster sends bytes to a server; server keeps client connections open (WebSockets) and forwards the same bytes to everyone.”

### Key limitations
- **Timed media, not a file:** Video/audio are sequences of frames/samples with **timestamps**. Late data is effectively “wrong”, not just “slow”.
- **Codec dependencies:** Most frames (P/B) depend on other frames. New viewers often can’t start decoding until the next **keyframe (I-frame)**.
- **Fan-out cost:** One publisher at 6 Mbps with 1,000,000 viewers implies ~6 Tbps outbound if your origin pushes to everyone. That’s why we use **CDNs** and cacheable HTTP objects (HLS segments).
- **Backpressure:** With push, slow viewers create per-connection queues. Memory grows, tail latency grows, and servers destabilize.
- **Adaptive bitrate (ABR):** A single bitrate stream fails users on poor networks. ABR requires multiple renditions (1080p/720p/480p/360p).
- **Operational reality:** Pull-based segmented delivery is easier to scale, cache, retry, and observe.

### Why we need “extra components” (encoder, transcoder, packager)
- **Encoder:** Converts raw frames to compressed bitstream (H.264/H.265/AV1). Can be software (CPU) or hardware (media engine).
- **Transcoder:** Produces multiple resolutions/bitrates for ABR (the “ladder”).
- **Packager:** Wraps compressed audio/video into streaming containers/segments (TS/fMP4/CMAF) and publishes manifests (HLS playlists).
- **Decoder:** On viewer device/browser. Often hardware-accelerated for performance and battery.

**Key insight:** “Just forward bytes” works for small audiences. At internet scale, you need **stateless pull**, **cacheable objects**, and **ABR**.

### Local experiment (keyframes)
- OBS streams to RTMP ingest: `rtmp://localhost:1935/live/test`
- VLC connects later and may wait for next keyframe to start decoding.
- Changing OBS keyframe interval (e.g., 2s → 10s) increases worst-case join delay.

---

## 3) Latency calculations (RTMP, HLS, LL‑HLS) + bandwidth math for resolution

### Where latency comes from (pipeline view)

Total latency ≈ capture + encode + network + buffering + decode + render

### RTMP latency (typical)
- Often: ~1–5 seconds end-to-end (varies widely by encoder settings and player buffering).
- Why: Persistent stream transport, but still needs buffering for stability.

### Traditional HLS latency (typical)
- Typical segment duration: 4–6 seconds
- Typical startup buffer: 2–3 segments
- Ballpark latency: ~15–30 seconds

**Example math:** If segment = 6s and player buffers 3 segments, latency can be ≈ 6s (segment completion) + 12s (buffer) + overhead ≈ **18–25s**.

### LL‑HLS latency (typical)
- Partial segment size: ~200–500ms
- Buffer target: a few partials
- Ballpark latency: ~2–5 seconds

**Example math:** If part = 0.5s and buffer = 3 parts, latency can be ≈ 0.5s (part ready) + 1.5s (buffer) + overhead ≈ **2–3s**.

### Why bitrate ladders look like: 1080p 6 Mbps, 720p 3 Mbps, 480p 1.5 Mbps, 360p 0.8 Mbps
- **Pixel count scaling:** More pixels per frame generally need more bits to preserve similar perceptual quality.
- **Bits-per-pixel intuition:** Many practical ladders keep a roughly similar “bits per pixel per second” range across resolutions.
- **ABR safety margin:** Players pick a rendition below measured bandwidth to avoid rebuffering.

### Bandwidth math (uncompressed baseline)

Uncompressed bitrate approximation:

```text
bitrate ≈ width × height × bits_per_pixel × fps
```

Example: 1920×1080, 24 bpp (RGB), 30 fps ≈ **1.49 Gbps** (far too large) → compression brings it down to single‑digit Mbps.

### Review questions (what we covered)
- Why video is a timed stream (frames, timestamps) and not “just bytes”.
- Why keyframe interval affects join latency.
- Why CDN-friendly segmented delivery scales better than pushing to millions of sockets.
- What makes LL‑HLS lower latency (partials, blocking playlist reload, preload hints).

---

## 4) Definitions of key terms

| Term | Definition | Why it matters |
|---|---|---|
| **Frame** | A single video image in time (part of a timed sequence). | Playback requires frames to arrive decodable and on time. |
| **I-frame (Keyframe)** | Intra-coded frame; self-contained image (like a JPEG in concept). | Entry point for new viewers; enables seeking; larger in size. |
| **P-frame** | Predicted from past reference frame(s) using motion vectors + residuals. | Smaller than I-frames but depends on previous frames. |
| **B-frame** | Bi-directional predicted using both past and future reference frames. | Better compression but introduces reordering; impacts latency/buffering. |
| **FPS (Frames Per Second)** | Number of frames displayed per second (e.g., 30 fps → 33ms/frame). | Defines timing cadence and impacts bitrate and motion smoothness. |
| **Bitrate** | Bits per second used for video/audio (e.g., 6 Mbps). | Higher bitrate can improve quality but requires more bandwidth. |
| **PTS (Presentation Timestamp)** | When a frame should be displayed. | Needed to play timed media correctly and keep A/V in sync. |
| **DTS (Decoding Timestamp)** | When a frame must be decoded (may differ from PTS when B-frames exist). | Explains why decode order ≠ display order; affects buffering/latency. |
| **GOP (Group of Pictures)** | Pattern/interval between keyframes (I-frame distance). | Controls join time and recovery; shorter GOP → faster join but more bits. |
| **Buffering** | Holding media ahead of playback to absorb network jitter/throughput drops. | More buffer → more stability but higher latency. |
| **ABR (Adaptive Bitrate)** | Client switches between multiple renditions based on bandwidth/conditions. | Prevents rebuffering and improves QoE across diverse networks. |
| **Encoder / Decoder** | Encoder compresses raw frames; decoder reconstructs frames for display. | Core compute for video; can be hardware-accelerated. |
| **Transcoder** | Creates multiple versions (resolutions/bitrates) from one input stream. | Enables ABR ladders (1080p/720p/480p/360p). |
| **Packager** | Wraps encoded streams into containers/segments + produces manifests (HLS playlists). | Makes delivery CDN-friendly and browser/player-friendly. |

**Practical debug hint:** FFmpeg `-vf showinfo` outputs per-frame info (type, `iskey`, `pts_time`, `duration_time`). Keyframes show `iskey:1`.

---

## Appendix: Commands we used in the lab

### Run local ingest (nginx-rtmp)
```bash
docker compose up -d
```

### OBS stream URL
```text
Server: rtmp://localhost:1935/live
Stream key: test
Full URL: rtmp://localhost:1935/live/test
```

### Inspect frames in real time
```bash
ffmpeg -i rtmp://localhost:1935/live/test -vf showinfo -f null -
```

### Capture a short sample
```bash
ffmpeg -y -i rtmp://localhost:1935/live/test -t 10 -c copy sample.flv
```

---

created with ❤️ by Chatgpt & Nikhil
