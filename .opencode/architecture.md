# OptiScaler Architecture

OptiScaler is a Windows x64 DLL injected into a game process by DLL proxying. It sits **inside the game process**,
between the game and the graphics/upscaler libraries, and re-routes upscaling and frame-generation calls to a
user-selected backend.

> Derived from the directory layout, `README.md`, `dllmain.cpp` and the `DetourAttach` call sites in `hooks/`.
> Exact call flows inside individual backends were not traced.

## 1. Block diagram

```
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │                                  GAME PROCESS                                │
 │                                                                              │
 │   Game engine                                                                │
 │     │ renders, then asks for "DLSS / FSR / XeSS / Streamline / FFX"          │
 │     ▼                                                                        │
 │ ┌────────────────────────────────────────────────────────────────────────┐   │
 │ │ LEVEL 0  Entry: DLL proxy  (dllmain.cpp, proxies/, exports/, Source.def)│   │
 │ │   OptiScaler.dll renamed to dxgi/winmm/version/winhttp/wininet/dbghelp/ │   │
 │ │   d3d12/.asi; loads the real DLL and forwards its exports               │   │
 │ └───────────────────────────────┬────────────────────────────────────────┘   │
 │                                 │ installs Detours hooks                     │
 │ ┌───────────────────────────────▼────────────────────────────────────────┐   │
 │ │ LEVEL 1  OS / loader hooks   (hooks/Ntdll, Kernel, LibraryLoad, Gdi32,  │   │
 │ │          Wintrust, Crypt32, Advapi32)                                   │   │
 │ │   LoadLibrary redirection: game's nvngx / FFX / XeSS / Streamline DLL   │   │
 │ │   loads are redirected; signature checks are bypassed                   │   │
 │ └───────────────────────────────┬────────────────────────────────────────┘   │
 │ ┌───────────────────────────────▼────────────────────────────────────────┐   │
 │ │ LEVEL 2  Graphics API hooks                                            │   │
 │ │  DXGI    : factory EnumAdapters*/EnumAdapterByLuid/GpuPreference,       │   │
 │ │            CreateSwapChain*/Present/ResizeBuffers   (Dxgi*, FG_Hooks)   │   │
 │ │  D3D11/12: CreateDevice, CreateCommittedResource/PlacedResource,        │   │
 │ │            CheckFeatureSupport, CreateRootSignature, ... (D3D*_Hooks)   │   │
 │ │  Vulkan  : vkCreateInstance/Device, Get*ProcAddr, QueuePresentKHR,      │   │
 │ │            Create/DestroySwapchainKHR                  (Vulkan_Hooks)   │   │
 │ │  Spoofing: GPU/vendor identity returned to the game    (spoofing/)      │   │
 │ └───────────────────────────────┬────────────────────────────────────────┘   │
 │ ┌───────────────────────────────▼────────────────────────────────────────┐   │
 │ │ LEVEL 3  Upscaler / FG API hooks and re-implementations (the "Inputs")  │   │
 │ │  NVNGX (DLSS)        inputs/NVNGX*, NVNGX_Parameter                     │   │
 │ │  FSR2 / FSR3 / FFX   inputs/FSR2_*, FSR3_Dx12, FfxApi_*                 │   │
 │ │  XeSS                inputs/XeSS_*       + Xell_Hooks                   │   │
 │ │  Streamline (sl.*)   hooks/Streamline_Hooks  (slInit, slSetTag,         │   │
 │ │                      slSetConstants, slEvaluateFeature, ...)            │   │
 │ │  Reflex / NvAPI      hooks/Reflex_Hooks, nvapi/ (fakenvapi)             │   │
 │ │  FG inputs           inputs/FG, hooks/FG_Hooks                          │   │
 │ └───────────────────────────────┬────────────────────────────────────────┘   │
 │                                 ▼  normalized params (color, depth, MVs,     │
 │                                    jitter, exposure, reactive mask, ...)     │
 │ ┌────────────────────────────────────────────────────────────────────────┐   │
 │ │ CORE   Config (OptiScaler.ini) · State · Menu (ImGui) · ResTrack · Hudfix│  │
 │ │        FeatureProvider_{Dx11,Dx12,Vk} chooses backend from Config       │   │
 │ └───────────────────────────────┬────────────────────────────────────────┘   │
 │                                 ▼                                            │
 │ ┌────────────────────────────────────────────────────────────────────────┐   │
 │ │ LEVEL 4  Output backends (upscalers/ + library/)                        │   │
 │ │  IFeature_Dx11 · IFeature_Dx12 · IFeature_Vk                            │   │
 │ │  IFeature_Dx11wDx12 · IFeature_VkwDx12   (hidden DX12 device interop)   │   │
 │ │  FSR 2.1.2 / 2.2.1 / 3.x / 4 · XeSS · DLSS / DLSS-D                     │   │
 │ │  Frame gen (framegen/): FSR3-FG, XeFG, DLSS-G, OptiFG (reprojection)    │   │
 │ └───────────────────────────────┬────────────────────────────────────────┘   │
 └─────────────────────────────────┼────────────────────────────────────────────┘
                                   ▼
         Real D3D11 / D3D12 / DXGI / Vulkan runtime  →  GPU driver  →  GPU(s)
```

## 2. Where OptiScaler intercepts, by level

