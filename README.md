# UnStrata

**A sovereign, open-source local AI workstation and autonomous agent host for Windows 11 — engineered to run 80B–120B Sparse Mixture-of-Experts (MoE) and dense models on consumer hardware with zero cloud telemetry and ~35 MB idle RAM.**

`UnStrata` replaces bloated Electron/Chromium wrappers (~1.8 GB RAM) with a native **Rust (`Tauri v2` + `WebView2`, Windows Subsystem GUI)** host bound to `127.0.0.1:47931`. It orchestrates multi-tier **VRAM / System RAM / NVMe `mmap`** offloading alongside a zero-trust Win32 execution sandbox, a native DOM + pixel vision inspector (`ui_map` / `visual_inspect`), a Rust-backed Theme & Glass Engine, and deterministic verification tools so an autonomous coding agent can inspect, edit, rebuild, and restart itself safely on a single consumer GPU.

---

## Current Project Status (~82% Complete)

    Overall Progress:              [████████████████░░░░] ~82%

    Native Rust Core & IPC:        [██████████████████░░]  92%  (190+ unit tests passing, 15 native tools, zero-trust firewall)
    Native Vision & Layout Loop:   [██████████████████░░]  90%  (CapturePreview + PrintWindow + dpr scaling + ui_map + Vision Proxy)
    Security & Win32 Sandboxing:   [██████████████████░░]  90%  (CSRF/Token firewall, Handle-List isolation, NTFS reparse/ADS guards)
    Engine & Memory Tiering:       [██████████████████░░]  90%  (3-tier MoE offload, VRAM/RAM margins, slot drain, :8080 guard)
    Frontend UI (Vanilla ES):      [█████████████████░░░]  85%  (Zero-bundler Vanilla ES + CSS tokens; Monochrome Frosted Glass & Theme Studio)
    Native CPU Audio (Whisper):    [████░░░░░░░░░░░░░░░░]  20%  (External bridge active; in-process CPU AVX2 FFI + Worklet next)
    Verify Coprocessor & KV Opts:  [███░░░░░░░░░░░░░░░░░]  15%  (verify_tools.rs, output spooling & anchored compaction next)

---

## System Architecture

    +-----------------------------------------------------------------------------+
    | Frontend UI (WebView2 / Modular Vanilla ES + CSS Tokens, Zero Bundler)      |
    | - Single-frame theme painting + procedural 8-orb ambient stage              |
    | - Cache-first instant view switching + rAF Markdown/KaTeX rendering         |
    +--------------------------------------+--------------------------------------+
                                           | Loopback HTTP/SSE (:47931) + Tauri IPC
                                           | (Origin/Host Firewall + Session Token)
    +--------------------------------------v--------------------------------------+
    | Native Rust Host (src-tauri/, ~35 MB idle RAM, zero bloat dependencies)     |
    | - Byte-Stable Frame 0 & <0.1 ms Auto Thinking Classifier                    |
    | - 15 Native Tools (FS / AST Outline / Search / Web / Vision / OS / Docs)    |
    | - Hybrid Vision Loop: STA CapturePreview + PrintWindow + ui_map + Pixel-Diff|
    | - Theme & Palette API (/api/themes, /api/palette) with strict validation    |
    | - Win32 Job Objects, Handle-List Spawning, Audit Logger & Out-of-Job Updater|
    +------------------+---------------------------------------+------------------+
                       | Embedded Mode (:8080)                 | External Mode
    +------------------v-----------------------+   +-----------v------------------+
    | Local Inference Engine (strata.exe)      |   | OpenAI-Compatible Endpoints  |
    | - Tier 1 VRAM: Dense layers + KV cache   |   | - llama.cpp / Ollama         |
    | - Tier 2 RAM: Resident hot MoE experts   |   | - vLLM / LM Studio / Cloud   |
    | - Tier 3 NVMe: mmap cold expert streaming|   +------------------------------+
    | - Speculative MTP drafting (--spec 4)    |
    +------------------------------------------+

