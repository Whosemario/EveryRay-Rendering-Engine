# CLAUDE.md — EveryRay Rendering Engine

This file provides context for AI assistants working in this codebase.

---

## Project Overview

EveryRay is a **Windows-only, DirectX 11/12 real-time rendering engine** written in C++17 and HLSL. It is a personal graphics prototyping sandbox (not a game engine). Key rendering techniques implemented include:

- Deferred PBR rendering with a GBuffer
- Cascaded shadow mapping (3 cascades)
- Voxel cone tracing (dynamic global illumination)
- Static indirect lighting via light probes (SH diffuse + cubemap specular)
- GPU-indirect drawing and GPU frustum culling
- Volumetric clouds, volumetric fog, SSR, SSS, bloom, tonemapping, FXAA
- GPU-tessellated terrain with LOD
- Foliage with wind simulation

---

## Repository Layout

```
/
├── source/
│   ├── EveryRay_Core/            # Static library: all engine logic
│   │   ├── RHI/                  # Rendering Hardware Interface (abstract)
│   │   │   ├── DX11/             # DirectX 11 implementation
│   │   │   └── DX12/             # DirectX 12 implementation
│   │   └── *.h / *.cpp           # ~120 engine source files
│   ├── EveryRay_Runtime_Win64_DX11/   # DX11 executable (entry point)
│   └── EveryRay_Runtime_Win64_DX12/   # DX12 executable (entry point)
├── content/
│   ├── shaders/                  # HLSL shaders (compiled at runtime)
│   │   ├── Common.hlsli          # Shared constants/macros
│   │   ├── GI/                   # Voxel cone tracing
│   │   ├── IBL/                  # Image-based lighting
│   │   ├── VolumetricClouds/
│   │   ├── VolumetricFog/
│   │   └── Terrain/
│   ├── levels/                   # JSON scene files
│   ├── models/                   # FBX/OBJ assets
│   └── textures/                 # PNG, JPG, DDS textures
├── external/                     # Third-party libraries (Assimp, ImGUI, etc.)
├── doc/                          # Engine and graphics documentation + UML
├── bin/x64/                      # Build output (dx11/ and dx12/)
├── graphics_config.json          # Quality presets
├── EveryRay_RenderingEngine_Win64_DX11.sln
└── EveryRay_RenderingEngine_Win64_DX12.sln
```

---

## Build System

- **Build tool:** Visual Studio 2019+ (MSBuild, `.vcxproj` / `.sln` files — no CMake)
- **Toolset:** MSVC v142, C++17, Unicode, x64 only
- **Windows SDK:** 10.0.19041.0+
- **Outputs:**
  - `bin/x64/dx11/{Debug,Release}/` — DX11 executable + assets
  - `bin/x64/dx12/{Debug,Release}/` — DX12 executable + assets
- **Projects:**
  - `EveryRay_Core_Win64_DX11.vcxproj` — Core static library (DX11 defines)
  - `EveryRay_Core_Win64_DX12.vcxproj` — Core static library (DX12 defines)
  - `EveryRay_Runtime_Win64_DX11.vcxproj` — DX11 executable
  - `EveryRay_Runtime_Win64_DX12.vcxproj` — DX12 executable
- **Key preprocessor defines:**
  - `ER_API_DX11` / `ER_API_DX12` — selected per project
  - `ER_PLATFORM_WIN64_DX11` / `ER_PLATFORM_WIN64_DX12`
- RTTI is disabled (`/GR-`) at the compiler level; the engine uses its own RTTI system.

To build: open the appropriate `.sln` file in Visual Studio, select configuration (Debug/Release), and build the Runtime project.

---

## Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Classes | `ER_` prefix + PascalCase | `ER_Camera`, `ER_GBufferMaterial` |
| Materials | `ER_*Material` | `ER_ShadowMapMaterial` |
| Managers | `ER_*Manager` | `ER_LightProbesManager` |
| Files | Match class name exactly | `ER_Camera.h` / `ER_Camera.cpp` |
| Namespace | `EveryRay_Core` for all engine code | — |
| Shader entry points | `VSMain`, `PSMain`, `CSMain`, `HSMain`, `DSMain`, `GSMain` | — |
| Alignment macros | `ER_ALIGN{8,16,32,64,128,256}` | `ER_ALIGN16 XMFLOAT4X4 mMatrix;` |

- All header files use `#pragma once`.
- One class per `.h`/`.cpp` file pair.

