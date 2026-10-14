# Cross-GPU Feasibility Plan

Goal: decide, with measurements, whether extending OptiScaler to use a second GPU is viable, and whether it is worth buying a second GPU for it. This file records the ideas discussed, the reasoning behind each, what is known, what is only an estimate, and the experiments that replace the estimates.

Hardware on hand: AMD Radeon RX 9070 XT (RDNA4 dGPU) and the Ryzen 9800X3D integrated GPU (iGPU). Windows native for all experiments (not WSL).

Companion document: `architecture.md` (how OptiScaler is structured and where it intercepts).

Status of claims: numbers below are **estimates from discussion** unless marked "measured". Nothing here has been verified against the code on this branch (`feat/multigpu-support`) or against a real game. Reading what the branch already does is an open item (see Section 9).

---

## 1. Ideas under consideration

The user's goals, in order of discussion:

1. Move **upscaling** and **frame generation (FG)** to a second GPU.
2. Use **same-vendor GPUs only** for the study. Reason: NVIDIA drivers do context checks and are closed, so DLSS cannot run on an AMD GPU through this project. The reverse (FSR/XeSS on NVIDIA) works. Same vendor also avoids cross-vendor shared-memory problems.
3. Explore whether **ray tracing (RT) and path tracing work can be split across two GPUs** using tensor-parallel (TP), pipeline-parallel (PP) or data-parallel (DP) style structures from AI, to make RT/path tracing viable on AMD GPUs.

Each idea is analysed below with its mechanism, costs and verdict so far.

---

## 2. Background needed to follow the analysis

### 2.1 Where upscaling, FG and RT sit in a frame

```
 game engine (all in the game's own command lists)
 ┌───────────┐  ┌────────────────────┐  ┌──────────┐  ┌───────────┐  ┌──────┐  ┌─────────┐
 │ raster /  │→ │ RT passes (DXR):   │→ │ denoise  │→ │ composite │→ │UPSCAL│→ │ FG →    │
 │ G-buffer  │  │ shadows/refl/GI/PT │  │ (NRD,    │  │ + tonemap │  │ER    │  │ HUD →   │
 │           │  │ ~1-2 spp, noisy    │  │ SVGF, RR)│  │           │  │      │  │ Present │
 └───────────┘  └────────────────────┘  └──────────┘  └───────────┘  └──────┘  └─────────┘
                        ▲ OptiScaler never sees this ▲                ▲ OptiScaler starts here
```

- RT happens before the upscaler. Rays are cast at low resolution and few samples per pixel (spp). The result is noisy, a denoiser cleans it, and the upscaler works on the cleaned image.
- DLSS Ray Reconstruction (RR) merges denoise and upscale in one network. This repo has a `dlssd` folder, so it already intercepts it.
- OptiScaler works because the upscaler and FG boundaries are small and standardized: color, depth, motion vectors, jitter and a few masks. RT has no such boundary.

### 2.2 DXR in plain terms (the three layers)

```
 GAME           its own ray programs (shaders) + calls: "build tree from these triangles",
                "launch rays with these shaders and textures"
   │
 DXR / D3D12    Microsoft's contract: the list of calls the game may make
   │
 DRIVER         vendor implementation: builds the tree, walks it with GPU ray hardware,
                compiles and runs the game's shaders
   │
 GPU
```

- DXR is an **API, not an algorithm**. It offers two fixed building blocks: build an acceleration structure (a tree of boxes, "BVH") from triangles, and `TraceRay` (find nearest hit). Everything else is developer code: which rays, how many, shading, bounces, denoising, sample reuse.
- Analogy: a vendor-supplied FlashAttention-style kernel inside a custom model. The loop is standard; the body (hit/miss shaders) is the game's.
- The tree format is **driver-specific** when the game builds it through DXR, so each GPU would build its own copy from the same geometry. If we owned the tree format (our own builder and traversal code) it would be transferable, but our traversal could not use the GPU's ray hardware, so it is likely several times slower per ray (factor not measured).
- Two ways to launch rays: a dedicated call, `DispatchRays(width, height)`, or **inline ray queries** inside ordinary compute or pixel shaders. Inline queries mean there is no single "ray launch" call to intercept; one would have to split arbitrary compute dispatches.
- Vulkan has an equivalent ray tracing extension; some games use it.