---

## Project Layout (`C:\UnStrataV2.1`)

Heavy GGUF model weights (`80+ GB`) live outside Git tracking and can be mounted into `Release\models` via an NTFS Directory Junction (`mklink /J`) without hardcoded machine paths:

    C:\UnStrataV2.1\                        # Clean portable Git root (UTF-8 no BOM)
    ├── .cargo\config.toml                  # [build] jobs = 2 (prevents rustc from starving RAM experts)
    ├── src-tauri\
    │   ├── Cargo.toml                      # Minimal dependencies; Win32 FFI via extern "system"
    │   ├── capabilities\default.json       # Tauri v2 IPC permissions for local WebView2
    │   └── src\
    │       ├── main.rs                     # Single-instance mutex, :47931 bind, tray, out-of-job updater
    │       ├── engine.rs                   # RAM/VRAM margins, LoadProfile Job Objects, :8080 guard, Prefetch
    │       ├── compat.rs                   # 15-tool registry, KV estimator, /api/models filtering
    │       ├── hardware.rs                 # Sub-ms CPU/RAM/GPU/VRAM telemetry (NVML + DXGI fallback)
    │       ├── desktop_vision.rs           # SendInput Focus Guard, privileged blacklist, emergency hotkey
    │       ├── pixel.rs                    # WebView2 CapturePreview + PrintWindow + native flate2 PNG
    │       ├── vision.rs                   # ui_map, ui_zoom (+64 px), 10-slot Vision Proxy
    │       └── api\
    │           ├── mod.rs                  # Origin/Host/Token firewall, /api/themes & /api/palette, SSE, audit.jsonl
    │           ├── tools.rs                # Handle-List shell, CP866/UTF-16 decoder, NTFS guards, spooling
    │           ├── prompt.rs               # Byte-stable Frame 0, THINK_KEEP_TURNS=7, Anchored Compaction
    │           ├── verify_tools.rs         # [Planned] compute_verify, data_probe, sys_probe
    │           └── tests.rs                # 190+ offline unit tests
    └── Release\
        ├── unstrata.exe                    # Native Rust GUI host (~35 MB idle RAM)
        ├── models\                         # Local GGUF weights or NTFS Junction (ignored by Git)
        ├── engine\ & serve\                # Inference worker strata.exe (:8080)
        ├── data\                           # Configs, sessions (.jsonl), skills, logs (audit.jsonl/crash.log)
        └── ui\                             # Zero-bundler Vanilla ES frontend (~530 KB total assets)
            ├── index.html                  # SPA entry point, inline SVG sprite (35 glyphs), boot guard
            ├── theme.js                    # Synchronous theme engine, HSL/RGB token generator & 8-orb stage
            ├── unstrata-v2.js              # Wire & SSE transport layer, session recorder, tool gating
            ├── unstrata-tokens.css         # Root geometry, typography (--font-ui/--font-mono) & motion tokens
            ├── unstrata-palette.css        # Base dark/light scheme contracts & quick theme popover styles
            ├── unstrata-ui.css             # Unified frosted-glass layout, chat, monitor & view styles (0 !important)
            ├── themes\                     # JSON theme definitions (amoled-black, amoled-white, espresso + custom)
            └── src\                        # Modular Vanilla ES UI controllers
                ├── native.js               # Hash router (#/chat, #/models, #/monitor, #/skills, #/tasks, #/settings)
                ├── chat-render.js          # Transcript virtualization, collapsible Thinking blocks, KaTeX & code blocks
                ├── composer.js             # Prompt capsule, slash commands, attachments & voice/vision hooks
                ├── inspector.js            # Right RUN SETTINGS drawer, model picker, LoadProfile & sampling sliders
                ├── monitor.js              # KPI cards, smoothed throughput SVG chart, context arc & worker logs
                ├── sidebar.js              # Left navigation rail, pinned chats & history list
                ├── modals.js               # Native dialogs, overlays & popover lifecycle
                ├── shortcuts.js            # Normalized keyboard contract (Enter / Shift+Enter / Escape)
                ├── scheduler.js            # Unified API polling & background tick scheduler
                ├── api-client.js           # Loopback HTTP client & error normalization
                ├── dom.js                  # Safe DOM helpers & layout measurement
                ├── state.js                # Shared reactive UI & engine state
                ├── prefs.js                # Persistent UI preferences
                ├── fonts.js                # Dynamic UI & monospace font switcher (/api/fonts)
                └── sessions.js             # Local session persistence & recovery

