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

## ⚡ Hardware Tiers & Throughput Calculations

Because Sparse MoE inference during token generation (decode) is **memory-bandwidth-bound** rather than compute-bound, `UnStrata_V2` maximizes throughput by matching active parameter reads (`~6B` active weights per token, `~1.0 GB` in `TQ1_0` quantization) to the fastest available memory bus and amplifying decode speed via **Multi-Token Prediction (MTP)** speculative drafting (`--spec 6`):

* **Tier 1 — VRAM (`288–1,792 GB/s`):** Holds dense/shared layers and compressed `k8v4` KV-cache (`~2.3 ms` per pass on a `288 GB/s` bus).
* **Tier 2 — System RAM (`51.2–96.0 GB/s`):** Pins hot routed experts (`~6–8 ms` per pass on dual-channel DDR4-3200).
* **Tier 3 — NVMe `mmap` (`3.4–7.4 GB/s`):** Streams only cold tail experts (`<15%` Zipfian miss rate), while MTP draft verification validates up to 6 tokens in a single parallel pass (**1.8×–2.5×** effective speedup).

### Recommended Configurations & Expected Performance

| Hardware Tier | Bus & Memory Specs | Recommended Engine Flags | Target Model Scale | Expected Decode |
| :--- | :--- | :--- | :--- | :--- |
| **Entry (12 GB VRAM)**<br>RTX 3060 12GB + 32 GB DDR4-3200 | `360 GB/s` GDDR6<br>`51.2 GB/s` DDR4<br>PCIe 4.0 x16 | `--resident-budget-gib 18.0`<br>`--vram-reserve-mib 1536`<br>`--kv k8v4 --kv-resident 16384`<br>`--spec 4` | MoE `35B–80B` (`TQ1_0`/`IQ2`)<br>or Dense `14B` (`Q4_K_M`) | **18–28 tok/s** (MoE)<br>**25–36 tok/s** (14B) |
| **Developer Reference (16 GB VRAM)**<br>RTX 4060 Ti 16GB + Ryzen 7 5700X + 32 GB DDR4-3200 + NVMe | `288 GB/s` GDDR6<br>`51.2 GB/s` DDR4<br>PCIe 3.0 x8 (`~6.5 GB/s`) | `--resident-budget-gib 20.0`<br>`--vram-reserve-mib 2048`<br>`--kv k8v4 --kv-resident 32768`<br>`--pool-workers 6 --spec 6` | Sparse MoE `80B–125B`<br>(`80.7 GB` weights + MTP draft) | **30–50+ tok/s** decode<br>**~200 tok/s** prefill |
| **Performance 16 GB (GDDR6X/GDDR7)**<br>RTX 4070 Ti Super / 5070 Ti / 5080 + 64 GB DDR5-6000 | `672–960 GB/s` VRAM<br>`96.0 GB/s` DDR5<br>PCIe 4.0/5.0 x16 | `--resident-budget-gib 48.0`<br>`--vram-reserve-mib 2048`<br>`--kv k8v4 --kv-resident 65536`<br>`--pool-workers 8 --spec 6` | MoE `100B–140B` (`Q3`/`Q4_K_M`)<br>or Dense `24B–27B` | **45–75 tok/s** (MoE)<br>**60–90 tok/s** (24B) |
| **Workstation (24–32 GB VRAM)**<br>RTX 3090 / 4090 (24GB) or RTX 5090 (32GB) + 64–128 GB DDR5 | `936–1,792 GB/s` VRAM<br>`96.0 GB/s` DDR5<br>PCIe 4.0/5.0 x16 | `--resident-budget-gib 80.0`<br>`--vram-reserve-mib 2560`<br>`--kv q8_0 --kv-resident 131072`<br>`--pool-workers 10 --spec 8` | Large MoE `120B–235B`<br>or Dense `32B–70B` | **55–110+ tok/s** (MoE)<br>**50–85 tok/s** (32B) |


