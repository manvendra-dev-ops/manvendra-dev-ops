# Manvendra Pratap Singh

### System Software Engineer — Embedded Linux · Multimedia Pipelines · Edge AI

Embedded systems engineer at **MosChip**, building production-grade multimedia pipelines and sensor fusion systems on ARM-based platforms. I work close to the hardware — at the boundary where low-level data handling, system architecture, and machine learning inference intersect.

Most of my professional work lives on proprietary hardware and cannot be shared publicly. What I can share is documented in the portfolio and project repositories below — architecture decisions, engineering challenges, and the reasoning behind every design choice.

---

## Tech Stack

![C](https://img.shields.io/badge/C-00599C?style=flat&logo=c&logoColor=white)
![C++](https://img.shields.io/badge/C++-00599C?style=flat&logo=cplusplus&logoColor=white)
![GStreamer](https://img.shields.io/badge/GStreamer-Multimedia-informational?style=flat)
![NVIDIA DeepStream](https://img.shields.io/badge/NVIDIA-DeepStream-76B900?style=flat&logo=nvidia&logoColor=white)
![TensorRT](https://img.shields.io/badge/TensorRT-Inference-76B900?style=flat&logo=nvidia&logoColor=white)
![CUDA](https://img.shields.io/badge/CUDA-12.6-76B900?style=flat&logo=nvidia&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Embedded-FCC624?style=flat&logo=linux&logoColor=black)
![PetaLinux](https://img.shields.io/badge/Xilinx-PetaLinux-E01F27?style=flat)
![Yocto](https://img.shields.io/badge/Yocto-BSP-4a90d9?style=flat)
![Meson](https://img.shields.io/badge/Build-Meson%20%2B%20Ninja-grey?style=flat)
![ROS2](https://img.shields.io/badge/ROS2-Robotics-22314E?style=flat&logo=ros&logoColor=white)

---

## Work

**Trainee Engineer — System Software @ MosChip** *(Jun 2025 — Present)*

Independently owned and delivered multiple projects end to end — including a client-facing production deployment and a production-ready POC. Key work includes:

- Designed a **generic LiDAR abstraction layer** in C++ using polymorphism and inheritance — sensor-agnostic GStreamer source plugin with build-time SDK selection via Meson, keeping binaries lean on constrained hardware
- Built a **real-time person detection pipeline** on NVIDIA Jetson Orin Nano — fusing CS20 LiDAR depth data with DeepStream ML inference (PeopleNet, TensorRT INT8) to display per-person physical distance on live video
- Designed a **SEI NAL unit embedding system** to transmit LiDAR data inside H.264/H.265 streams over a single UDP channel — eliminating a separate data channel entirely, figured out from scratch using only SDK docs and the NAL specification
- Worked in team to deliver different client requirements, developed and delivered a **four-channel RTSP simulcast system** with PetaLinux — full stack: C/C++ GStreamer backend, HTML/JS frontend, SQL database with live sync, SMPTE timecode metadata over UDP

---

## Projects

| Project | Platform | Description |
|---------|----------|-------------|
| [LiDAR + Camera Fusion POC](https://github.com/manvendra-dev-ops/manvendra-dev-ops/blob/main/LIDAR_PROJECT.md) | Jetson Orin Nano · ZCU106 | Person detection with real-time LiDAR distance overlay |
| [GStreamer Plugin](https://github.com/manvendra-dev-ops/GStreamer-plugin) | Linux | Custom shape drawing plugin on NV12 video |
| [IPC Client-Server](https://github.com/manvendra-dev-ops/IPC-Client-Server) | Linux | Multithreaded TCP socket server in C |
| [ROS2 Attendance System](https://github.com/manvendra-dev-ops/ROS2-based-automated-attendance-system) | ROS2 | Automated attendance monitoring with XLS sync |

---

## Currently Learning

- **Linux Device Driver Programming** — Udemy: *Linux Device Driver Programming Using BeagleBone Black*
- **CUDA Programming** — GPU compute and parallel programming for edge inference optimization
- **Completed** — *Mastering Linux* (Udemy)

---

## Looking For

Open to embedded systems, multimedia pipeline, driver development, DSP/multimedia subsystems, and edge AI roles. Open to relocation across India — Bangalore, Pune, Hyderabad, Chennai, or remote.

---

## Connect

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Manvendra%20Pratap%20Singh-0A66C2?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/manvendra-pratap-singh-8864042b6)
[![GitHub](https://img.shields.io/badge/GitHub-manvendra--dev--ops-181717?style=flat&logo=github&logoColor=white)](https://github.com/manvendra-dev-ops)
[![Portfolio](https://img.shields.io/badge/Portfolio-Live-c8a96e?style=flat)](https://manvendra-dev-ops.github.io/portfolio)
[![Email](https://img.shields.io/badge/Email-manvendrapsingh780%40gmail.com-EA4335?style=flat&logo=gmail&logoColor=white)](mailto:manvendrapsingh780@gmail.com)