---

## Core Mechanics & Subsystems

### 1. Three-Tier MoE Memory Orchestration & Slot Synchronization
Running an **80B–120B Sparse MoE model (`~80.7 GB` GGUF weights)** on a 16 GB GPU and 32–80 GB system RAM requires deterministic memory budgeting so Windows 11 avoids thrashing the NVMe pagefile:
* **Dynamic OS & VRAM Headroom (`engine.rs`):** Enforces `OS_RESERVE_GIB = 10.0` via `GlobalMemoryStatusEx` and reserves `1.5–2.0 GiB` of VRAM (`--vram-reserve-mib 1536..2048`) to protect DWM and WebView2 `CapturePreview` surfaces. Keeping resident VRAM experts + quantized KV cache (`q8_0`/`q4_0`) within `13.5–14.5 GiB` on a 16 GB GPU eliminates PCIe x8 expert misses during decode.
* **Windows Job Object Load Profiles (`Full` / `Background`):** Switches worker priority (`ABOVE_NORMAL` vs. `BELOW_NORMAL` + `JOB_OBJECT_CPU_RATE_CONTROL_HARD_CAP`) on the fly via `POST /api/engine/load-profile` without unloading model weights.
* **Duplicate-Spawn Protection & Slot Drain:** Probes `127.0.0.1:8080` via `GetExtendedTcpTable` before spawning `strata.exe`. If a compatible worker is already serving `/v1/models`, the host attaches cleanly without duplicating weights in RAM. During cold long-context `prefill` (`25–45 s` to first token), Rust emits `: keepalive\n\n` SSE frames every `5 s` and marks cancelled slots (`cancelling`) on `AbortController` disconnect to prevent slot deadlocks on rapid retries.

### 2. Byte-Stable `Frame 0`, `Auto` Thinking & Dual-Layer History
* **Zero-Invalidation Prompt Caching (`prompt.rs`):** System instructions and all tool JSON schemas are serialized deterministically using `BTreeMap` and sorted vectors with normalized `\n` line endings. Dynamic `[MEMORY]` blocks and timestamps are kept out of `Frame 0` and injected into the first `user` turn so KV prompt caches survive across turns.
* **Dual-Layer Conversation State:** The UI renders clean assistant responses with collapsible `Thinking (Xs)` blocks, while the engine receives raw `<think>` blocks inside a sliding window (`THINK_KEEP_TURNS = 7`).
* **Deterministic `<0.1 ms` Thinking Router:** Classifies prompt complexity on the fly (`low` / `medium` / `high`) and automatically throttles reasoning effort to `low` during multi-step tool-use loops to maximize KV throughput.
* **Embedded vs. External Mode Parity:** Automatically disables `refusal_bias` and assistant syntax-locked prefill when connected to standard OpenAI-compatible external endpoints.

