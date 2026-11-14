# UnStrata_V2

**A sovereign, open-source local AI workstation and autonomous agent host for Windows 11 — engineered to run 125B+ Sparse Mixture-of-Experts (MoE) and dense models on consumer hardware with zero cloud telemetry and ~35 MB idle RAM.**

`UnStrata_V2` replaces bloated Electron/Chromium wrappers (~1.8 GB RAM) with a native **Rust (`Tauri v2` + `WebView2`, Windows Subsystem 2 GUI)** host bound to `127.0.0.1:47931`. It orchestrates multi-tier **VRAM / System RAM / NVMe `mmap`** offloading alongside a zero-trust Win32 execution sandbox, a native DOM + pixel vision inspector (`ui_map` / `visual_inspect`), and deterministic verification tools so an autonomous coding agent can inspect, edit, rebuild, and restart itself safely on a single consumer GPU.

---

## 📊 Current Project Status (~70% Complete)

```text
Overall Progress:              [██████████████░░░░░░] ~70%

Native Rust Core & IPC:        [██████████████████░░]  92%  (190 unit tests passing, 15 native tools, zero-trust firewall)
Native Vision & Layout Loop:   [██████████████████░░]  90%  (CapturePreview + PrintWindow + dpr scaling + ui_map + Vision Proxy)
Security & Win32 Sandboxing:   [██████████████████░░]  90%  (CSRF/Token firewall, Handle-List isolation, NTFS reparse/ADS guards)
Engine & Memory Tiering:       [█████████████████░░░]  88%  (3-tier MoE offload, OS/VRAM margins, slot drain, :8080 guard)
Frontend UI (Vanilla ES):      [█████░░░░░░░░░░░░░░░]  25%  (Legacy patch layer active; zero-bundler Vanilla ES kit next)
Native CPU Audio (Whisper):    [████░░░░░░░░░░░░░░░░]  20%  (External bridge active; in-process CPU AVX2 FFI + Worklet next)
Verify Coprocessor & KV Opts:  [███░░░░░░░░░░░░░░░░░]  15%  (verify_tools.rs, output spooling & anchored compaction next)
```

---

## 🏗️ System Architecture

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ Frontend UI (WebView2 / Modular Vanilla ES + CSS Tokens, No Bundler)        │
│ • Strict CSP (script-src 'self') + synchronous boot-guard.js error trap     │
│ • rAF top-level block freezing (Markdown/KaTeX) + content-visibility: auto  │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ Loopback HTTP/SSE (:47931) + Tauri IPC
                                       │ (Origin/Host Firewall + X-UnStrata-Token)
┌──────────────────────────────────────▼──────────────────────────────────────┐
│ Native Rust Host (src-tauri/, ~35 MB idle RAM, zero bloat dependencies)     │
│ • Byte-Stable Frame 0 & <0.1 ms Auto Thinking Classifier                    │
│ • 15 Native Tools (FS / AST Outline / Search / Web / Vision / OS / Docs)    │
│ • Hybrid Vision Loop: STA CapturePreview + PrintWindow + ui_map + Pixel-Diff │
│ • Win32 Job Objects, Handle-List Spawning, Audit Logger & Out-of-Job Updater │
└──────────────────┬───────────────────────────────────────┬──────────────────┘
                   │ Embedded Mode (:8080)                 │ External Mode
┌──────────────────▼───────────────────────┐   ┌───────────▼──────────────────┐
│ Local Inference Engine (strata.exe)      │   │ OpenAI-Compatible Endpoints  │
│ • Tier 1 VRAM: Dense layers + k8v4 KV    │   │ • llama.cpp / Ollama         │
│ • Tier 2 RAM: ~20 GB resident hot experts│   │ • vLLM / LM Studio / Cloud   │
│ • Tier 3 NVMe: mmap cold expert streaming│   └──────────────────────────────┘
│ • Speculative MTP drafting (--spec 6)    │
└──────────────────────────────────────────┘
```

---

## 📁 Isolated Project Layout (`C:\UnStrata_V2`)

Model weights (`80.7 GB` GGUF) are externalized outside the Git repository and linked into `Release\models` via an NTFS Directory Junction (`mklink /J`) with zero disk duplication:

```text
C:\UnStrata_Models\                     # Externalized 80.7 GB LLM weights (outside Git, attrib +I)
└── whisper\                            # Isolated ggml-*.bin CPU speech models