### 2.3 What a BVH ray cast is as a compute workload

- A BVH is a tree of boxes over the scene's triangles. A ray starts at the root, skips every box it misses, descends into those it hits, and tests the few triangles in the leaves it reaches.
- Compared with attention/matmul:

| | Attention / matmul | BVH traversal |
|---|---|---|
| Access pattern | Regular, tiled, predictable | Pointer chasing; each address depends on the previous node |
| Work per thread | Uniform | Varies 10x or more, so thread groups diverge |
| Limiter | FLOPs, or streaming bandwidth | Memory latency and cache hit rate |
| Reuse | High, via tiles | Top tree levels hot, leaves cold |
| Coherence | Perfect | Primary rays coherent; diffuse bounces scattered |

- Closest AI analogies: embedding-table lookups, graph traversal, paged attention with random block tables.
- Rough volume: 1440p, 1-2 spp, 2-4 bounces is about 10-25M rays per frame, so on the order of a billion rays per second at 60 fps.
- On NVIDIA the RT core does traversal and box/triangle tests in fixed function. On AMD RDNA2/3 the hardware does intersection tests while the traversal loop is shader code; RDNA4 reportedly improved this. Not verified here.

### 2.4 Sizes used in estimates (1440p internal, 4K output)

| Item | Size |
|---|---|
| Color RGBA16F, 1440p | about 30 MB |
| Depth, 1440p | about 15 MB |
| Motion vectors, 1440p | about 15 MB |
| Inputs to the upscaler | about 60 MB total |
| 4K RGBA16F (upscaled HDR) | about 66 MB |
| 4K RGBA8 (final) | about 33 MB |
| G-buffer (surface info) 1440p, full / half | about 100 MB / about 50 MB (assumes 24-32 B per pixel) |
| Noisy radiance 1440p, full / half | about 30 MB / about 15 MB |

Bandwidth reference points: PCIe 4.0 x16 about 25 GB/s practical, x8 about half of that, x4 about 7-8 GB/s. A 60 MB copy at 16 GB/s is about 4 ms.

---

## 3. Idea A: frame generation on the second GPU

**Mechanism.** Render GPU produces frames and sends them to GPU B, which generates the extra frames and presents. Best arrangement: monitor connected to GPU B so no result returns to GPU A.

**Why it fits.**
- FG already adds about a frame of latency, so a few extra ms of copy hurts less.
- It removes a large chunk of GPU time and memory from the render GPU.
- There is no mid-frame dependency: the game's frame is finished before FG needs it.

**Data.** Final frame about 33 MB (4K 8-bit) one way. Better quality (OptiFG-style) also needs depth and motion vectors, about 60-95 MB at 1440p internal.

**Prior art (web search; community and secondary sources only):**
- Lossless Scaling dual-GPU mode: render GPU plus a second GPU for FG, cross-vendor works. It uses only the final image, no depth or motion vectors. Reported community numbers: RTX 3090 + RTX 3050 build on PCIe 4.0 x8 for both, a 4090 (x8) + 4060 LP (x4) capped at 52 fps in one game, an x1 secondary limited to about 1 GB/s, roughly 3-5 ms of added latency per one guide, and weak iGPUs reported as poor FG GPUs. These are anecdotes, not benchmarks.
- No documentation found of OptiScaler itself offloading to a second GPU.

**Verdict so far.** Best candidate: proven demand, lowest effort, no return trip. Open question: quality and latency when depth/motion vectors are sent as well.

---

## 4. Idea B: upscaler (and denoiser) on the second GPU

**Mechanism.** Send the low-res inputs to GPU B, run the upscaler there, send the result back.

**The return trip, explained.** Games call the upscaler in the middle of their pipeline and then keep processing on the upscaled image (tonemap, film grain, UI at native resolution). So the upscaled result must go back to GPU A. This applies to upscaler offload and not to FG offload.

**Cost.** About 60 MB out and about 66 MB back (4K HDR). At about 16 GB/s that is roughly 8 ms round trip, about 20 ms on an x4 slot. The upscale itself is about 1-3 ms. It adds to the FG latency instead of replacing it.