| Level | What is hooked | Why |
|---|---|---|
| 0. Process entry | Exports of a proxied system DLL (`dxgi.dll`, `version.dll`, ...) | The game loads OptiScaler by name from its own folder; the real DLL is forwarded. |
| 1. OS / loader | `LoadLibrary*`, Ntdll/Kernel functions, Wintrust/Crypt32 (signature verification), Gdi32, Advapi32 | Redirect loads of vendor upscaler DLLs and keep modified or unsigned DLLs accepted. `EarlyHooking` config installs these before the game's first loads. |
| 2. Graphics API | DXGI factory/adapter/swapchain; D3D12 device/resource/root-signature creation; D3D11; Vulkan instance/device/swapchain | Learn the device, queue and swapchain; track resources; draw the overlay at Present; spoof GPU identity; adapter selection happens here. |
| 3. Upscaler API | NVNGX (DLSS) exports, FSR2/3 and FFX API, XeSS, Streamline `sl*` functions, Reflex, NvAPI | This is the "Input": the game believes it is calling its chosen upscaler, but OptiScaler receives the call. |
| 4. Backend | Not hooked, this is OptiScaler's own code calling the chosen SDK | This is the "Output": the backend that actually runs. |

### Graphics API detail

- **DXGI** (`hooks/Dxgi_Hooks.cpp`, `DxgiFactory_Hooks.cpp`): `EnumAdapters`, `EnumAdapters1`, `EnumAdapterByLuid`,
  `EnumAdapterByGpuPreference`; `CreateSwapChain`, `CreateSwapChainForHwnd`, `CreateSwapChainForCoreWindow`; plus the
  DLSS-G variants of swapchain creation. Adapter enumeration is where the game picks its GPU, so this is the relevant
  layer for any multi-GPU logic.
- **Swapchain** (`hooks/FG_Hooks.cpp`): `Present`, `Present1`, `ResizeBuffers`, `ResizeTarget`, `SetFullscreenState`,
  `GetFrameLatencyWaitableObject`. Frame generation takes over presenting here.
- **D3D12** (`hooks/D3D12_Hooks.cpp`): `D3D12CreateDevice`, `CreateCommittedResource`, `CreatePlacedResource`,
  `GetResourceAllocationInfo`, `CreateSampler`, `CreateRootSignature`, `CheckFeatureSupport`, `SetResidencyPriority`,
  `D3D12GetInterface`. `ResTrack_dx12` and `CommandBuffer_StateTracker.h` track resources and command lists, which is
  how HUD-less color targets are found for Hudfix and OptiFG.
- **Vulkan** (`hooks/Vulkan_Hooks.cpp`): `vkCreateInstance`, `vkCreateDevice`, `vkGetInstanceProcAddr`,
  `vkGetDeviceProcAddr`, `vkCreateWin32SurfaceKHR`, `vkQueuePresentKHR`, `vkCreateSwapchainKHR`,
  `vkDestroySwapchainKHR`, `vkGetSwapchainImagesKHR`, `vkAcquireNextImageKHR`. `VulkanwDx12_Hooks` supports running
  DX12 backends for Vulkan games.
- **Streamline** (`hooks/Streamline_Hooks.cpp`): both SL1 and SL2+ interposers (`slInit`, `slSetTag`,
  `slSetConstants`, `slEvaluateFeature`, `slAllocateResources`, ...) so DLSS/DLSS-G/Reflex calls made through
  Streamline are captured.

## 3. Upscale call path

```
Game calls DLSS/FSR/XeSS/sl*  ──►  inputs/ or hooks/Streamline_Hooks
                                        │ translate params + resources
                                        ▼
                        FeatureProvider_{Dx11,Dx12,Vk}   (reads Config)
                                        │ creates
                                        ▼
                    IFeature_* backend  ──►  FSR / XeSS / DLSS SDK call on GPU
                                        │
                      output written back to the game's expected target
```

- **Same API, different upscaler:** a DX12 game goes through `IFeature_Dx12` with an FSR, XeSS or DLSS backend.
- **API bridge (`*wDx12`):** a DX11 or Vulkan game gets a hidden DX12 device. Inputs are shared as resources, the
  DX12-only backend runs there, and the result is copied back. README says this costs up to about 10%.

## 4. Frame generation path

```
Game FG call (FFX FG / DLSS-G / XeFG) or OptiFG (no game support)
   ├─ FG input capture    inputs/FG, hooks/FG_Hooks
   ├─ HUD-less color      resource_tracking/, hudfix/
   └─ IFGFeature_{Dx12,Vk}  ─►  ffx | xefg | dlssg | nvngx | reprojection
                                      │
                         swapchain proxy takes over Present
```

OptiFG is DX12-only. It works by wrapping the swapchain (`framegen/reprojection/swapchain`) and presenting generated
frames itself.

## 5. Supporting subsystems

| Area | Role |
|---|---|
| `Config.*` / `OptiScaler.ini` | All settings, readable and writable at runtime. |
| `State.h` | Shared global runtime state: device, swapchain, active features. |
| `menu/` | ImGui overlay (Insert key), drawn from the swapchain Present hook. DX11, DX12 and Vulkan backends. |
| `spoofing/`, `nvapi/` | Report spoofed GPU or vendor IDs via DXGI, Vulkan, NvAPI and user32. `nvapi/fakenvapi` provides an NvAPI shim. |
| `low_latency/`, `gpu_time/` | Reflex / AntiLag / LatencyFlex handling and GPU timing. |
| `fsr4/` | FSR4 DLL loading. |
| `scanner/` | Signature scanning. |
| `shaders/` | Compute shaders for resource copy and similar helpers. |

## 6. Multi-GPU notes (branch `feat/multigpu-support`)

Not analysed in this document. Points where GPU selection can enter the pipeline:

- DXGI adapter enumeration hooks (`DxgiFactory_Hooks`, `EnumAdapterByGpuPreference`, `EnumAdapterByLuid`).
- Device creation in `D3D12_Hooks` / `hkvkCreateDevice`.
- The hidden-device interop in `IFeature_*wDx12`, where a second device is created for DX11 or Vulkan games.
- The `spoofing/` layer, which controls what adapter identity the game sees.