C:\UnStrata_V2\                         # Clean Git root (attrib +I, Defender excluded)
├── .cargo\config.toml                  # [build] jobs = 2 (prevents rustc from evicting RAM experts)
├── finish_migration.ps1                # Moves weights to C:\UnStrata_Models & links NTFS junction
├── src-tauri\
│   ├── Cargo.toml                      # Minimal dependencies; Win32 FFI via extern "system"
│   ├── capabilities\default.json       # Tauri v2 IPC permissions for local WebView2
│   └── src\
│       ├── main.rs                     # Single-instance mutex, :47931 bind, tray, out-of-job updater
│       ├── engine.rs                   # RAM/VRAM margins, --prompt-cache 2, :8080 guard, Prefetch
│       ├── compat.rs                   # 15-tool registry, KV estimator, /api/models filtering
│       ├── hardware.rs                 # Sub-ms CPU/RAM/GPU/VRAM telemetry (NVML + DXGI fallback)
│       ├── desktop_vision.rs           # SendInput Focus Guard, privileged blacklist, emergency hotkey
│       ├── pixel.rs                    # WebView2 CapturePreview + PrintWindow + native flate2 PNG
│       ├── vision.rs                   # ui_map (650 ms), ui_zoom (+64 px), 10-slot Vision Proxy
│       └── api\
│           ├── mod.rs                  # Origin/Host/Token firewall, SSE keepalive/abort, audit.jsonl
│           ├── tools.rs                # Handle-List shell, CP866/UTF-16 decoder, NTFS guards, spooling
│           ├── prompt.rs               # Byte-stable Frame 0, THINK_KEEP_TURNS=7, Anchored Compaction
│           ├── verify_tools.rs         # [Planned] compute_verify, data_probe, sys_probe
│           └── tests.rs                # 190+ offline unit tests
└── Release\
    ├── unstrata.exe                    # Native Rust GUI host (~35 MB idle RAM)
    ├── models\                         # NTFS Directory Junction -> C:\UnStrata_Models
    ├── engine\ & serve\                # Inference runtime (:8080)
    ├── data\                           # Configs, sessions (.jsonl), skills, logs (audit.jsonl/crash.log)
    └── ui\                             # Modular Vanilla ES frontend (no Vite/Webpack/node_modules)