**The user's counter-argument (accepted in part).**
- Offloading saves **memory and compute** on the render GPU. DLSS transformer-model memory was cited at roughly 300 MB-1 GB (not verified); 4K history buffers and FG buffers add more. On an 8-12 GB card that matters.
- FG latency is already much greater than upscaler latency, so the extra copy may be tolerable.

**Where we agree and where not.** The memory and compute headroom is a real benefit, separate from latency. The latency is additive, not "the same either way", but may be acceptable.

**Vendor constraint.** Same vendor only (see Section 1). On AMD, FSR4 needs RDNA3/RDNA4-class hardware per the project README, which rules out the iGPU for FSR4. On NVIDIA a second NVIDIA GPU could run DLSS or Ray Reconstruction (the cross-vendor "AMD renders, NVIDIA denoises" idea is **out of scope** because of NVIDIA driver restrictions).

**Verdict so far.** Viable if memory/compute headroom is the goal; poor for latency. Reuse of cross-GPU copy code from Idea A.

---

## 5. Idea C: splitting the ray tracing work across two GPUs

This is the hardest idea and where most of the discussion went.

### 5.1 The AI-parallelism mapping

| Strategy | RT translation | Verdict |
|---|---|---|
| **DP over rays** (pixel split / split-frame rendering, SFR) | Each GPU holds the full scene and traces half the pixels (tiles or checkerboard) | Closest fit. Rays are independent. No all-reduce is needed, only a gather of results. |
| **DP over frames** (alternate frame rendering, AFR) | Next frame on GPU B while A is still tracing | Dead end without the engine: temporal history (TAA, denoisers, upscalers) links frames; stutter and latency. Died with SLI/CrossFire. |
| **TP analog** (shard the BVH/scene, combine hit distances like partial attention results) | Each GPU holds half the geometry; every ray goes to both; take the nearest hit | Saves memory, not time. Needs a PCIe exchange per bounce, too slow for a 16 ms frame. Suited to offline renderers and scenes larger than VRAM. |
| **PP analog** (stages on different GPUs) | GPU A rasterizes G-buffer, GPU B does RT and denoise, A does upscale/post | Transfers of tens of MB per frame fit the PCIe budget, but need engine integration and add latency. |

Why AI-style TP does not transfer: AI TP relies on NVLink-class bandwidth (hundreds of GB/s) and large uniform tensors. Consumer PCIe is 10-30x slower, and the RT work is irregular.

### 5.2 Where the split can cut (the key design question)