### 3. Hybrid Native Vision & Layout Inspector (`desktop_vision.rs`, `pixel.rs`, `vision.rs`)
Designed so a text-only LLM can visually inspect and debug its own UI without burning context tokens on raw base64 images:
* **Zero-Bloat Capture Pipeline:** Captures the WebView2 surface asynchronously via `ICoreWebView2::CapturePreview` (dispatched on the UI STA thread, rewound via `IStream::Seek(0)`, and awaited via `oneshot` with an `800 ms` timeout), with a Win32 `PrintWindow(hwnd, hdc, PW_CLIENTONLY | PW_RENDERFULLCONTENT)` fallback for external windows.
* **DPI-Aware Coordinate & Color Verification:** Scales logical CSS rects from `ui_map` by `window.devicePixelRatio` (`dpr`) before cropping physical frames, encodes PNGs natively via `flate2` (`ZlibEncoder` + `Crc`), computes localized `Pixel-Diff` percentages, and runs deterministic `3×3` / `5×5` median `Pixel-Assert` checks in linearized sRGB. Never injects `data:image/png;base64,...` into LLM context.
* **Semantic DOM/CSS Inspector (`ui_map` & `ui_zoom`):** Waits for active CSS animations (`document.getAnimations()`), inspects the live DOM/CSSOM in `~150–250` tokens, and flags `CLIPPED_*`, `OVERLAP` (`> 1.5 CSS px`, z-index and ancestor aware), `ZERO_HEIGHT`, `TRUNCATED_TEXT`, `VERTICAL_CRUSH`, `OCCLUDED_CENTER`, `CONTRAST_FAIL`, and `THEME_SURFACE_MISMATCH`.
* **Optional Multi-Key `Vision Proxy`:** Supports up to 10 user-configured VLM slots (`Gemini`, `OpenRouter`, or custom OpenAI-compatible endpoints, disabled by default) with `+64 px` contextual padding, `Per-Provider Circuit Breakers`, exponential backoff, and strict key redaction.

### 4. Zero-Trust Loopback Firewall, Win32 Isolation & Self-Update Lifecycle
* **IPC & CSRF Firewall (`api/mod.rs`):** Validates `Origin` (`http://127.0.0.1:47931`, `http://tauri.localhost`, `https://tauri.localhost`), `Host`, and `OPTIONS` preflight headers, enforcing a cryptographic per-boot session token on all `/api/*`, `/v1/*`, and mutating routes.
* **Hardened Shell & Process Isolation (`api/tools.rs`, `engine.rs`):**
  * Normalizes `cmd.exe` caret (`^`) escapes and quotes before deny-list inspection; blocks encoded PowerShell (`-e`/`-ec`/`-enc`), destructive `robocopy /MIR /purge`, `rd /s`, `format`, and `taskkill` targeting host/engine PIDs.
  * Spawns subprocesses via `STARTUPINFOEXW` + `PROC_THREAD_ATTRIBUTE_HANDLE_LIST` (preventing child processes from inheriting the `:47931` listener socket) inside `TOOLS_JOB` (`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE | JOB_OBJECT_LIMIT_ACTIVE_PROCESS = 32`).
  * Passes `cmd.exe /d /c` commands via `CommandExt::raw_arg` (`chcp 65001 >nul 2>nul & <cmd>`) and decodes `UTF-16` BOM, `UTF-8`, `CP866`, and `CP1251` console streams cleanly.
* **NTFS Traversal & Reparse Guards (`resolve_tool_path`):** Walks path components top-down via `symlink_metadata()`, rejecting symlinks, NTFS Junctions (such as `Release/models`), Alternate Data Streams (`:`), trailing dots/spaces, and 8.3 short-name bypasses (`GetLongPathNameW`).
* **Non-Blocking Redacted Audit Trail:** Streams tool calls and HTTP routes over a bounded `mpsc` channel (`1024` capacity) to `Release/data/logs/audit.jsonl` (`10 MB` auto-rotation), masking API keys (`sk-`, `AIza`, `Bearer`, `?key=`, `-----BEGIN`).
* **Crash-Safe Lifecycle & Self-Updater (`main.rs`):** Protects single-instance execution via `Local\UnStrataV2_Master_Instance_Mutex`, cleans orphaned `EBWebView/LOCK` processes prior to window creation, prevents system sleep via `SetThreadExecutionState` during active agent loops, captures full backtraces to `crash.log` via `std::panic::set_hook`, and spawns an out-of-job micro-updater (`--wait-pid`) to swap `unstrata.exe` -> `unstrata.exe.old` autonomously.

---

## Built-In Native Agent Tools

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

## Key Engineering Problems Already Solved