```

---

## ⚙️ Core Mechanics & Subsystems

### 1. Three-Tier MoE Memory Orchestration & Slot Synchronization
Running a **125B Sparse MoE model (`80.7 GB` GGUF weights)** on a 16 GB GPU and 32 GB system RAM requires deterministic memory budgeting so Windows 11 never thrashes the NVMe pagefile:
* **Dynamic OS & VRAM Headroom (`engine.rs`):** Enforces `OS_RESERVE_GIB = 10.0` via `GlobalMemoryStatusEx` (locking `~20.0 GiB` for hot routed MoE experts in DDR4 while leaving 10+ GiB for Windows, WebView2, and Rust builds) and sets `--vram-reserve-mib 2048` on `>= 12 GiB` GPUs (`1536` on `< 12 GiB`) to protect DWM and `CapturePreview` surfaces.
* **KV-Cache Slot Capping (`--prompt-cache 2`):** Restricts KV cache allocation on `<= 32 GiB` RAM systems to 2 slots (1 active conversation + 1 service slot), preventing 4-slot KV allocations from evicting resident MoE experts back to SSD.
* **Duplicate-Spawn Protection & Slot Drain:** Probes `127.0.0.1:8080` via `GetExtendedTcpTable` before spawning `strata.exe`. If an instance from the project directory is already serving `/v1/models`, the host attaches cleanly without duplicating 80.7 GB in memory. During cold long-context `prefill` (`25–45 s` to first token), Rust emits `: keepalive\n\n` SSE frames every `5 s` and marks cancelled slots (`cancelling`) on `AbortController` disconnect to prevent `BDONE` slot deadlocks on rapid retries.

### 2. Byte-Stable `Frame 0`, `Auto` Thinking & Dual-Layer History
* **Zero-Invalidation Prompt Caching (`prompt.rs`):** System instructions and all tool JSON schemas are serialized deterministically using `BTreeMap` and sorted vectors with normalized `\n` line endings. Dynamic `[MEMORY]` blocks and timestamps are kept out of `Frame 0` and injected into the first `user` turn so `--prompt-cache 2` survives across sessions.
* **Dual-Layer Conversation State:** The UI renders clean assistant responses without `<think>` clutter, while the engine receives raw `<think>` blocks inside a sliding window (`THINK_KEEP_TURNS = 7`).
* **Deterministic `<0.1 ms` Thinking Router:** Classifies prompt complexity on the fly (`low` / `medium` / `high`) and automatically throttles reasoning effort to `low` during multi-step tool-use loops to maximize KV throughput.
* **Embedded vs. External Mode Parity:** Automatically disables `refusal_bias` and assistant syntax-locked prefill when connected to standard OpenAI-compatible external endpoints.

### 3. Hybrid Native Vision & Layout Inspector (`desktop_vision.rs`, `pixel.rs`, `vision.rs`)
Designed so the text-only 125B model can visually inspect and debug its own UI without burning context tokens on raw base64 images:
* **Zero-Bloat Capture Pipeline:** Captures the WebView2 surface asynchronously via `ICoreWebView2::CapturePreview` (dispatched on the UI STA thread, rewound via `IStream::Seek(0)`, and awaited via `oneshot` with an `800 ms` timeout), with a Win32 `PrintWindow(hwnd, hdc, PW_CLIENTONLY | PW_RENDERFULLCONTENT)` fallback for external windows.
* **DPI-Aware Coordinate & Color Verification:** Scales logical CSS rects from `ui_map` by `window.devicePixelRatio` (`dpr`) before cropping physical frames, encodes PNGs natively via `flate2` (`ZlibEncoder` + `Crc`), computes localized `Pixel-Diff` percentages, and runs deterministic `3×3` / `5×5` median `Pixel-Assert` checks in linearized sRGB. Never injects `data:image/png;base64,...` into LLM context.
* **Semantic DOM/CSS Inspector (`ui_map` & `ui_zoom`):** Waits for active CSS animations (`document.getAnimations()`), inspects the live DOM/CSSOM in `~150–250` tokens, and flags `CLIPPED_*`, `OVERLAP` (`> 1.5 CSS px`, z-index and ancestor aware), `ZERO_HEIGHT`, `TRUNCATED_TEXT`, `VERTICAL_CRUSH`, `OCCLUDED_CENTER`, `CONTRAST_FAIL`, and `THEME_SURFACE_MISMATCH`.
* **Optional Multi-Key `Vision Proxy`:** Supports up to 10 user-configured VLM slots (`Gemini`, `OpenRouter`, or custom OpenAI-compatible endpoints, disabled by default) with `+64 px` contextual padding, `Per-Provider Circuit Breakers`, exponential backoff, and strict key redaction.

### 4. Zero-Trust Loopback Firewall, Win32 Isolation & Self-Update Lifecycle
* **IPC & CSRF Firewall (`api/mod.rs`):** Validates `Origin` (`http://127.0.0.1:47931`, `http://tauri.localhost`, `https://tauri.localhost`), `Host`, and `OPTIONS` preflight headers, enforcing a cryptographic per-boot `X-UnStrata-Token` (delivered exclusively via Tauri IPC `get_boot_token`) on all `/api/*`, `/v1/*`, and mutating routes.
* **Hardened Shell & Process Isolation (`api/tools.rs`, `engine.rs`):**
  * Normalizes `cmd.exe` caret (`^`) escapes and quotes before deny-list inspection; blocks encoded PowerShell (`-e`/`-ec`/`-enc`), destructive `robocopy /MIR /purge`, `rd /s`, `format`, and `taskkill` targeting host/engine PIDs.
  * Spawns subprocesses via `STARTUPINFOEXW` + `PROC_THREAD_ATTRIBUTE_HANDLE_LIST` (preventing child processes from inheriting the `:47931` listener socket) inside `TOOLS_JOB` (`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE | JOB_OBJECT_LIMIT_ACTIVE_PROCESS = 32`).
  * Passes `cmd.exe /d /c` commands via `CommandExt::raw_arg` (`chcp 65001 >nul 2>nul & <cmd>`) and decodes `UTF-16` BOM, `UTF-8`, `CP866`, and `CP1251` console streams cleanly.
* **NTFS Traversal & Reparse Guards (`resolve_tool_path`):** Walks path components top-down via `symlink_metadata()`, rejecting symlinks, NTFS Junctions (such as `Release/models`), Alternate Data Streams (`:`), trailing dots/spaces, and 8.3 short-name bypasses (`GetLongPathNameW`).
* **Non-Blocking Redacted Audit Trail:** Streams tool calls and HTTP routes over a bounded `mpsc` channel (`1024` capacity) to `Release/data/logs/audit.jsonl` (`10 MB` auto-rotation), masking API keys (`sk-`, `AIza`, `Bearer`, `?key=`, `-----BEGIN`).
* **Crash-Safe Lifecycle & Self-Updater (`main.rs`):** Protects single-instance execution via `Local\UnStrataV2_Master_Instance_Mutex`, cleans orphaned `EBWebView/LOCK` processes prior to window creation, prevents system sleep via `SetThreadExecutionState` during active agent loops, captures full backtraces to `crash.log` via `std::panic::set_hook`, and spawns an out-of-job micro-updater (`--wait-pid`) to swap `unstrata.exe` -> `unstrata.exe.old` autonomously.

