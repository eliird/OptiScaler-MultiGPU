# AGENTS.md

Fork of [optiscaler/OptiScaler](https://github.com/optiscaler/OptiScaler) — a C++/MSVC proxy DLL that replaces upscalers/frame-gen in games. Working branch `feat/multigpu-support` (currently identical to `master`); git history is shared with upstream.

## Build (Windows only — cannot be verified in this Linux checkout)
- Visual Studio 2022 (v143), x64 only (Win32 targets were removed), `stdcpplatest`. One project: `OptiScaler.sln` -> `OptiScaler/OptiScaler.vcxproj` -> `OptiScaler.dll`.
- Submodules are mandatory: `git submodule update --init --recursive`. `external/` mixes 9 submodules (xess, FidelityFX-SDK, FidelityFX-SDK-v2, nvapi, spdlog, ...) with vendored trees (freetype, nlohmann, streamline, nvngx_dlss_sdk, ...); the Release post-build copies prebuilt DLLs out of the submodules, so a partially initialized `external/` fails the build.
- CI builds with `msbuild /m /p:Configuration=Release .` from the repo root (see `.github/workflows/build.yml`).
- Configurations: `Release` (post-build assembles the shippable package in `x64/Release/a/` incl. bundled XeSS/FidelityFX DLLs, `OptiScaler.ini`, setup scripts), `ReleaseDebug` (optimized + debug info, post-build writes timestamped .7z), `Debug`.
- Pre-build generates `OptiScaler/resource_build_date.h` and `resource_build_commit.h` — gitignored, never commit or hand-edit them.
- This environment has no MSVC and no clang-format installed; compile verification is only possible via Windows/CI. Validate locally by review + formatting discipline.

## Lint / format (the only check that runs automatically on push/PR)
- CI (.github/workflows/clang-format.yml) runs **clang-format 22** over `OptiScaler/`, excluding `OptiScaler/include/` (vendored third-party: imgui, detours, fsr*, ...). `external/` is not checked.
- Style (`.clang-format`): 4-space indent, Allman braces, 120 cols, `SortIncludes: false` — never reorder includes.
- Approximate local check: `find OptiScaler -type f \( -name '*.cpp' -o -name '*.h' \) -not -path 'OptiScaler/include/*' -print0 | xargs -0 clang-format --dry-run -Werror`

## Code conventions (from CONTRIBUTING.md — PCH discipline is enforced)
- Every `.cpp` must include `"pch.h"` as its very first include. Never include `pch.h` from a header.
- Do not dump utilities/macros/globals into `pch.h`; use `SysUtils.h` or a new functional header. Only large, stable system/third-party headers belong in `pch.h`.

## Testing
- There is **no test suite or test runner**. `.github/workflows/test.yml` is misnamed — it is a manual nightly-release helper, not tests. Build workflows are `schedule`/`workflow_dispatch` only; don't expect CI to build your push.

## Architecture (worth knowing before editing)
- `OptiScaler.dll` is loaded into the game as a proxy (`dxgi.dll`, `version.dll`, `d3d12.dll`, ... — see `OptiScaler/exports/`, `Source.def`, `DllNames.h`); `proxies/` + `hooks/` forward to the real system DLLs.
- Data flow: **Inputs** (game-facing APIs in `inputs/`: NVNGX/DLSS, FSR2/3, XeSS, FfxApi) -> **Outputs** (backends in `upscalers/` behind `IFeature` / `FeatureProvider_*`, one per graphics API: Dx11, Dx11wDx12, Dx12, Vk, VkwDx12). Frame generation in `framegen/` (`IFGFeature_Dx12` / `_Vk`), ImGui overlay in `menu/`.
- Global runtime state: `State.h` singleton. Settings load from `OptiScaler.ini` via `Config::Instance()` (`Config.h/.cpp`); parameter docs live in `Config.md`.
- `misc/GameConfigs.*` applies per-game patches/quirks keyed by executable name.
- `OptiScaler/include/` is vendored third-party code — don't modify except when deliberately updating a dependency.

## Misc
- Version numbers live in `OptiScaler/resource.h` (`VER_MAJOR_VERSION` etc.); CI parses these for artifact names.
- `setup_windows.bat`, `setup_linux.sh`, `Streamlined_fetcher_windows.bat` are end-user deployment/fetch scripts (shipped in release packages) — **not** developer environment setup.
- Runtime troubleshooting/feature docs: `Features.md`, `Spoofing.md`, `Issues.md`, `Changelog.md`. Don't duplicate config documentation; point to `Config.md`.