### 3D LiDAR + Camera Fusion — Real-Time Human Detection with Distance Measurement

> **Domain:** Embedded Systems · GStreamer · Computer Vision · NVIDIA DeepStream · H.264/H.265  
> **Platform:** NVIDIA Jetson Orin Nano · Xilinx ZCU106  
> **Status:** Completed

---

## Core Skills

![C](https://img.shields.io/badge/C-00599C?style=flat&logo=c&logoColor=white)
![C++](https://img.shields.io/badge/C++-00599C?style=flat&logo=cplusplus&logoColor=white)
![GStreamer](https://img.shields.io/badge/GStreamer-Multimedia-informational?style=flat)
![NVIDIA DeepStream](https://img.shields.io/badge/NVIDIA-DeepStream-76B900?style=flat&logo=nvidia&logoColor=white)
![CUDA](https://img.shields.io/badge/CUDA-12.6-76B900?style=flat&logo=nvidia&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Embedded-FCC624?style=flat&logo=linux&logoColor=black)
![PetaLinux](https://img.shields.io/badge/Xilinx-PetaLinux-E01F27?style=flat)
![Meson](https://img.shields.io/badge/Build-Meson%20%2B%20Ninja-grey?style=flat)

---

## Projects

---

This project started as a straightforward integration task — get a LiDAR sensor talking to a camera on an embedded board. It evolved, stage by stage, into a full real-time human detection system that measures the physical distance to each detected person using fused LiDAR depth data. Every plugin, algorithm, and architectural decision documented here was designed and implemented by me from scratch.

---

#### Stage 1 — Getting the Hardware to Talk

Before any software architecture could be designed, the hardware had to work reliably. This meant:

- Integrating the proprietary **Synexens CS20 SDK** to initialise the device, configure streaming parameters, and retrieve frames
- Understanding the CS20's output formats: RGB depth images at 320×240 and 640×480, and raw 3D point cloud data
- Resolving hardware-level issues on embedded Linux — USB permissions, device enumeration, and power delivery

The last point caused the most debugging time. The CS20 is sensitive to USB power quality. On embedded boards with limited USB power budgets, the sensor would initialise but produce corrupted or unstable output. The fix was a **12V externally powered USB hub** — something no documentation mentioned. Finding this required systematic elimination of every other possible cause first.

---

#### Stage 2 — Building the GStreamer Plugin Layer

With stable hardware output, the next problem was clean integration into a GStreamer pipeline. GStreamer has no built-in concept of a LiDAR sensor — so the first task was building the plugin layer from scratch.

**LiDAR Source Plugin (C)**

A custom GStreamer source element that captures LiDAR data and pushes it downstream as a GStreamer buffer. Key design decisions:

- **Dual output mode:** The plugin operates in either RGB depth image mode (for visualisation) or packed 3D point cloud mode (for AI inference), selectable at pipeline construction time
- **Task-based push model:** A dedicated GStreamer task thread runs the capture loop, continuously pulling frames from the SDK and pushing them downstream — decoupled from the pipeline's main thread
- **GObject properties:** `depth`, `width`, and `height` are exposed as runtime-configurable properties, so pipeline behaviour can be changed without recompilation

**LiDAR Display Plugin (C)**

A GStreamer transform element that receives LiDAR depth buffers and superimposes the depth data onto a live camera frame in real time. Depth images are colour-coded and blended into a corner of the video frame, with a distance legend rendered using **Cairo + Pango**.

**Result:** A working visualisation pipeline — live camera feed with real-time LiDAR depth overlay — running on both Jetson Orin Nano and Xilinx ZCU106.

---

#### Stage 2b — Designing for the Future: The Generic LiDAR Architecture

After the CS20 integration was working, a decision had to be made: leave the plugin hardcoded to CS20, or design it to support any LiDAR. The hardcoded approach was faster. The generic approach was correct.

The redesign introduced a **C++ abstraction layer** behind the C plugin — a clean separation between the GStreamer element (which knows nothing about LiDAR hardware) and the sensor backend (which knows everything about one specific LiDAR).

```
GStreamer Plugin (pure C)
        │
        │  calls only 5 bridge functions
        ▼
   C Bridge Header          ← explicit C ↔ C++ language boundary
        │
        ▼
   C++ Wrapper Layer
        ├── LidarBase (abstract interface)
        ├── CS20                   — compiled only if -Dlidar_type=CS20
        ├── G2                     — compiled only if -Dlidar_type=G2
        └── Wrapper Class          — selects and owns the active backend
```

**What this design achieves:**

- The GStreamer plugin never knows which LiDAR is active. It calls five bridge functions — initialise, start, get frame, stop, destroy — and nothing else.
- Adding support for a new LiDAR sensor means implementing one derived class and one overlay handler. Zero changes to the plugin itself. Zero changes to the build system beyond adding a new option value.
- The Meson build system compiles only the selected backend. Passing `-Dlidar_type=CS20` at configure time pulls in only the CS20 SDK and its implementation files. The final binary contains no dead code from unused sensor backends — keeping executable size minimal, which matters on constrained embedded hardware.
- The C/C++ language boundary is explicit and deliberate. No GLib types cross into C++ territory.

This is the **Open/Closed Principle** applied to embedded hardware integration — the system is open for extension (new sensors) and closed for modification (existing plugin code).

---

#### Stage 3 — Transmitting LiDAR Data Over the Network (ZCU106)

On the ZCU106 deployment, the requirement changed: the system needed to transmit both the camera video and LiDAR depth data to a remote receiver over UDP — as a **single stream**. Two separate streams would require receiver-side synchronisation logic. The goal was to avoid that entirely.

**The approach: embed LiDAR data inside the H.264 bitstream using SEI NAL units.**

---

##### What SEI Metadata Is — and Why It Fits

The H.264 and H.265 standards define a NAL unit type called **SEI — Supplemental Enhancement Information**. SEI NAL units travel inside the video bitstream but carry side-channel data. Decoders that do not understand them simply skip them, leaving the video stream fully intact. A standard-compliant H.264 decoder receiving this stream will decode video correctly — it will never see the LiDAR payload.

This made SEI the natural fit: one stream, no receiver-side sync complexity, no protocol changes. The LiDAR data rides alongside the video invisibly.

Understanding the SEI NAL unit structure required reading the H.264 and H.265 specifications in detail. The format is non-trivial: a 4-byte start code, a NAL unit header (different byte sequences for AVC vs HEVC), a payload type byte (`0x05` for unregistered user data), a multi-byte size field that uses `0xFF` prefix bytes for payloads larger than 255 bytes, a 16-byte UUID to identify the payload owner, the actual payload, and an RBSP stop bit. Getting every field right — especially the AVC vs HEVC header differences and the `0xFF` size encoding — took significant research and iteration.

**Transmitter — `data insertion` plugin:**

```
lidar source (data_sink pad) ──┐
                               ├──► data insertion ──► H.264 stream with embedded LiDAR ──► UDP
USB Camera (video_sink pad) ───┘
```

The plugin has two sink pads — one receiving the H.264 encoded video stream, one receiving the LiDAR buffer from `lidar source`. On every video frame, if a LiDAR buffer is available, the plugin constructs a correctly-formed SEI NAL unit from the LiDAR data, prepends it to the video buffer, and pushes the combined buffer downstream. A mutex protects the LiDAR buffer between the two pad threads.

**Receiver — `data extraction` plugin:**

```
UDP ──► data extraction ──► H.264 stream (clean) ──► decoder ──► display
              │
              └──► custom GStreamer event ("lidar-message") ──► lidar display
```

The extract plugin parses incoming buffers, detects the SEI NAL unit by its start code pattern and UUID, extracts the LiDAR payload, and wraps it in a **custom GStreamer downstream event** (`lidar-message`). This event travels downstream through the pipeline and is caught by the `lidar display` plugin, which uses it to render the depth overlay. The video buffer is reconstructed with the SEI NAL unit removed — a clean H.264 stream reaches the decoder.

---

##### The Byte Corruption Problem — and the DEADBEEF Algorithm

Embedding arbitrary binary data inside an H.264 bitstream introduced a serious problem. The H.264 start code sequence — `0x00 0x00 0x01` and its variants — is a reserved pattern in the bitstream. It signals the beginning of a NAL unit. If the LiDAR payload happened to contain these byte sequences, the H.264 parser on the receiver would misinterpret payload bytes as NAL unit boundaries, corrupting the entire stream. The result: garbled video, incorrect depth data, and in some cases complete decoder failure.

The standard solution to this is SODB (string of data bits) emulation prevention, which inserts `0x03` escape bytes. However, this approach added significant per-frame processing overhead and introduced memory allocation pressure in the data path — unacceptable on an embedded system running a real-time pipeline.

**The DEADBEEF escape algorithm** was designed as a lighter alternative:

**Encoder (transmitter side):** Before embedding the LiDAR payload, scan the data for any occurrence of `0x00 0x00` followed by `0x00`, `0x01`, `0x02`, or `0x03` — the byte patterns that could be mistaken for start codes. When found, replace the sequence with `0x00 0x00 0xDE 0xAD 0xBE 0xEF XX`, where `XX` is the original third byte. The DEADBEEF marker (`0xDE 0xAD 0xBE 0xEF`) cannot appear in a legitimate H.264 start code context, so it is unambiguous.

**Decoder (receiver side):** Scan the extracted payload for `0x00 0x00 0xDE 0xAD 0xBE 0xEF XX`. When found, restore the original `0x00 0x00 XX` sequence and continue. The decode pass runs in-place with no additional allocation.

The result: no stream corruption, no memory pressure, minimal per-frame CPU overhead. The entire algorithm fits in a tight linear scan of the payload buffer.

---

#### Outcomes

- End-to-end real-time pipeline on NVIDIA Jetson Orin Nano: live person detection with LiDAR distance overlay at camera framerates
- Same `lidar source` and overlay plugin codebase runs without modification on Xilinx ZCU106 under PetaLinux — full cross-platform portability
- On ZCU106: LiDAR depth data transmitted inside H.264 streams over UDP using custom SEI NAL unit embedding, with full extraction and visualisation on the receiver
- Plugin architecture supports adding new LiDAR hardware with zero changes to the core plugin or build system

---

#### Stage 4 — AI Inference and the Distance Measurement Problem

The core use case: detect humans in real time and display the actual physical distance to each detected person. This required combining NVIDIA DeepStream's inference pipeline with live LiDAR depth data — on a Jetson Orin Nano.

**Pipeline architecture:**

```
USB Camera
    │
    ▼
[nvstreammux]
    │
    ▼
[nvinfer]  ←  PeopleNet model (TensorRT INT8)
    │              produces 2D bounding boxes
    ▼
[Modified nvdsosd]  ←────────────────────────  lidar source
    │                    depth buffer fused
    ▼                    with bounding boxes
 Display / Encoder
```

The NVIDIA PeopleNet model runs TensorRT-optimised INT8 inference and produces bounding boxes around detected persons. A heavily modified version of the DeepStream On-Screen Display plugin receives both the bounding box metadata from the inference stage and the raw LiDAR depth buffer, fuses them, and renders distance measurements directly on the video output.

---

##### The Point Cloud Data Problem — Bit-Packing

Raw CS20 point cloud data contains a large number of 3D points per frame. Passing this volume of data through a GStreamer buffer on every frame introduced measurable lag in the pipeline output.

The solution was a **custom bit-packing algorithm** that encodes each 3D point `(px, py, pz)` as a compact fixed-width bit-stream:

- Projected pixel X, Y, Z(depth) coordinate: packed into minimum required bits to hold the data properly

The unpacker on the receiving end reconstructs the original 3D coordinates precisely. No heap allocation occurs inside the per-frame data path. Buffer sizes dropped significantly, and the lag was eliminated.

---

##### The Distance Measurement Problem — Why the Naive Approach Fails

The obvious approach to distance measurement is: find all LiDAR points within a bounding box, take their mean depth. It does not work reliably. Two reasons:

**1. Bounding boxes are noisy.** The PeopleNet bounding box is sized to contain the whole person — which means it also contains background pixels, clothing edges, partial occlusions, and whatever is behind or beside the subject. Averaging depth over the full box area averages in a large amount of noise.

**2. Camera and LiDAR are not hardware-synchronised.** On an embedded system without a hardware sync signal between the camera and LiDAR, even a small temporal offset between frames causes spatial misalignment between the bounding box position and the point cloud. The mean of a spatially misaligned region is not a reliable distance measurement.

**The solution — a two-step refinement:**

**Step 1: Reduce the region of interest to 20% of the bounding box, centred.**
Shrinking the box to its central 20% discards the noisy perimeter — background bleed, partial occlusions, edge artefacts — and focuses the depth estimate on the solid core of the person's body, which is the most stable region across frames.

**Step 2: Sample along the diagonal of the reduced box, not across the full area.**
Instead of computing a mean over all points in the reduced region, depth values are sampled only along the diagonal line of the reduced bounding box. A single diagonal sample:
- Is far less sensitive to frame-to-frame synchronisation jitter between camera and LiDAR
- Remains stable under partial occlusion
- Requires significantly less computation than a full area mean

The result was stable, accurate distance measurements under real operating conditions — asynchronous camera and LiDAR streams, partial occlusions, varying lighting — which is the only condition that matters on a deployed embedded system.

---

#### What I Built From Scratch

| Component | Description |
|---|---|
| `lidar source` | GStreamer C source plugin — task-based push model, dual output mode, GObject properties |
| `lidar display` | GStreamer C transform plugin — per-sensor overlay implementations, Cairo/Pango rendering |
| C++ Wrapper + Abstraction Layer | `LidarBase` interface, CS20 + G2 implementations, explicit C bridge design |
| Meson build system | Cross-platform, per-sensor SDK selection at configure time, conditional compilation |
| Bit-packing algorithm | Custom fixed-width encoder/decoder for 3D point cloud data — no heap allocation in data path |
| Bounding box refinement | 20% centre crop + diagonal mean — robust distance estimation without hardware sync |
| `data insertion` | GStreamer C plugin — dual-pad SEI NAL unit construction for H.264 and H.265 streams |
| `data extraction` | GStreamer C plugin — SEI NAL unit extraction, custom downstream event dispatch |
| DEADBEEF escape algorithm | Lightweight byte-level encoder/decoder to prevent start code collisions in embedded payloads |

---

#### Tech Stack

| Layer | Technology |
|---|---|
| Language | C (GStreamer plugins), C++ (wrapper + SDK integration) |
| Multimedia Framework | GStreamer 1.20+ |
| AI Inference | NVIDIA DeepStream 7.1, TensorRT 10.3, PeopleNet (INT8) |
| Hardware | NVIDIA Jetson Orin Nano, Xilinx ZCU106 |
| OS | Ubuntu 22.04 (Jetson), PetaLinux 2023.1 (ZCU106) |
| Build System | Meson + Ninja |
| Graphics | Cairo, Pango (pangocairo) |
| Video Codec | H.264 (AVC), H.265 (HEVC) |
| Transport | UDP (RTP/RTSP) |
| CUDA | 12.6 |
| LiDAR Hardware | Synexens CS20 (3D depth), YDLiDAR G2 (2D) |

---

## About

I am a System Software Engineer working at the intersection of embedded Linux, multimedia frameworks, and real-time computer vision. The problems I find most interesting are the ones where hardware constraints force genuine engineering decisions — where the right answer is not more compute, but a better algorithm, a cleaner abstraction, or a more careful understanding of the data.

I am actively looking for roles in embedded systems, multimedia pipeline engineering, and edge AI in Bangalore — particularly in environments where the full stack from hardware bring-up to application is in scope.

---

## Contact

Reach out via GitHub Issues or Discussions on this repository.