---

## 🧰 Built-In Native Agent Tools

### Currently Implemented (15 Tools)

| Tool | Mechanism | Purpose & Guardrails |
| :--- | :--- | :--- |
| `execute_command` | `STARTUPINFOEXW` + `TOOLS_JOB` | Async shell execution (`<= 30 s`), `CancelIoEx`, `CP866`/`CP1251` decoding, caret/quote deny-list |
| `read_slice` | `BufReader` streaming | Line-range file reader with pagination (`<= 250` lines / `16 KB` cap) |
| `file_outline` | Lexical Rust/JS/Py scanner | Extracts structs, traits, functions, and signatures without reading full files |
| `write_file` / `edit_file` | Atomic `.tmp` + `sync_all` | UTF-8/BOM/CRLF-preserving file creation and exact-match patching with `mtime` guard |
| `fast_search` | `ripgrep` + native fallback | High-speed codebase search with `--` arg barrier and `logs/`/`temp/` exclusions |
| `file_search` | Directory walker (`POST /api/search`) | Fast filename discovery respecting NTFS reparse guards |
| `read_skill` | Allowlisted Markdown loader | On-demand engineering runbooks (`rust_win32.md`, `win_internals.md`, `ui_layout.md`) |
| `doc_extract` | Pure-Rust `flate2` ZIP/XML | `~1.4 ms` text extraction from `.docx`, `.xlsx`, and `.txt` |
| `web_search` | HTTPS Exa API client | Web search with SSRF private-IP blocking and strict per-request timeouts |
| `web_read_page` | Native HTML-to-text parser | Clean webpage extraction with redirect validation and secret redaction |
| `visual_inspect` | `CapturePreview` + `PrintWindow` | DPI-scaled screenshot crop, `Pixel-Diff`, `3×3`/`5×5` `Pixel-Assert`, optional VLM proxy |
| `ui_map` | WebView2 DOM/CSSOM bridge | `~150–250` token layout audit (`OVERLAP`, `CLIPPED_*`, `CONTRAST_FAIL`, `OCCLUDED_CENTER`) |
| `ui_zoom` | Sub-rect DOM + `Pixel-Assert` | High-detail CSS/geometry inspection inside a target bounding box |
| `desktop_inspect` | `EnumWindows` / `EnumChildWindows` | External Win32 window hierarchy and signed `i32` multi-monitor coordinates |
| `desktop_action` | `SendInput` + `Focus Guard` | Foreground-verified mouse/keyboard automation with privileged-process blacklist |

### Planned Deterministic Coprocessor (3 Multi-Tools)

| Tool | Operations | Purpose & Guardrails |
| :--- | :--- | :--- |
| `compute_verify` | `calc`, `time`, `hash`, `json_check`, `regex_test` | Overflow-checked `i128` / IEEE-754 math, streaming `CRC32`/`SHA-256` (`<= 128 MiB`), bracket/JSON audit |
| `data_probe` | `summary`, `filter`, `aggregate`, `top_n` | Streaming `.jsonl`/`.json`/`.csv` analyzer (`<= 32 MiB`, `2000 ms` budget, `<= 512 B` field clamp) |
| `sys_probe` | `ports`, `processes` | Native `AF_INET`/`AF_INET6` TCP table (`GetExtendedTcpTable`) & Working Set process snapshot |

---

## 🛠️ Key Engineering Problems Already Solved

* **Eliminated Electron Memory Bloat (`~1.8 GB → ~35 MB RAM`):** Reclaimed nearly 2 GB of system memory for model weights and KV caches while decoupling completely from legacy ports (`:47831`).
* **Resolved Windows Startup Access Violations (`0xC0000005`) & Nested Job Conflicts:** Removed recursive host self-containment (`shelter_self`), added dynamic WebView2 runtime discovery, deferred DWM caption styling until window readiness, and added a `SetPriorityClass` fallback against `ERROR_INVALID_PARAMETER (87)`.
* **Amputated Slow Python Sidecars:** Eliminated external HTTP sidecars (`:47937`, `:47936`) for command execution, document parsing, search, and scraping in favor of sub-millisecond native Rust handlers.
* **Fixed High-DPI (`125%` / `150%`) Vision Coordinate Drift:** Separated `client_origin` from DWM shadow bounds (`-7 px`) and scaled logical CSS coordinates by `dpr` prior to physical frame cropping.
* **Eliminated WMI / PowerShell Telemetry Stalls:** Replaced slow WMI queries with sub-millisecond Win32/driver calls (`GlobalMemoryStatusEx`, `GetSystemTimes`, `NVML` + `DXGI`).

