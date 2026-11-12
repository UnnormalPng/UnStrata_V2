60% of all project done, started 23/10/2026.

# UnStrata V2

UnStrata V2 is a sovereign, ultra-fast local desktop runtime for large AI models. It is designed to run heavy Sparse Mixture-of-Experts (MoE) models on regular consumer PCs with limited video memory.

Completely free from cloud dependencies, telemetry, proprietary licensing checks, and Electron bloat.

---

## What is UnStrata V2?

Running modern, massive language models locally usually requires expensive enterprise GPUs or suffers from extreme slowness when offloaded to standard RAM.

UnStrata solves this problem through an intelligent multi-tier memory pipeline:
- The base model layers live directly in fast GPU VRAM.
- The most frequently used expert layers stay cached in system RAM.
- Infrequent expert layers stream instantly from high-speed NVMe storage on demand.

The result: you can run massive, highly capable MoE models locally on a setup like an RTX 4060 Ti (16 GB) and 32 GB of system RAM, maintaining high generation speeds (35-50+ tokens per second).

---

## Key Features

### 1. Built with Pure Rust (Tauri v2)
- Replaced Electron (~1.8 GB idle RAM) with a lightweight native Rust architecture consuming only around 35 MB of RAM in idle.
- Fast, secure Win32 integration with no console windows popping up in the background.
- Eliminates zombie processes: internal tools and background scripts run inside managed Windows Job Objects that clean up automatically on exit.

### 2. Universal Model Support
- UnStrata is not locked to a single model architecture. Any Sparse MoE model (Qwen, Mixtral, DeepSeek, DBRX, or custom hybrid fine-tunes) can be executed.
- Supports high-efficiency quantization formats (such as Trellis TQ1_0 and GGUF) designed to preserve model reasoning while drastically reducing memory consumption.

### 3. Dual Engine Architecture
- **Embedded Engine:** Ships with an optimized, hardened local C++ inference binary (`strata.exe`) calibrated for physical CPU cores and GPU offloading.
- **External Provider Mode:** Can act as a universal, clean front-end for any OpenAI-compatible server. Easily route your chats to local `llama.cpp --server`, `Ollama`, `vLLM`, or custom remote endpoints.

### 4. Speculative Decoding Pipeline (MTP)
- Uses an onboard lightweight draft head to predict upcoming tokens ahead of time.
- Bypasses traditional PCIe bus bottlenecks, multiplying token throughput up to 3-4x compared to unassisted offloading.

### 5. Private, Sovereign & Offline
- Strict local network loopback (`127.0.0.1`). No usage data, chats, or telemetry ever leave your machine.
- Fully offline document processing: inspect spreadsheets, documents, and local files directly on your hardware.
- Anti-sycophancy contract: the runtime instructs models to stay objective, verify facts using tools, and never capitulate under simple doubt.

---

## System Requirements

- **Operating System:** Windows 10 or Windows 11 (64-bit).
- **Processor:** 8 physical CPU cores recommended.
- **Memory (RAM):** 32 GB.
- **Graphics (GPU):** NVIDIA GPU with at least 16 GB VRAM for stable work (RTX 4060 Ti 16GB (mine), RTX 3090, RTX 4080, etc.).
- **Storage:** High-speed NVMe PCIe SSD (2,000+ MB/s read speed recommended for expert streaming).