* **Eliminated Electron & React Bundle Bloat (`~1.8 GB → ~35 MB RAM`, `3.9 MB → ~530 KB UI`):** Reclaimed nearly 2 GB of system memory for model weights and KV caches while decoupling completely from legacy ports (`:47831`).
* **Resolved Windows Startup Access Violations (`0xC0000005`) & Nested Job Conflicts:** Removed recursive host self-containment (`shelter_self`), added dynamic WebView2 runtime discovery, deferred DWM caption styling until window readiness, and added a `SetPriorityClass` fallback against `ERROR_INVALID_PARAMETER (87)`.
* **Amputated Slow Python Sidecars:** Eliminated external HTTP sidecars (`:47937`, `:47936`) for command execution, document parsing, search, and scraping in favor of sub-millisecond native Rust handlers.
* **Fixed High-DPI (`125%` / `150%`) Vision Coordinate Drift:** Separated `client_origin` from DWM shadow bounds (`-7 px`) and scaled logical CSS coordinates by `dpr` prior to physical frame cropping.
* **Eliminated WMI / PowerShell Telemetry Stalls:** Replaced slow WMI queries with sub-millisecond Win32/driver calls (`GlobalMemoryStatusEx`, `GetSystemTimes`, `NVML` + `DXGI`).

---

## System Requirements (Recommended)

* **OS:** Windows 11 x64 (with Microsoft Edge WebView2 Runtime)
* **CPU:** x86_64 processor with `AVX2` support (8+ cores recommended for MoE expert routing and CPU audio)
* **GPU:** NVIDIA GPU with 12–16+ GB VRAM for heavy local MoE/dense inference (or any GPU / CPU setup when using External Provider mode)
* **RAM & Storage:** 32–80 GB DDR4/DDR5 and a fast PCIe NVMe SSD (with system-managed or `>= 32–48 GB` Windows pagefile for `80+ GB` `mmap` model weights)

---

## Roadmap: What Remains (~18%)

### Phase 1: Rust Theme Engine & Frosted Glass UI Polish *(In Progress)*
* Finalize the JSON theme schema (`colors`, `glass` blur/alpha, up to 8 procedural `ambient` gradients) with `GET/POST/DELETE /api/themes` and `/api/palette` persistence in Rust, paired with the in-app **Theme Studio** (`App settings -> Appearance`), 1-click accent switcher, glow-free frosted glass panels, and smoothed Fritsch-Carlson throughput telemetry charts.

### Phase 2: Native CPU `AVX2` Speech-to-Text (`whisper.dll` FFI)
* Replace the final Python audio bridge (`:47932`) with in-process `whisper.dll` (`AVX2`, strictly CPU-pinned so 0 MB of VRAM is taken from MoE weights), capturing native WASAPI audio via `AudioWorkletNode`, resampling in Rust to `16 kHz` with DC-blocker + subnormal flush + RMS VAD, and supporting `ru | uk | en | auto` switching.

### Phase 3: Analytical Coprocessor & KV-Cache Optimizations
* Implement `verify_tools.rs` (`compute_verify`, `data_probe`, `sys_probe`), one-shot `Output Spooling` (`<= 4 KiB` deduplicated previews), parallel read-only `tokio::JoinSet` batching, atomic multi-hunk `multi_edit`, targeted `PrefetchVirtualMemory` for `*.gguf` headers/tails, and two-stage `Anchored Compaction` (`75%` turn-boundary / `80%` mid-tool-loop).

### Phase 4: End-to-End Verification Gate
* Validate offline unit tests (`cargo test --offline -j 2`), cold `:8080` startup & prefill `Abort` recovery, `unstrata.exe` self-restart with `EBWebView/LOCK` verification, multi-viewport (`760px` / `900px` / `1440px`) `ui_map` + `Pixel-Assert` checks (`0` console errors), and end-to-end tool + voice audit logging.

---

## Hardware Tiers & Throughput Calculations