- **Cut 1: rays out, hit results back.** B only walks the tree; A keeps shading. About 10-25M rays per frame at roughly 32 B each is 300-800 MB outbound plus 150-400 MB back, and each bounce depends on the previous bounce's shading, so one round trip per bounce. **Fails on bandwidth and latency.**
- **Cut 2: B traces and shades its own pixels, returns final noisy color.** This is the pixel split. About 50 MB in (surface data for B's half plus per-frame dynamic updates) and 15-60 MB out. **This is the right shape**, but B must run the game's own shading code on its own copy of the data.

### 5.3 What cut 2 requires from OptiScaler

1. Capture the game's ray shaders (hook `CreateStateObject`); same-vendor GPUs can recompile them.
2. Mirror the data those shaders read: geometry, textures, materials, per-frame surface info. Games use one big descriptor table ("bindless") that points at most of the scene, so in practice most texture memory is mirrored and VRAM use roughly doubles.
3. Turn one `DispatchRays` over the full grid into two launches, one per GPU, each with half the pixels. For inline ray queries, split the compute dispatch instead.
4. Exchange results and synchronize.
5. Handle dynamic data (animated meshes, transforms) each frame, and rebuild the tree on GPU B each frame (a few ms, not split).

### 5.4 Wrap vs replace (clarifying the design)

- **Wrap the API (Option A):** sit between the game and D3D12/DXR, forward calls to **two real drivers**. This is like PyTorch DDP: wrap the model, replicate it, split the batch, merge results. No kernels are rewritten. This is the design discussed.
- **Replace the driver's part (Option B):** write our own two-GPU traversal. It would lose the ray hardware and would still need to run the game's hit programs inside it.
- The game's API calls always matter: they are the only way to learn the triangles, shaders and resources.
- OptiScaler already hooks and wraps many calls (D3D12 resource creation, `wrapped/` COM wrappers, resource tracking), but not at the scale needed to mirror a whole device. Most of D3D12 would have to be virtualized because ray shaders read the same textures, buffers and descriptors as the rest of the frame.

### 5.5 Break-even (compute-starved case)

If one GPU is compute-starved by ray count, splitting is where the largest gain is. Example, one GPU: 40 ms tracing plus 10 ms everything else (about 20 fps):

| Step | Time |
|---|---|
| Tracing, half the pixels each | 20 ms |
| Data in and out | about 6 ms |
| Wait for the slower half | about 2 ms |
| B rebuilds the scene tree | about 2 ms |
| Tracing portion total | about 30 ms (was 40) |
| Frame total | about 40 ms (was 50), about 25 fps |

- With 60 ms tracing the portion drops to about 41 ms (about 1.45x).
- Realistic range: 1.3-1.6x on the tracing part; the ceiling is below 2x.
- Compounds with FG: a higher base frame rate is doubled by FG.
- Improvements: weight the split by GPU speed (a weaker B gets fewer pixels); trade transfer for duplicate work by letting an idle B redo cheap steps (raster of the surface buffer) instead of receiving about 50 MB each frame; balance the load dynamically from last frame's timings.

### 5.6 What the `DispatchRays` launch needs and returns

| Input | Size (est.) | When it moves |
|---|---|---|
| Ray shaders (compiled code) | KB to a few MB | Once at creation |
| Shader tables | KB to MB | When objects change |
| The tree | hundreds of MB to GBs | B rebuilds its own, or transfer |
| Geometry (for normals/UVs) | hundreds of MB to GBs | Static once; animated meshes tens of MB per frame |
| Textures and materials | GBs | Once, plus streaming updates |
| Per-frame constants | KB | Every frame |
| Per-pixel surface info from earlier in the frame | about 100 MB full, about 50 MB half | Every frame (the big one) |
| Previous-frame history read by sample-reuse | could be 100s of MB | Every frame, if read |

Outputs written to textures: noisy color about 15 MB for B's half; extra denoiser outputs another 15-45 MB; sample-reuse buffers could be hundreds of MB if the game writes them.

Per-frame total in the simple case: about 65-110 MB, so 4-7 ms at 16 GB/s (about 9-15 ms on x4). Mostly not hidden behind other work. One-time cost: static scene data, e.g. 10 GB takes about a second at 16 GB/s, and VRAM use roughly doubles; level streaming repeats it.

### 5.7 Other risks specific to the ray split

- Sample-reuse techniques with large per-pixel buffers (reservoirs, history) could make the transfer several times larger and erase the gain.
- Uneven halves: some regions need more bounces, so one GPU waits.
- Two GPUs must agree: GPU readbacks to the CPU (auto-exposure, culling) and small floating-point differences could make duplicated copies drift.
- Duplicated work (raster, post) gets no speedup.

### 5.8 Prior art on the ray split (web search; shallow)

- Microsoft's DirectX 12 multi-adapter sample splits ray tracing of a scene between an integrated and a discrete GPU and reportedly saved about 26 ms; authors warn it may not translate to real games (simple pixel-shader ray tracer).
- Unreal's Elemental demo gave the integrated GPU only post-processing; whole-frame splitting was judged impractical because of transfer latency and the speed gap.
- Developer guidance: almost no game used explicit multi-adapter; temporal techniques make AFR hard; inter-GPU bandwidth is limited.
- No shipping real-time path tracer that splits one frame across two GPUs was found. No paper comparing replicated versus partitioned trees in a real-time multi-GPU setting was found.

**Verdict so far.** Technically sound shape (pixel split, cut 2), but it is a per-game fragile research prototype and much larger than FG or upscaler offload. Not viable as a small change to OptiScaler.

---

## 6. Ranking of ideas (current belief)

| Idea | Difficulty | Value | Status |
|---|---|---|---|
| FG on a second GPU | Medium-high | High | Worth prototyping first |
| Upscaler (and DLSS-RR) on a second GPU, same vendor | Medium | Medium (memory and compute headroom; latency additive) | After FG; shares copy/sync code |
| Pixel-split ray tracing across two GPUs | Very high | Potentially high in path tracing | Test in a standalone app before touching OptiScaler |
| Scene split (TP-like), AFR | Very high or not viable | Low | Not pursued |

---

## 7. Hardware limits of the machine on hand

- 9800X3D iGPU: as far as known, a 2-compute-unit RDNA2 part, roughly 1-2% of the 9070 XT's compute. It uses system RAM, so copies to it do not cross PCIe. Not sure whether it exposes DXR; the probe will report it.
- FSR4 needs RDNA3/4-class hardware, so the iGPU cannot run it.
- A fair split would give the iGPU about 2% of the pixels, so **no speedup can be measured with this pair**. It can only validate plumbing and the system-RAM path.
- A real speed test needs a second dGPU of comparable speed in an x8 or x16 slot. A chipset x4 slot is the common weak point.

---

## 8. Experiments

### The two questions this phase answers

| # | Question | Why it gates everything |
|---|---|---|
| Q1 | Can two D3D12 devices, one per GPU, share memory and synchronize with fences in one process? | If not, FG, upscaler or ray split offload is blocked at the API level. |
| Q2 | What do cross-GPU copies of 15 / 60 / 100 MB cost (throughput and latency)? | Every offload design is a bet that copy cost is smaller than the saved GPU time. |

Out of scope for this phase: DXR on the iGPU, actual ray splitting, upscaler/FG running on the iGPU.

### Deliverable

A standalone Windows C++ program (D3D12 + DXGI, no game, no OptiScaler code) plus a results file. Proposed location: `experiments/crossgpu-probe/` at the repo root with its own project file, so `OptiScaler.sln` is unaffected. Code can be written on this machine and built and run on the Windows PC with Visual Studio 2022.

### Phase 0: Environment checks

1. Windows native, Visual Studio 2022 with the Windows SDK. (D3D12 in WSL is limited; this repo cannot be built under WSL either.)
2. Confirm the iGPU is visible: Device Manager > Display adapters shows two entries. If not, enable the iGPU in BIOS; some AM5 boards only enable it when a display is connected to the motherboard output.
3. Record the PCIe link of the 9070 XT slot (generation and width, via GPU-Z or HWiNFO), plus the motherboard model and its second slot's link width and lane source (CPU or chipset).
4. Record the driver version.

Exit: two adapters visible, notes saved.

### Phase 1 (Q1): Probe and two-device handshake

1. Enumerate adapters with `IDXGIFactory6::EnumAdapterByGpuPreference`; skip the software adapter. Print name, vendor and device ID, LUID, memory.
2. Create one `ID3D12Device` per adapter, with a direct and a copy queue each.
3. Query capabilities per device: `D3D12_FEATURE_D3D12_OPTIONS` (`CrossAdapterRowMajorTextureSupported`), `D3D12_FEATURE_D3D12_OPTIONS5` (`RaytracingTier`, informational), node and heap tier.
4. Shared fence: `D3D12_FENCE_FLAG_SHARED | D3D12_FENCE_FLAG_SHARED_CROSS_ADAPTER` on device A, `CreateSharedHandle`, `OpenSharedHandle` on B. Signal from A and wait on B, then the reverse.
5. Shared memory: a heap on A with `D3D12_HEAP_FLAG_SHARED | D3D12_HEAP_FLAG_SHARED_CROSS_ADAPTER`, export, open on B; a buffer placed in that heap on each device (`D3D12_RESOURCE_FLAG_ALLOW_CROSS_ADAPTER`; buffers first, row-major textures later). Reference: Microsoft's `D3D12HeterogeneousMultiadapter` sample in the `DirectX-Graphics-Samples` repository.
6. Correctness: fill a native buffer on A with a known pattern, copy into the cross-adapter buffer, fence, copy out into a native buffer on B, read back and verify; repeat in the other direction, with random patterns and several sizes.

Exit (Q1 answered): all steps pass both directions for at least 1,000 iterations, zero mismatches. If a step fails, record exactly where (heap open, fence, copy, data) and stop; that is the answer.

### Phase 2 (Q2): Copy-cost measurements

| Variable | Values |
|---|---|
| Size | 1, 15, 30, 60, 100, 200 MB |
| Direction | dGPU to iGPU, iGPU to dGPU |
| Mode | one copy at a time (latency); several in flight (throughput) |
| Load | idle; source GPU busy with a synthetic load; CPU doing a memory-heavy loop (iGPU and CPU share DDR5) |

Per cell: 20 warm-up and 200 measured iterations; record median, p95 and derived GB/s.

Timing:
- Per-leg GPU time: GPU timestamp queries (`ID3D12QueryHeap`, `GetTimestampFrequency`) for the write into the shared heap (A side) and the read out (B side).
- End-to-end latency: wall clock (QPC) from "copy submitted on A" to "copy complete on B" through the shared fence.
- Cross-device timeline: align the two GPU clocks with `GetClockCalibration`.

Key trick for the buying decision: the shared heap very likely lives in system memory, so the A-side leg runs over the 9070 XT's PCIe link and the B-side leg over whatever link GPU B has. Measure the **A-side leg alone**; that gives the real best-case PCIe leg on this machine. Verify the system-memory assumption from the numbers (limited by PCIe speed, not VRAM speed). The iGPU adds the other leg and the software overhead (fence latency, queue scheduling) but its leg runs over system RAM, so it is not a stand-in for a second dGPU's PCIe link.

Frame-like test: a producer on GPU A pushes a payload at a fixed rate (60 and 120 per second) and GPU B consumes it. Count dropped or late frames and report added latency. Payloads from the estimates above:

| Scenario | Payload |
|---|---|
| FG offload, final frame, 4K 8-bit, one way | about 33 MB |
| FG offload with depth and motion vectors, 1440p internal | about 60-95 MB |
| Upscaler offload round trip, 4K HDR out | about 60 MB in plus about 66 MB back |
| Ray split per frame (inputs plus results) | about 65-110 MB |

Exit (Q2 answered): a results table for the matrix, plus the frame-like test outcome at 60 and 120 per second.

### Decision rules (fixed before measuring)

Required throughput = payload times frame rate. Example: 100 MB at 60 per second is 6 GB/s; at 120 per second, 12 GB/s.

- **Go for buying a second GPU** if the measured sustained A-side leg is at least about 2x the required throughput for the scenario of interest, and end-to-end latency for one frame's payload is under roughly 4-5 ms (the figure community reports give for dual-GPU FG setups; not a benchmark).
- **Check the slot, not just the card.** GPU B's throughput is bounded by its slot's link width. Compare the second slot's link with the required throughput.
- **No-go or rethink** if copies run far below the expected link speed, if sync latency dominates small payloads, or if Q1 failed.

---

## 9. Follow-on work (after the gates pass)

1. FG offload prototype: copy a frame to GPU B, run simple interpolation there, present from GPU B. Then try sending depth and motion vectors.
2. **PIX capture of a path-traced game** (PIX can record DXR frames) to get the real list and sizes of resources each ray launch reads and writes. This replaces the transfer estimates in Section 5.6 and shows whether sample-reuse buffers blow up the transfer.
3. Choose and buy the second GPU using the measured numbers and the slot's link width.
4. A standalone DXR test app: trace a simple scene split by pixels across two GPUs, with a weighted split, and measure real scaling, exchange cost and imbalance. This decides whether the virtualization layer for OptiScaler is worth attempting.
5. Read the existing multi-GPU work on this branch. Not yet done. Find where it creates devices and selects adapters (likely `DxgiFactory_Hooks`, `D3D12_Hooks`, `hkvkCreateDevice`, and the hidden-device interop in `IFeature_Dx11wDx12` and `IFeature_VkwDx12`) and how close it already is to Ideas A and B.
6. Check OptiScaler's GitHub wiki and issues for any second-GPU work.

## 10. Open items and risks

- The iGPU might not enumerate (BIOS) or may not support everything the dGPU does; Phase 1 exposes this early.
- The iGPU's compute is tiny, so it only validates plumbing and the system-RAM path.
- The "shared heap lives in system memory" assumption is checked in Phase 2; if the driver behaves differently, the reading of the numbers changes.
- Every size and break-even figure in this document is an estimate from discussion until PIX captures and the Phase 2 measurements replace them.
- Prior-art claims come from a shallow web search of community and secondary sources and have not been checked against primary documents.