---

## 💻 System Requirements (Recommended)

* **OS:** Windows 11 x64 (with Microsoft Edge WebView2 Runtime)
* **CPU:** x86_64 processor with `AVX2` support (8+ cores recommended for MoE expert routing and CPU audio)
* **GPU:** NVIDIA GPU with 12–16+ GB VRAM for heavy local MoE/dense inference (or any GPU / CPU setup when using External Provider mode)
* **RAM & Storage:** 32+ GB DDR4/DDR5 and a fast PCIe NVMe SSD (with system-managed or `>= 32–48 GB` Windows pagefile for `80+ GB` `mmap` model weights)

---

## 🚧 Roadmap: What Remains (~30%)

### Phase 1: Isolated Root Finalization (`C:\UnStrata_V2`) *(Finalizing)*
* Execute `finish_migration.ps1` to move `80.7 GB` of GGUF weights into `C:\UnStrata_Models` and mount `C:\UnStrata_V2\Release\models` via NTFS Junction (`mklink /J`) with Windows Search indexing disabled (`attrib +I`).

### Phase 2: Zero-Bundler Vanilla ES Frontend (`Release/ui/`)
* Replace the legacy minified React bundle and CSS patch layer with modular ES modules (`tokens.css`, `ui-kit.css`, `chat.css`, `boot-guard.js`, `stream.js`, `monitor.js`) under a strict `script-src 'self'` CSP, `rAF` top-level block freezing (preserving `<ol>` numbering and `KaTeX`), `content-visibility: auto`, and live `:8080` metrics polling.

### Phase 3: Native CPU `AVX2` Speech-to-Text (`whisper.dll` FFI)
* Replace the final Python audio bridge (`:47932`) with in-process `whisper.dll` (`AVX2`, strictly CPU-pinned so 0 MB of VRAM is taken from MoE weights), capturing native WASAPI audio via `AudioWorkletNode`, resampling in Rust to `16 kHz` with DC-blocker + subnormal flush + RMS VAD, and supporting `ru | uk | en | auto` switching.

### Phase 4: Analytical Coprocessor & KV-Cache Optimizations
* Implement `verify_tools.rs` (`compute_verify`, `data_probe`, `sys_probe`), one-shot `Output Spooling` (`<= 4 KiB` deduplicated previews), parallel read-only `tokio::JoinSet` batching, atomic multi-hunk `multi_edit`, targeted `PrefetchVirtualMemory` for `*.gguf` headers/tails, and two-stage `Anchored Compaction` (`75%` turn-boundary / `80%` mid-tool-loop).

### Phase 5: 5-Stage End-to-End Verification Gate
* Validate offline unit tests (`cargo test --offline`), cold `:8080` startup & prefill `Abort` recovery, `unstrata.exe` self-restart with `EBWebView/LOCK` verification, multi-theme `ui_map` + `Pixel-Assert` (`960px`/`1440px`, `anomalies: []`), and end-to-end tool + voice audit logging.

---

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
| **Performance 16 GB (GDDR6X/GDDR7)**<br>RTX 4070 Ti Super / 5070 Ti / 5080 + 64 GB DDR5-6000 | `672–960 GB/s` VRAM<br>`96.0 GB/s` DDR5<br>PCIe 4.0/5.0 x16 | `--resident-budget-gib 48.0`<br>`--vram-reserve-mib 2048`<br>`--kv k8v4 --kv-resident 65536`<br>`--pool-workers 8 --spec 6` | MoE `100B–140B` (`Q3`/`Q4_K_M`)<br>or Dense `22B–24B` (`Q4_K_M`) | **45–75 tok/s** (MoE)<br>**48–85 tok/s** (24B) |
| **Workstation (24–32 GB VRAM)**<br>RTX 3090 / 4090 (24GB) or RTX 5090 (32GB) + 96–128 GB DDR5 | `936–1,792 GB/s` VRAM<br>`96.0 GB/s` DDR5<br>PCIe 4.0/5.0 x16 | `--resident-budget-gib 80.0`<br>`--vram-reserve-mib 2560`<br>`--kv q8_0 --kv-resident 131072`<br>`--pool-workers 10 --spec 8` | Large MoE `120B–235B`<br>or Dense `32B–70B` | **55–110+ tok/s** (MoE)<br>**50–85 tok/s** (32B) |