---

## Architecture

### Core Systems

| File | Role |
|---|---|
| `ER_Core.h/cpp` | Main update/draw loop, window management |
| `ER_RuntimeCore.h/cpp` | Extends `ER_Core` with input, editor, sandbox |
| `ER_Sandbox.h/cpp` | Holds all graphics systems + the active scene |
| `ER_Scene.h/cpp` | Scene data loaded from JSON |
| `ER_RenderingObject.h/cpp` | Generic renderable with meshes, materials, LODs |
| `ER_Material.h/cpp` | Abstract base for all materials |
| `RHI/ER_RHI.h` | Abstract GPU interface (buffers, textures, shaders, draw calls) |

### RTTI & Service Locator

The engine uses a custom RTTI system (compiler RTTI is off).

```cpp
// In header:
RTTI_DECLARATIONS(ER_Camera, ER_CoreComponent)

// In .cpp:
RTTI_DEFINITIONS(ER_Camera)

// Lookup pattern:
auto* cam = (ER_Camera*)core->GetServices().FindService(ER_Camera::TypeIdClass());
```

All major systems register themselves into `ER_CoreServicesContainer`. Retrieve them via `FindService()`.

### RHI Abstraction

`ER_RHI` is the abstract graphics interface. Never call DX11/DX12 APIs directly from rendering systems — always go through `ER_RHI`. The concrete implementation is created at startup based on the platform define.

```cpp
ER_RHI* rhi = (ER_RHI*)core->GetServices().FindService(ER_RHI::TypeIdClass());
rhi->Draw(vertexCount);
```

### Material System

- Derive from `ER_Material`.
- Standard materials implement `PrepareResourcesForStandardMaterial()`.
- Special materials (shadow, GBuffer, voxelization) are invoked by their respective systems, not through the standard material path.
- Materials are bound to objects in scene JSON via string names.

### Memory Cleanup Macros

```cpp
DeleteObject(ptr)               // delete + nullptr
DeleteObjects(ptr)              // delete[] + nullptr
ReleaseObject(comPtr)           // ->Release() + nullptr
DeletePointerCollection(vec)    // delete each element, clear vector
ReleasePointerCollection(vec)   // release each COM element, clear
```

Use these macros consistently — do not write raw `delete` without nulling the pointer.

### Constant Buffers

- Defined as `struct` with `ER_ALIGN_GPU_BUFFER` (256-byte alignment for DX12 compatibility).
- Keep structs aligned to 16 bytes using `ER_ALIGN16` on members.
- Must match exactly between C++ and HLSL.

---

## Rendering Pipeline (Frame Order)

1. **GPU Frustum Culling** (`ER_GPUCuller`) — marks visible objects / selects LODs
2. **Shadow Maps** (`ER_ShadowMapper`) — 3 cascades, runs `ER_ShadowMapMaterial`
3. **GBuffer** (`ER_GBuffer`) — albedo, normals, roughness, metalness, extra flags
4. **Direct Lighting** (`ER_Illumination`) — deferred PBR shading pass
5. **Static Indirect** (`ER_LightProbesManager`) — SH diffuse + cubemap specular
6. **Dynamic Indirect** (`ER_Illumination`) — voxel cone tracing passes
7. **Terrain** (`ER_Terrain`) — tessellated, forward rendered
8. **Foliage** (`ER_FoliageManager`) — forward rendered
9. **Post-Processing** (`ER_PostProcessingStack`) — volumetric clouds, fog, SSR, SSS, bloom, tonemapping, FXAA

---

## Shader Conventions (HLSL)

- Shared definitions in `content/shaders/Common.hlsli` — always include this for flag constants and common structs.
- Rendering object flags are bitmasks written to GBuffer extra target #2. The flag names are defined in both `ER_RenderingObject.h` and `Common.hlsli` — **keep them in sync**.
- Shaders are compiled at runtime via `ER_RHI::CompileShader()` / material `Initialize()`.
- Shader files live in `content/shaders/` and are loaded by path string in material code.

---

## Scene / Level Format

Scenes are JSON files in `content/levels/<scene_name>/scene.json`. Key fields:

```json
{
  "camera": { "position": [x,y,z], "near_plane": 0.1, "far_plane": 1000.0 },
  "directional_light": { "direction": [x,y,z], "color": [r,g,b] },
  "rendering_objects": [
    {
      "name": "MyObject",
      "model": "content/models/mymodel.fbx",
      "materials": ["ER_GBufferMaterial", "ER_ShadowMapMaterial"],
      "textures": { "albedo": "...", "normal": "...", "roughness": "..." },
      "transform": { "position": [x,y,z], "rotation": [x,y,z], "scale": [x,y,z] }
    }
  ],
  "light_probes": [...],
  "post_processing_volume": {...},
  "terrain": {...},
  "wind": {...}
}
```

See `doc/Engine_Overview.md` for the full schema.

---

## Graphics Configuration

`graphics_config.json` at the repo root defines quality presets ("ultra low" → "ultra high"):

| Key | Values | Meaning |
|---|---|---|
| `resolution_width/height` | integers | Window resolution |
| `texture_quality` | 0–2 | Texture mip/detail level |
| `shadow_quality` | 0–2 | Shadow map resolution |
| `gi_quality` | 0–2 | Voxel GI resolution/cascades |
| `foliage_quality` | 0–3 | Foliage density |
| `aa_quality` | 0–1 | Anti-aliasing |
| `volumetric_fog_quality` | 0–2 | Fog resolution |
| `volumetric_clouds_quality` | 0–3 | Cloud resolution |

---

## External Dependencies

| Library | Location | Purpose |
|---|---|---|
| Assimp 5.0.1 | `external/Assimp/` | Model loading (FBX, OBJ, etc.) |
| JsonCpp | `external/JsonCpp/` | Scene/config JSON parsing |
| ImGUI | `external/ImGUI/` | In-engine editor UI |
| ImGuizmo | `external/ImGUI/` | 3D manipulation gizmos |
| DirectXMath | `external/DirectXMath/` | SIMD math (vectors, matrices) |
| DirectXTK | `external/DirectXTK/` | DX11 utilities |
| DirectXTK12 | `external/DirectXTK12/` | DX12 utilities |
| DirectXTex | `external/DirectXTex/` | Texture loading/processing |
| WinPixEventRuntime | `packages/` | GPU profiling (PIX) |

Do **not** add new external dependencies without updating both `.vcxproj` files (DX11 and DX12) and `packages.config`.

---

## Key Files to Know

| File | Why Important |
|---|---|
| `source/EveryRay_Core/RHI/ER_RHI.h` | Abstract GPU interface — all rendering goes through this |
| `source/EveryRay_Core/ER_Core.h` | Main loop, window, service container |
| `source/EveryRay_Core/ER_Sandbox.h` | Top-level graphics system holder |
| `source/EveryRay_Core/ER_RenderingObject.h` | Per-object rendering state, flags, LODs |
| `source/EveryRay_Core/ER_Material.h` | Base material class |
| `source/EveryRay_Core/ER_Illumination.h` | Lighting calculations (direct + GI) |
| `content/shaders/Common.hlsli` | Shared HLSL definitions — keep in sync with C++ |
| `graphics_config.json` | Quality preset configuration |

---

## Important Constraints

1. **Windows-only.** No cross-platform abstractions. Do not introduce Linux/macOS paths.
2. **No unit tests.** Validate changes by running demo scenes (`testScene`, `sponzaScene`, `testScene_simple`, `terrainScene`).
3. **Dual-API.** Any change touching `ER_RHI` or graphics systems must be verified in **both** DX11 and DX12 builds. Changes that work in DX11 may fail in DX12 due to its more explicit resource state tracking.
4. **RTTI is disabled at the compiler level.** Use the custom `RTTI_DECLARATIONS`/`RTTI_DEFINITIONS` macros. Never use `dynamic_cast` or `typeid`.
5. **GBuffer flags must stay in sync** between `ER_RenderingObject.h` and `content/shaders/Common.hlsli`.
6. **Constant buffer alignment.** GPU constant buffers must be 256-byte aligned for DX12 (`ER_ALIGN_GPU_BUFFER`). Always pad structs to 16-byte boundaries.
7. **COM cleanup.** Use `ReleaseObject()` (not raw `->Release()`) for DirectX COM objects.

---

## Documentation

- `README.md` — Project introduction, feature list, controls, requirements, roadmap
- `doc/Engine_Overview.md` — Architecture, service locator pattern, scene JSON format, RHI
- `doc/Graphics_Overview.md` — Full frame breakdown, rendering algorithm details
- `doc/UML.drawio` — UML class diagram (open with draw.io)
- `CHANGE_LOG.md` — Version history
