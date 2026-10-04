# KytyPS5 AMD fork — Silent Hill 2 & GPU focus

Personal research fork baseline: upstream [KytyPS5/KytyPS5](https://github.com/KytyPS5/KytyPS5) at `main`.

Target host (Riandre): **Ryzen 5 7500F**, **RX 9060 XT 16GB**, **32GB RAM**, Windows.

## Goal sanity check

| Milestone | Realism on current upstream |
|-----------|------------------------------|
| Silent Hill 2 boots past splash / intro | **Active upstream work** — issue [#77](https://github.com/KytyPS5/KytyPS5/issues/77) is **BOOTS** (splash then crash) on build `7ff9e49`; not “doesn’t boot” on latest tested builds |
| SH2 reaches main menu | **Hard** — crashes with `Guest abort()` in `libC.cpp` (game assert), often before GPU is the bottleneck |
| SH2 in-game | **Not yet** — UE5 + heavy AV/PlayGo paths |
| SH2 @ 60 FPS | **Long-term** — requires correct HLE *and* major GPU/submit/recompile optimizations; comparable to years of RPCS3-style work, not a driver tweak |

Minecraft 10–20 FPS and Stray 1–5 FPS on your GPU are **not** mainly “AMD Vulkan is broken.” They match an early emulator that **recompiles RDNA2 guest shaders and builds Vulkan pipelines on the render thread** while much of the CPU side is still incomplete.

## Silent Hill 2 — what actually blocks you

- **Title ID:** `PPSA08710` (US); EU builds use `PPSA08709`.
- **Reported behavior (collaborator retest):** logo/splash screens, then crash — promoted to **BOOTS**, not playable.
- **Typical failure:** `Guest abort()` — the **game** called `abort()`; the log includes guest stack words and `RuntimeLinker` symbols. Fix path is **compatibility/HLE**, not RX tuning first.
- **Reference fork:** [KytyPlus](https://github.com/Coder787-source/KytyPlus) reportedly reached the **content warning** screen (~4 FPS on iGPU). Worth **diffing** for PlayGo, firmware, and dispatch changes — still not gameplay.

### First debug loop (no code required)

1. Use the **latest official Windows build** or a **Release** self-build (`KYTY_BUILD_RELEASE`, clean git hash — see pipeline cache below).
2. Launch with **`--amd-cpu`** (see below) and try **`--playgo-hack`** if chunks fail to resolve.
3. Capture full **`_kyty.txt`** from boot through crash.
4. In the log, find **`Guest abort diagnostics`** and **`Guest abort stack words`** — note the **`module+offset`** lines.
5. File or track against upstream **Game Emulation Status Report** with build hash and log.

## Why FPS is low (GPU stack)

### 1. Synchronous shader recompilation (main thread)

`PipelineCache::ProgramCache::Get()` in `src/graphics/host_gpu/renderer/pipeline/pipelineCache.cpp`:

- On cache miss: **translate guest RDNA2 → SPIR-V → `vkCreateShaderModule`** during draw preparation.
- `PipelineCache` constructor: **`EXIT_NOT_IMPLEMENTED(!Common::Thread::IsMainThread())`** — all of this is tied to the **main/render thread**.

UE5 titles (Stray, SH2) churn huge shader permutations → constant stutter and single-digit FPS even on strong GPUs.

### 2. Vulkan pipeline cache (partial mitigation)

- On-disk cache: `_PipelineCache/<TITLE_ID>.bin` — **Release builds only**, valid git revision, not `-dirty`.
- Caches **driver pipeline blobs**, not SPIR-V / guest shader IR. Helps **second** launch, not first-run compile cost.

### 3. `shader_optimization_type` is UI-only today

Launcher exposes **None / Size / Performance**, but **`Config::GetShaderOptimizationType()` is not consumed** under `src/graphics/shader/`. Wiring this to SPIR-V optimization (e.g. spirv-opt) is a concrete fork task.

### 4. PS5 GPU ≈ RDNA2, host RX ≠ free speed

Guest shaders are **recompiled to SPIR-V for Vulkan**, not executed as native RDNA2 on the host. AMD alignment helps **ISA fidelity** in `src/graphics/shader/recompiler`, not automatic performance.

## AMD / Ryzen host tuning (do this before forking)

| Setting | Why |
|---------|-----|
| **`--amd-cpu`** | Enables guest instruction patching (SSE4a `EXTRQ`, `VRSQRTPS`, `INSERTQ`/`RDPID`/`CLWB` fallbacks) — **important on Ryzen 7500F** |
| **Release build** | Enables Vulkan pipeline disk cache |
| **Disable** Vulkan validation, shader validation, RenderDoc unless debugging | Large CPU/GPU overhead |
| **Launcher → shader optimization Performance** | Harmless today; ready when wired up |
| **Select discrete GPU** in graphics settings | Avoid iGPU if hybrid |
| **Update Adrenalin** | Vulkan 1.3 baseline required |
| **Second session** | Warm `_PipelineCache` after a long first run |

Optional: **`--profile`** for `profiler_enabled` to see whether time is in `ShaderRecompiler`, `createGraphicsPipelines`, or guest CPU.

## Code map for later optimization work

| Area | Path | Fork ideas (after SH2 boots) |
|------|------|------------------------------|
| RDNA2 decode → SPIR-V | `src/graphics/shader/recompiler/` | Fix UE5-specific ops; reduce permutations |
| Pipeline + shader cache | `src/graphics/host_gpu/renderer/pipeline/pipelineCache.cpp` | **Disk SPIR-V cache**; **background compile worker** |
| Draw / submit | `src/graphics/host_gpu/renderer/renderDraw.cpp` | Reduce sync, batch barriers |
| Guest CP | `src/graphics/guest_gpu/` | Command buffer parsing hot paths |
| Present / vblank | `src/graphics/presentation/videoOut.cpp` | Frame pacing vs uncapped compile stalls |

**Do not** start with “AMD-only Vulkan extensions” until profiling shows driver-bound work. Current evidence points to **compile-on-draw** and **missing HLE**.

## Suggested fork workflow

```powershell
cd $env:USERPROFILE\Projects\KytyPS5-AMD
git checkout -b amd-sh2-research
# After creating github.com/<you>/KytyPS5-AMD fork:
git remote add fork https://github.com/<you>/KytyPS5-AMD.git
```

CMake fork branding (optional):

```powershell
cmake -S . -B _Build/windows -G Ninja -DCMAKE_BUILD_TYPE=Release `
  -DKYTY_BUILD_ORIGIN=Fork -DKYTY_BUILD_REPOSITORY=<you>/KytyPS5-AMD ...
```

## Phase plan

1. **Boot SH2 past splash** — triage `Guest abort` stack; compare KytyPlus; upstream cherry-picks where GPL-compatible.
2. **Menu / intro video** — `libVideoDec2`, `AvPlayer`, PlayGo, NP web API (see recent upstream commits).
3. **In-game correctness** — rendering bugs, crashes.
4. **Performance** — async shader pipeline, persistent SPIR-V cache, spirv-opt, reduce validation in Release.
5. **Broader AMD polish** — auto `--amd-cpu` on AuthenticAMD, RDNA wave-size hints where safe.

## Astro Bot @ 60 FPS (later)

Treat as separate milestone: upstream already demos Astro Bot; profile that title once SH2 boot path is understood — many fixes will be shared, but Astro is a better **GPU perf** reference than SH2.