Because Sparse MoE inference during token generation (decode) is **memory-bandwidth-bound** rather than compute-bound, `UnStrata` maximizes throughput by matching active parameter reads (`~4B–6B` active weights per token) to the fastest available memory bus and amplifying decode speed via **Multi-Token Prediction (MTP)** speculative drafting (`--spec 4`):

* **Tier 1 — VRAM (`288–1,792 GB/s`):** Holds dense/shared layers, quantized KV-cache (`q8_0`/`q4_0`), and hot routed experts that fit inside physical VRAM (`13.5–14.5 GiB` budget on a 16 GB card). Exceeding physical VRAM spills hot experts over PCIe (`~6.5 GB/s` on PCIe 3.0 x8) and caps 80B MoE speed around `~30 tok/s`.
* **Tier 2 — System RAM (`47–96 GB/s`):** Holds remaining hot/warm MoE experts in DDR4/DDR5 (`80 GB` RAM eliminates SSD pagefile stalls on 70B–120B weights).
* **Tier 3 — NVMe `mmap` (`3.4–7.4 GB/s`):** Streams only cold tail experts (`<15%` Zipfian miss rate), while MTP draft verification validates up to 4 tokens per pass (**1.8×–2.3×** effective speedup).

### Recommended Configurations & Expected Performance

| Hardware Tier | Bus & Memory Specs | Recommended Engine Flags | Target Model Scale | Expected Decode |
| :--- | :--- | :--- | :--- | :--- |
| **Entry (12 GB VRAM)**<br>RTX 3060 12GB + 32 GB DDR4-3200 | `360 GB/s` GDDR6<br>`51.2 GB/s` DDR4<br>PCIe 4.0 x16 | `--resident-budget-gib 10.5`<br>`--vram-reserve-mib 1536`<br>`--kv q8_0 --spec 4` | MoE `30B–35B` (`IQ3_M`/`Q3_K_M`)<br>or Dense `14B` (`Q4_K_M`) | **45–65 tok/s** (30B MoE)<br>**25–36 tok/s** (14B) |
| **Developer Reference (16 GB VRAM)**<br>RTX 4060 Ti 16GB + Ryzen 7 5700X + 80 GB DDR4 + NVMe | `288 GB/s` GDDR6<br>`~48 GB/s` DDR4<br>PCIe 3.0 x8 (`~6.5 GB/s`) | `--resident-budget-gib 14.0`<br>`--vram-reserve-mib 1536`<br>`--kv q8_0 --pool-workers 6 --spec 4` | Sparse MoE `80B` (`TQ1_0`)<br>or MoE `30B` / Dense `14B` | **50–65+ tok/s** (80B MoE tuned)<br>**65–110 tok/s** (30B MoE / 14B) |
| **Performance 16 GB (GDDR6X/GDDR7)**<br>RTX 4070 Ti Super / 5070 Ti / 5080 + 64 GB DDR5-6000 | `672–960 GB/s` VRAM<br>`96.0 GB/s` DDR5<br>PCIe 4.0/5.0 x16 | `--resident-budget-gib 14.2`<br>`--vram-reserve-mib 1536`<br>`--kv q8_0 --pool-workers 8 --spec 6` | MoE `80B–120B` (`IQ2_M`/`Q3_K_S`)<br>or Dense `22B–24B` (`Q4_K_M`) | **55–85 tok/s** (MoE)<br>**50–85 tok/s** (24B) |
| **Workstation (24–32 GB VRAM)**<br>RTX 3090 / 4090 (24GB) or RTX 5090 (32GB) + 96–128 GB DDR5 | `936–1,792 GB/s` VRAM<br>`96.0 GB/s` DDR5<br>PCIe 4.0/5.0 x16 | `--resident-budget-gib 22.0..30.0`<br>`--vram-reserve-mib 2048`<br>`--kv q8_0 --pool-workers 10 --spec 6` | Large MoE `120B–235B`<br>or Dense `32B–70B` | **65–115+ tok/s** (MoE)<br>**50–85 tok/s** (32B) |
