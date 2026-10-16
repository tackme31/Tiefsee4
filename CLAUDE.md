# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Tiefsee is an open-source image viewer for Windows built with C# WinForm (.NET 8) and TypeScript. The UI is entirely rendered inside a WebView2 control — almost all UI logic lives in the `Www` frontend project, not in C#.

## Sub-projects

| Project | Purpose |
|---|---|
| `Tiefsee` | Main WinForm app; hosts WebView2, runs a local HttpListener web server for JS↔C# communication |
| `TiefseeNativeHost` | Native C++ bootstrapper (`Tiefsee.exe`). Handles start modes and forwards launch args to an already-running instance via Named Pipe before the .NET CLR is loaded; otherwise hosts the CLR via hostfxr and calls `Tiefsee.Program` |
| `TiefseeAppPackager` | Packages output into MSIX for the Microsoft Store |
| `BuildAll` | Compiles Tiefsee + TiefseeNativeHost and zips the output |
| `Www` | Frontend source (EJS, TypeScript, SCSS, Rust/WASM); compiled output goes to `Output/Www` |

## Build Commands

### Frontend (Www)

First-time setup (from `Www/` directory):
```sh
npm i
rustup target add wasm32-unknown-unknown
cargo install wasm-pack
```

Build once:
```sh
cd Www
gulp build
```

Watch mode (no Rust rebuild — faster iteration):
```sh
cd Www
gulp watch
```

> `gulp build` outputs to `Output/Www/`. The C# project reads static files from there at runtime.

### C# Backend

Open `Tiefsee4.sln` in Visual Studio and build normally, or use `dotnet build Tiefsee/Tiefsee.csproj`.

`TiefseeNativeHost` is a C++ project (MSVC v143, x64); build it from Visual Studio or MSBuild. See `Building.md` for the required toolset.

> `dotnet build` cannot build the C++ native host — it silently skips it (prints "NativeHost 未重新編譯"), so `Output/Tiefsee.exe` is not produced/updated. Use Visual Studio's MSBuild via `BuildAll/InvokeMsBuild.ps1` whenever `Tiefsee.exe` is needed.

### Release package (zip)

Run `gulp build` in `Www/` first, then build `BuildAll` with Visual Studio's MSBuild (from the repo root, PowerShell):

```powershell
.\BuildAll\InvokeMsBuild.ps1 -Project .\BuildAll\BuildAll.csproj -Target Build   # Release|x64 by default
```

This builds Tiefsee (Release) + TiefseeNativeHost into `Output/`, removes pdb/BuildAll leftovers, and zips the whole `Output/` folder to `Tiefsee.zip` at the repository root (ignored by git). Stale files in `Output/` are included in the zip as-is, so delete everything in `Output/` except `Www/` beforehand if the layout may have changed.

- `InvokeMsBuild.ps1` does not restore NuGet packages. On a fresh clone or after deleting `obj/`, run `dotnet restore BuildAll/BuildAll.csproj -p:Platform=x64` and `dotnet restore Tiefsee/Tiefsee.csproj -p:Platform=x64` first.
- Before packaging, check that `Output/Www/` is populated (`js/`, `css/`, `MainWindow.html`, `img/logo.ico`). If it is missing, the app crashes on startup with `FileNotFoundException: ...\Www\img\logo.ico`. Re-run `gulp build`.
- Older versions of `Tiefsee.csproj` included `Output/Www/**` as Content. A stale `Tiefsee/obj/` from such a build makes MSBuild's IncrementalClean delete the files in `Output/Www/`. If they disappear after a C# build, delete `Tiefsee/obj/` and run `gulp build` again.

## Architecture

### C# Layer (`Tiefsee/`)

Follows a three-layer directory structure:

- **`App/`** — startup, assembly, and shared runtime state
  - `Bootstrap/AppBootstrapper.cs` — single entry point that wires `ServiceRegistry`
  - `Bootstrap/ServiceRegistry.cs` — holds shared service instances and calls `RegisterHttpRoutes()` for all feature modules
  - `InstancePipeServer.cs` / `InstancePipeClient.cs` / `InstancePipeProtocol.cs` — Named Pipe used to receive launch args from `TiefseeNativeHost` or another instance
  - `AppRuntimeContextBuilder.cs`, `EarlyAppPathResolver.cs`, `StartupConfigLoader.cs` — startup runtime context and paths
  - `SingleInstanceCoordinator.cs`, `StartMode.cs`, `AppScheduler.cs`, `PluginRegistry.cs`

- **`Features/`** — user-visible feature modules (Archive, File, Directory, Image, Window, SystemIntegration). Each feature can contain:
  - `Application/` — business logic
  - `Contracts/` — data models for the feature
  - `Http/` — local HTTP route handlers (called from JS via `fetch`)
  - `WebView/` — C# objects exposed to JS via WebView2's `AddHostObjectToScript`
  - `UI/` — WinForms UI specific to that feature

- **`Infrastructure/`** — reusable technical components (Win32/Shell/OS integration, extensions, caching, file format helpers, `Vendor/` for third-party source)

**Placement rule:** App = project startup/state; Features = user-visible functionality; Infrastructure = technical utilities that could be reused in any WinForms/WebView2 project. Details: `Docs/Tiefsee-Architecture-Conventions.zh_TW.md`.

### JS↔C# Communication

There are two channels:

1. **HTTP (fetch)**: JS calls `http://127.0.0.1:{port}/api/...`. Each feature registers routes via `*HttpEndpoints.RegisterRoutes()` called from `ServiceRegistry`. Route handlers inherit `HttpEndpointModuleBase` (`Infrastructure/Web/`). Internal API requests must carry the capability token (`Lib.getApiTokenHeaders()`; `WebAPI.ts` adds it for you), and cross-origin browser requests are rejected.

2. **WebView2 host objects**: C# classes (e.g. `WindowWebViewBridge`, `FileWebViewBridge`) are registered via `AddHostObjectToScript` and called synchronously from JS.

### Frontend (`Www/`)

- **Entry points**: `ts/MainWindow/MainWindow.ts` and `ts/SettingWindow/SettingWindow.ts` — bundled with esbuild
- **Workers**: `TiefseeviewWorker.ts`, `TiefseeviewWorkerSub.ts`
- **Templates**: EJS in `ejs/`, compiled to HTML in `Output/Www/`
- **Styles**: SCSS in `scss/`, compiled to CSS in `Output/Www/css/`
- **Icons**: SVGs in `img/default/` are bundled into `Output/Www/js/SvgList.js` via `gulp svg`. Use `var(--color-white)` in SVG fill to respect the active theme. Access in TS via `SvgList["name.svg"]`.
- **Rust/WASM**: Built with `wasm-pack` into `Www/wasm/`, then copied to output.

## Development Notes

- Most features can be developed by editing TypeScript/SCSS only — no C# changes needed.
- The recommended workflow is to open the `Www` folder in VS Code and run `gulp watch` for live recompilation.
- The `WebAPI.ts` file in `Www/ts/` is the TypeScript-side wrapper for all HTTP calls to the C# server.
- `gulp watch` skips the Rust build for speed — only run `gulp build` when WASM changes are needed.
