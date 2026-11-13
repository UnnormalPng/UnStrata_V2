# UnStrata_V2

**A sovereign, open-source local AI workstation for Windows 11 engineered to run Sparse Mixture-of-Experts (MoE) and dense models alongside autonomous agents on consumer hardware.**

`UnStrata_V2` eliminates cloud dependencies, telemetry, and bloated Electron runtimes. Built around a native **Rust (Tauri v2 + WebView2)** host that idles at **~35 MB RAM** (compared to ~1.8 GB for Electron-based apps), it orchestrates multi-tier VRAM/RAM/NVMe offloading so virtually any local MoE or dense model—as well as any OpenAI-compatible inference server—can run smoothly on a single consumer GPU.

---

## 📊 Current Project Status (~62% Complete)

    Overall Progress:         [████████████░░░░░░░░] ~62%

    Native Rust Core:         [█████████████████░░░]  85%  (154 unit tests passing, 11 native tools)
    Engine & Memory Tiering:  [██████████████░░░░░░]  70%  (3-tier MoE offload ready, tuning RAM/VRAM margins)
    Security & Sandboxing:    [█████████░░░░░░░░░░░]  45%  (Win32 Job Objects & SSRF active; CSRF & anti-bypass next)
    Frontend UI (ui-kit):     [█████░░░░░░░░░░░░░░░]  25%  (Temporary patch layer active; clean prefab UI next)
    Native Vision Loop:       [███░░░░░░░░░░░░░░░░░]  15%  (Win32 window/input primitives ready; CapturePreview next)

---

## 🏗️ Architecture Overview

    ┌─────────────────────────────────────────────────────────────────────┐
    │ Frontend UI (WebView2 / HTML+JS+CSS)                                │
    │ Served locally over loopback with no-cache headers                  │
    └───────────────────────────────┬─────────────────────────────────────┘
                                    │ HTTP / SSE + Tauri IPC
    ┌───────────────────────────────▼─────────────────────────────────────┐
    │ Native Rust Host (src-tauri/, ~35 MB idle RAM)                      │
    │ • API Router & SSE Proxy        • Frame 0 & Auto Thinking Router    │
    │ • 11 Native Tools (FS/Web/OS)   • Native Telemetry (/api/hardware)  │
    │ • Win32 Job Objects & Lifecycle • Panic Hook & Single-Instance Lock │
    └───────────────┬───────────────────────────────────┬─────────────────┘
                    │ Embedded Mode (:8080)             │ External Mode
    ┌───────────────▼───────────────────┐   ┌───────────▼─────────────────┐
    │ Local Inference Engine            │   │ OpenAI-Compatible Endpoints │
    │ • VRAM: Dense layers + k8v4 KV    │   │ • llama.cpp / Ollama        │
    │ • RAM: Resident hot MoE experts   │   │ • vLLM / LM Studio          │
    │ • NVMe: mmap cold expert streaming│   └─────────────────────────────┘
    │ • Speculative MTP drafting        │
    └───────────────────────────────────┘

---

## 🛠️ Key Engineering Problems Already Solved

Building a self-contained, local agentic workstation on Windows 11 required solving several low-level OS, memory, and runtime bottlenecks:

* **Eliminated Electron Memory Bloat (~1.8 GB → ~35 MB RAM):** Replaced heavy Chromium/Node desktop wrappers with a native Rust Tauri v2 + WebView2 host (`Subsystem 2 GUI`), reclaiming nearly 2 GB of system memory for model weights and KV caches while decoupling completely from legacy ports (`:47831`).
* **Resolved Windows Startup Access Violations (`0xC0000005`) & Nested Job Conflicts:** Fixed startup crashes by removing recursive host self-containment (`shelter_self`), implementing dynamic WebView2 runtime discovery without hardcoded versions, deferring DWM caption manipulation until window readiness, and adding a `SetPriorityClass` fallback to bypass `ERROR_INVALID_PARAMETER (87)` when launched inside restricted parent Job Objects.
* **Amputated Slow Python Sidecars & Port Sprawl:** Eliminated external Python HTTP sidecars (`:47937`, `:47936`) and legacy `.py` scripts (`command_runner.py`, `doc_extract.py`, `exa_search.py`, `scrapling_reader.py`), moving execution directly into native Rust (~1.4 ms ZIP/XML document extraction, ~20 ms process spawning).
* **Prevented Zombie Subprocess Trees & Hanging Commands:** Bound agent shell execution to Win32 `Job Objects` (`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`) with strict 30-second timeouts and forced process-tree termination (`taskkill /T /F`), preventing runaway child processes.
* **Eliminated WMI / PowerShell Telemetry Stalls:** Replaced slow WMI/PowerShell hardware queries with sub-millisecond native Win32 and GPU driver calls (`GlobalMemoryStatusEx`, `GetSystemTimes`, and `NVML` with a `DXGI` fallback).
* **Solved Cross-Platform Search & Path Traversal Edge Cases:** Fixed `ripgrep` case-insensitivity and Windows drive-letter colon parsing (`C:\...`), normalized extended-length `\\?\` paths, and enforced two-layer Path Traversal protection (`canonicalize` + strict allowlist validation) for on-demand skill loading.
* **Balanced Reasoning Depth vs. KV-Cache Latency:** Built a `<0.1 ms` deterministic `Auto` Thinking Orchestrator (`classify_thinking_effort`) that dynamically scales reasoning effort based on prompt complexity and automatically drops to `low` effort during multi-step tool loops.
* **Mitigated Refusal Loops & Multi-Instance Conflicts:** Implemented monolithic `Frame 0` prompt grounding, two-pass `Stealth Alignment Bypass` (`logit_bias` + syntax-locked prefill), Win32 `Single-Instance Mutex` window restoration, `Tray-only exit` background persistence, and a global `Panic Hook` (`crash.log`).

---

## ✅ What Is Implemented

### 1. Native Rust Core (`src-tauri/`)
* **Modular Backend Architecture:** Refactored from a 6,700-line monolith into a clean, modular structure (`src-tauri/src/api/`: `mod.rs`, `prompt.rs`, `tools.rs`, `extract.rs`, `web.rs`, `tests.rs` alongside `main.rs`, `win.rs`, `engine.rs`, `compat.rs`, `hardware.rs`, `desktop_vision.rs`) backed by **154 passing unit tests** and zero FFI signature warnings.
* **11 Built-In Native Agent Tools:**
  * `execute_command` — Async `tokio::process::Command` execution inside Win32 Job Objects with 30-second timeout and process-tree cleanup.
  * `read_slice` — Streaming line-range file reader (`BufReader`, up to 2,000 lines) to prevent context window flooding.
  * `fast_search` & `file_search` (`POST /api/search`) — High-speed codebase search powered by `ripgrep` with a native fallback scanner and strict execution time budgets.
  * `read_skill` — On-demand Markdown engineering skill loader (`rust_win32.md`, `win_internals.md`) with Path Traversal guards.
  * `doc_extract` (`POST /api/extract`) — Pure-Rust ZIP/XML parser for `DOCX`, `XLSX`, and `TXT` built on `flate2` (~1.4 ms per file).
  * `web_search` & `web_read_page` — Native HTTPS search (Exa API) and HTML-to-text extraction with SSRF protection blocking private IP ranges and unsafe redirects.
  * `desktop_inspect` & `desktop_action` — Win32 UI inspection and input synthesis (`EnumWindows`, `EnumChildWindows`, `SendInput`).
  * `voice_transcribe` — Local speech-to-text integration via `whisper-cli.exe` (`AVX2`).

### 2. Model-Agnostic Inference & Memory Orchestration
* **Universal MoE & External Provider Support (`Dual Backend`):**
  * **Embedded Mode:** Runs any supported Sparse MoE or dense model locally via the C++/Python inference engine (`:8080`), dynamically partitioning weights across **VRAM** (dense layers + compressed `k8v4` KV-cache), **System RAM** (resident hot-expert pool), and **NVMe SSD** (`mmap` cold-expert streaming) with speculative `MTP` drafting. *(Reference benchmark: 34–50+ tok/s decode and ~200 tok/s prefill on an 80.7 GB 125B Sparse MoE model using a 16 GB VRAM GPU + 32 GB DDR4).*
  * **External Provider Mode:** Seamlessly connects to any OpenAI-compatible server (`llama.cpp`, `Ollama`, `vLLM`, `LM Studio`) with persistent cold-start state recovery (`engine_backend.json`).
* **Prompt & Telemetry Controls:** Monolithic `Frame 0` system architecture, `Stealth Alignment Bypass`, and live `x-unstrata-thinking-level` response headers.

---

## 💻 System Requirements (Recommended)

* **OS:** Windows 11 x64 (with Microsoft Edge WebView2 Runtime)
* **CPU:** x86_64 processor with `AVX2` support (8+ cores recommended for MoE expert routing and CPU audio)
* **GPU:** NVIDIA GPU with 12–16+ GB VRAM for heavy local MoE/dense inference (or any GPU / CPU setup when using External Provider mode)
* **RAM & Storage:** 32+ GB DDR4/DDR5 and a fast PCIe NVMe SSD (with system-managed or ≥32 GB Windows pagefile for large `mmap` model weights)

---

## 🚧 What Is Not Yet Implemented (Roadmap)

### Phase 1: Memory Margin Tuning & Clean Workspace Migration *(In Progress)*
* Raising `OS_RESERVE_GIB` (`6.0 → 10.0`) and setting adaptive VRAM reserves (`--vram-reserve-mib 2048` for ≥12 GB GPUs) to prevent swap and Commit Charge spikes under heavy `mmap` workloads.
* Migrating to a short ASCII root path (`C:\UnStrata_V2`) with externalized model weights linked via NTFS Directory Junctions (`mklink /J`) and clean Git initialization.

### Phase 2: Autonomous Dev Harness & Security Hardening
* **Strict CSRF & IPC Token Firewall:** Enforcing `Origin` + `OPTIONS` preflight checks across all `/api/tools/*`, `/api/extract`, `/api/vision/*`, and `/api/desktop/*` routes, paired with a per-boot `X-UnStrata-Token` delivered exclusively via Tauri IPC.
* **Command Guard & Encoding Hardening:** Locking `allow_any_cwd` against request overrides, normalizing `cmd.exe` caret (`^`) escapes and quotes before regex filtering, blocking encoded PowerShell (`-e`/`-ec`/`-enc`) and destructive `robocopy /MIR /purge` flags, protecting host/engine PIDs from `taskkill`, capping `TOOLS_JOB` active processes (`32`), and decoding Windows `CP866`/`CP1251` console output cleanly.
* **KV-Cache Preservation & Context Tools:** Keeping `Frame 0` byte-identical across sessions (`--prompt-cache` optimization) with one-time `[MEMORY]` injection, separating UI `<think>` stripping from raw engine history, adding a lexical `file_outline` signature scanner, tracking external file `mtime` edits without OS folder locks, stripping `\\?\` prefixes before C++ CLIs, and writing masked tool traces to `Release/data/logs/audit.jsonl` (excluded from search).
* **Self-Restart & Runtime Safeguards:** Out-of-job `.exe` self-updater for autonomous Rust rebuilds (`CARGO_BUILD_JOBS=2`), `SendInput` Focus Guard (`GetForegroundWindow`), global emergency `RegisterHotKey` modifier reset, `SetThreadExecutionState` lock-screen prevention during active agent loops, immediate SSE cancellation drop, and tool-loop step/cycle breakers.

### Phase 3: Native Vision Loop (`visual_inspect`)
* **Zero-Bloat Window Capture:** Asynchronous `ICoreWebView2::CapturePreview` (STA → `oneshot`) with `devicePixelRatio` scaling, CSS animation settling (`document.getAnimations()`), local `Pixel-Diff` gating, and sub-millisecond deterministic `3×3`/`5×5` median `Pixel-Assert` color/contrast verification.
* **Optional Multi-Key Vision Proxy:** Configurable rotation across up to 10 VLM endpoints (Gemini, OpenRouter, or local custom servers) with `Per-Provider Circuit Breakers` and exponential backoff.

### Phase 4: Clean Prefab Frontend (`ui-kit`) & Native CPU Audio
* **Full UI Replacement:** Removing the legacy minified React bundle and CSS patch layer in favor of a lightweight, schema-driven component frontend (`Preact/React + Tailwind + ui-kit` prefabs: `kitCard`, `kitRow`, `kitToggle`, `kitSelect`, `kitSlider`, `kitButton`) with live `:8080/metrics` throughput telemetry.
* **100% Python-Free Voice:** Replacing the final voice bridge (`:47932`) with native `whisper-rs` (`whisper.cpp`) pinned strictly to CPU `AVX2` so 100% of GPU VRAM remains dedicated to MoE inference.
