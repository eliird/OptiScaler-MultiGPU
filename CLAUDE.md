# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

OptiScaler is a Windows x64 DLL that is dropped next to a game (as `dxgi.dll`, `winmm.dll`, `version.dll`, `d3d12.dll`, `OptiScaler.asi`, etc. — see `OptiScaler/DllNames.h`). It hooks the game's graphics/NGX/Streamline/FFX calls and swaps DLSS/FSR/XeSS upscalers and frame generation for another backend. This checkout is on the `feat/multigpu-support` branch (a fork adding multi-GPU support) of upstream OptiScaler.

## Build

- Windows only: Visual Studio 2022, open `OptiScaler.sln` (single project `OptiScaler/OptiScaler.vcxproj`), or from a VS dev prompt: `msbuild /m /p:Configuration=Release OptiScaler.sln` (this is what CI in `.github/workflows/just_build.yml` runs).
- `external/` contains submodules (FidelityFX SDKs, XeSS, streamline, vulkan, spdlog, etc.); clone with `--recursive` or run `git submodule update --init --recursive`.
- There is no test suite or linter run in CI other than `clang-format.yml`. Formatting: `.clang-format` (LLVM base, 4-space indent, 120 columns).
- This repo is typically developed under WSL, where it cannot be compiled; build/verify on Windows.
- `setup_linux.sh` / `setup_windows.bat` are end-user install scripts for a built release, not dev tooling.
- Version is read from `OptiScaler/resource.h` by CI.

## Conventions (from CONTRIBUTING.md)

- Every `.cpp` must include `"pch.h"` as its first non-comment line; never include `pch.h` from a header; don't put utilities/macros in `pch.h` (use `SysUtils.h` or a new header). Only large stable third-party/system headers belong in `pch.h`.
- New source files must be added to `OptiScaler.vcxproj` (and `.vcxproj.filters`).

## Architecture

All code lives in `OptiScaler/`; entry is `dllmain.cpp`, which sets up config, logging, and installs hooks. Global runtime state is in `State.h`; settings are in `Config.h/.cpp` (backed by `OptiScaler.ini`, documented in `Config.md`).

Layers, roughly in call order:

- **`proxies/` + `exports/` + `Source.def`**: forwarding of the original system DLL's exports (when loaded under a spoofed name) and lazy resolution of real `d3d12`/`dxgi`/`kernel32`/`ntdll`/NGX/FFX functions.
- **`hooks/`**: Detours-based (`library/detours`) interception of D3D11/D3D12/DXGI/Vulkan, library loading (`LibraryLoad_Hooks` — redirects games' loads of nvngx/FFX/XeSS/Streamline DLLs), Reflex, Streamline, XeLL, AMD amdxc64, and FG entry points. `DxgiFactory_*` hooks are where adapter enumeration/selection happens, which is central to multi-GPU work. `wrapped/` holds COM wrapper objects.
- **`inputs/`**: Implementations of the APIs the game *calls* — NVNGX (DLSS), FSR2/FSR3, FFX API, XeSS — per graphics API (`_Dx11`, `_Dx12`, `_Vk`). These translate game inputs into the internal feature model. `inputs/FG` handles frame-gen inputs.
- **`upscalers/`**: The backend side. `IFeature` base classes (`IFeature_Dx11`, `_Dx12`, `_Vk`, and cross-API `_Dx11wDx12`, `_VkwDx12`) with `FeatureProvider_*` choosing and creating the active backend; concrete backends under `upscalers/dlss`, `dlssd`, and `library/` (fsr2, fsr2_212, fsr31, …). Cross-API variants run a DX12 backend for a DX11/Vulkan game via shared resources.
- **`framegen/`**: `IFGFeature` (+ Dx12/Vk) and implementations: `ffx`, `xefg`, `dlssg`, `nvngx`, `reprojection` (OptiFG).
- **`spoofing/`, `nvapi/`, `fsr4/`, `low_latency/`, `gpu_time/`, `hudfix/`, `resource_tracking/`, `scanner/`**: GPU/vendor spoofing (see `Spoofing.md`), NvAPI shims, FSR4 loading, Reflex/AntiLag/LatencyFlex, HUD-less resource detection, signature scanning.
- **`menu/`**: ImGui in-game overlay (INSERT key), per-API backends (`menu_dx11/dx12`, `menu_overlay_vk`); tabs built in `menu_common.cpp` (includes the Quirks tab and ASI plugin list). Vendored ImGui is in `include/imgui`.
- **`shaders/`**: HLSL/compute shaders compiled into the DLL for resource copy, reprojection, etc.

Key behaviors to keep in mind: the code runs inside arbitrary game processes, so hooks must be defensive about call order/threads (`OwnedMutex.h`), and most behavior is toggled via `Config` rather than compile-time flags.

## Docs in repo

`Config.md` (ini options), `Features.md`, `Spoofing.md`, `Issues.md`, `Changelog.md` — update `Config.md`/`Changelog.md` when adding user-facing settings or features.
