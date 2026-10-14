# Cross-GPU Feasibility Plan (before buying a second GPU)

Goal: answer two questions on hardware we already own, so the decision to buy a second GPU rests on measurements.

Hardware: AMD Radeon RX 9070 XT (RDNA4 dGPU) + the Ryzen 9800X3D integrated GPU (iGPU). OS: Windows (native, not WSL).

Background and design reasoning: see `architecture.md` in this folder. This plan uses numbers from discussion that are estimates, not measurements. The point of this plan is to replace them.

## The two questions

| # | Question | Why it gates everything |
|---|---|---|
| Q1 | Can two D3D12 devices, one per GPU, share memory and synchronize with fences in one process? | If not, FG, upscaler or ray split offload is blocked at the API level. |
| Q2 | What do cross-GPU copies of 15 / 60 / 100 MB cost (throughput and latency)? | Every offload design (FG on GPU B, upscaler on GPU B, ray split) is a bet that the copy cost is smaller than the saved GPU time. |

Out of scope for this plan: DXR on the iGPU, actual ray splitting, FSR4 or any upscaler/FG running on the iGPU (the iGPU is far too small for gains, and FSR4 needs RDNA3/4). These come after a second real GPU is chosen.

## Deliverable

A small standalone Windows C++ program (D3D12 + DXGI, no game, no OptiScaler code) plus a results file. Proposed location: `experiments/crossgpu-probe/` at the repo root, with its own project file so it never affects `OptiScaler.sln`. Written here, built and run on Windows with Visual Studio 2022.

## Phase 0: Environment checks (before writing code)

1. Windows native, Visual Studio 2022 with the Windows SDK. D3D12 inside WSL is limited, so it is not used.
2. Confirm the iGPU is visible to Windows: Device Manager > Display adapters should show two entries. If only one appears, enable the iGPU in BIOS (AM5 boards usually have an iGPU or "Integrated Graphics" option). Some boards only enable it when a display is connected to the motherboard output; note this if it happens.
3. Record the PCIe link of the 9070 XT slot (GPU-Z or HWiNFO): generation and width. Also record the motherboard model and its second PCIe slot's link width and lane source (CPU or chipset). This feeds the buying decision.
4. Record driver version (one AMD driver covers both adapters).

Exit criterion: two adapters visible, notes saved.

## Phase 1 (answers Q1): Probe and two-device handshake

1. **Enumerate adapters** with `IDXGIFactory6::EnumAdapterByGpuPreference`. Skip the software adapter ("Microsoft Basic Render Driver"). Print name, vendor and device ID, LUID, dedicated and shared memory.
2. **Create one `ID3D12Device` per adapter** and one direct (and one copy) queue each.
3. **Query capabilities** per device: `D3D12_FEATURE_D3D12_OPTIONS` (`CrossAdapterRowMajorTextureSupported`), `D3D12_FEATURE_D3D12_OPTIONS5` (`RaytracingTier`; informational), and node and heap tier values. Print them.
4. **Shared fence.** Create a fence with `D3D12_FENCE_FLAG_SHARED | D3D12_FENCE_FLAG_SHARED_CROSS_ADAPTER` on device A, export it with `CreateSharedHandle`, open it on device B with `OpenSharedHandle`. Prove it works by signaling from A's queue and waiting on B's queue, then the reverse.
5. **Shared cross-adapter memory.** Create a heap on A with `D3D12_HEAP_FLAG_SHARED | D3D12_HEAP_FLAG_SHARED_CROSS_ADAPTER`, export, open on B. Create a buffer placed in that heap on each device (`D3D12_RESOURCE_FLAG_ALLOW_CROSS_ADAPTER`, row-major layout for textures; buffers are the simplest). Reference: Microsoft's `D3D12HeterogeneousMultiadapter` sample in the `DirectX-Graphics-Samples` repository shows this pattern.
6. **Correctness test.** Fill a native buffer on A with a known pattern, copy to the cross-adapter buffer, fence, copy from the cross-adapter buffer to a native buffer on B, read back and verify. Repeat in the other direction. Use random patterns and several sizes so ordering bugs show up.

Exit criterion (Q1 answered): all steps pass in both directions for at least 1,000 iterations with zero mismatches. If any fail, record exactly where (heap open, fence, copy, data) and stop; that is itself the answer.

## Phase 2 (answers Q2): Copy cost measurements

Test matrix:

| Variable | Values |
|---|---|
| Size | 1, 15, 30, 60, 100, 200 MB |
| Direction | dGPU to iGPU, iGPU to dGPU |
| Mode | one copy at a time (latency); several copies in flight (throughput) |
| Load | idle; with the source GPU busy running a synthetic compute or render load; with the CPU doing a memory-heavy loop (iGPU and CPU share DDR5) |

For each cell, 20 warm-up and 200 measured iterations. Record median, p95, and derived GB/s.

What to time:
- **Per-leg GPU time:** GPU timestamp queries on each queue (`ID3D12QueryHeap`, `GetTimestampFrequency`), for the write into the shared heap (A side) and the read out of it (B side).
- **End-to-end latency:** wall clock (QPC) from "copy submitted on A" to "copy complete on B", through the shared fence.
- **Cross-device timeline:** use `GetClockCalibration` to align the two GPU clocks when comparing timestamps.

Key trick for the buying decision: the shared heap very likely lives in system memory, so the A-side leg runs over the 9070 XT's PCIe link and the B-side leg over whatever link GPU B has. Measure the **A-side leg alone** (the 9070 XT writing to and reading from the shared heap). That gives the real best-case PCIe leg on this machine. Verify the "lives in system memory" assumption from the numbers (a copy limited to PCIe speed, not VRAM speed). The iGPU test adds the other leg and the software overhead (fence latency, queue scheduling), but its leg runs over system RAM, so it does not stand in for a second dGPU's PCIe link.

Frame-like test: a producer on GPU A pushes a payload at a fixed rate (60 and 120 per second) with the size set to the estimated per-frame transfers below, and GPU B consumes it. Count dropped or late frames and report added latency.

| Scenario (from the architecture discussion, estimates) | Payload |
|---|---|
| FG offload, final frame, 4K 8-bit, one way | about 33 MB |
| FG offload with depth and motion vectors, 1440p internal | about 60-95 MB |
| Upscaler offload round trip, 4K HDR out | about 60 MB in plus about 66 MB back |
| Ray split per frame (inputs plus results) | about 65-110 MB |

Exit criterion (Q2 answered): a results table for the matrix, plus the frame-like test outcome at 60 and 120 per second.

## Decision rules (define before measuring)

Required throughput = payload times frame rate. For example 100 MB at 60 per second is 6 GB/s; at 120 per second, 12 GB/s.

- **Go for buying a second GPU** if the measured sustained A-side leg is at least about 2x the required throughput for the scenario of interest, and the end-to-end latency for one frame's payload is under roughly 4-5 ms (the figure community reports give for dual-GPU frame generation setups).
- **Check the slot, not just the card.** Projected throughput for GPU B is bounded by its slot's link width. Compare the second slot's measured or spec link (Phase 0) with the required throughput. A chipset x4 slot is the common weak point.
- **No-go or rethink** if copies run far below the link's expected speed, if sync latency dominates small payloads, or if Q1 failed.

## Risks and open items

- The iGPU might not enumerate (BIOS) or might not support everything the dGPU does. Phase 1 exposes this early.
- The iGPU's compute is tiny, so "iGPU as the second GPU" only validates plumbing and the system-RAM path. It cannot show speedups.
- The heap-in-system-memory assumption needs the check described in Phase 2. If cross-adapter heaps behave differently on this driver, the measurement reading changes.
- Mixed architecture (RDNA4 and RDNA2) with one driver is fine for D3D12 in principle, but shader or format differences do not matter for raw copies.
- Numbers on bandwidth and payload sizes in this plan are estimates from discussion, not verified against a real game. A PIX capture of a real path-traced game would give actual resource sizes; do it later.

## After this plan

If both gates pass, the next steps, in order of value and cost:
1. FG offload prototype: copy a frame to GPU B, run a simple interpolation there, present from GPU B.
2. A PIX capture of a path-traced game to measure real per-frame resource sizes.
3. Choose and buy the second GPU based on the slot's link width and the measured numbers.
4. Revisit the ray split in a standalone DXR test app before touching OptiScaler.
5. Read the existing multi-GPU work on this branch to see what already overlaps with the probe (not yet reviewed).
