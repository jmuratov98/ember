# Graphics API survey for Ember's backends

Research for [jmuratov98/ember#3](https://github.com/jmuratov98/ember/issues/3). Gathered 2026-11-08 from primary sources only (vendor docs, Khronos, Apple, and the projects' own repos and headers). GitHub links are pinned to the commit read on that date.

## Question

What facts are Ember's backend decisions waiting on? Ember is a modular 3D engine written in C for Windows, macOS and Linux desktop. Each render backend is a dynamic library. The planned backends are Vulkan, Metal and Direct3D; OpenGL is undecided. This file covers five areas:

1. Direct3D 11 vs 12.
2. Vulkan on macOS: MoltenVK, KosmicKrisp and native Metal.
3. The state of OpenGL.
4. Shader authoring and cross-compilation.
5. Prior art for putting several graphics APIs behind one interface.

It feeds four decision tickets: "Is OpenGL a v1 backend?", "Direct3D 11 or 12?", "Shading language and shader compilation", and "The renderer's interface to its backends". It records facts and trade-offs only. It makes no decisions.

## Summary

- **D3D12 is the only Direct3D API that gets new GPU features.** Shader Model 6.x (DXIL), ray tracing, mesh shaders, work graphs and enhanced barriers are all D3D12-only. D3D11 stops at Shader Model 5.0 (DXBC, compiled by FXC). D3D12's objects map almost one-to-one onto Vulkan's: command list = command buffer, PSO = pipeline, root signature = pipeline layout, descriptor table = descriptor set, resource barrier = pipeline barrier. D3D11 keeps implicit synchronization and per-stage slot binding, which is closer to classic Metal and OpenGL. Two multi-backend libraries have dropped D3D11 recently: SDL3 GPU (Nov 2024) and The Forge (Feb 2025).
- **D3D12 has more debugging tools, and its runtime loading affects a DLL backend.** PIX, GPU-based validation and DRED are D3D12-only; RenderDoc covers both D3D11 and D3D12. The Agility SDK runtime is normally chosen by exports from the main `.exe`. A newer `ID3D12SDKConfiguration1::CreateDeviceFactory` path, which appears in the shipping headers, lets one component load its own runtime instead.
- **MoltenVK 1.4.2 (July 2026) is an "almost-complete subset" of Vulkan 1.4 and is not fully conformant.** It runs on macOS 12+ with any Mac GPU. It has no geometry shaders, mesh shaders, ray tracing or pipeline-statistics queries, and apps must opt in to portability enumeration. It converts SPIR-V to MSL at runtime with SPIRV-Cross.
- **KosmicKrisp is LunarG's Mesa driver that layers Vulkan over Metal 4.** It became Vulkan 1.4 conformant in September 2026 and ships in Vulkan SDK 1.4.363.0. It needs Apple silicon and macOS 26 or later. It advertises geometry and tessellation shaders but no mesh shaders or ray tracing. It is MIT-licensed, and GPU-assisted validation works with it but not with MoltenVK.
- **Native Metal has features neither Vulkan layer exposes yet.** It has had ray tracing since macOS 11 and mesh shaders since macOS 13. Metal 4 (macOS 26, M1 or later) adopts an explicit model built to ease ports from D3D12 and Vulkan: command allocators, argument tables, untracked resources and explicit barriers.
- **OpenGL on macOS stops at 4.1 and has been deprecated since macOS 10.14 (2018).** 4.1 lacks everything that became core in 4.2–4.6: image load/store, compute shaders, SSBOs, `KHR_debug` debug output, buffer storage, direct state access (DSA), clip control and SPIR-V. On Linux, Mesa provides GL 4.6 on most drivers. Windows' own OpenGL is 1.1; newer versions come from GPU vendors' drivers (ICDs). RenderDoc does not run on macOS.
- **SPIR-V is the hub format for shader tooling.** DXC compiles HLSL to DXIL or SPIR-V. glslang and shaderc compile GLSL to SPIR-V. SPIRV-Cross turns SPIR-V into MSL, GLSL or HLSL. Slang targets all of them. Tools with a C API: glslang, shaderc, SPIRV-Cross (ABI-stable), `D3DCompile`, Metal Shader Converter and vkd3d-shader. DXC and Slang have C++/COM-style APIs only. glslang deprecated its HLSL front end in April 2026, and DirectX plans to accept SPIR-V from Shader Model 7.
- **D3D11 needs DXBC, which only FXC (Windows-only) or vkd3d-shader (LGPL) can produce.** Apple's Metal Shader Converter turns DXIL (SM 6.0–6.6) into metallib and has a C API.
- **Prior art ranges from thick interfaces to thin ones.** Thick: bgfx (views and sort keys, no command buffers), sokol_gfx (passes, no command buffers) and Kohi (about 90 backend function pointers built around renderer-level concepts). Thin: The Forge (explicit command buffers, barriers and descriptor sets). In between: SDL3 GPU (explicit command buffers and passes, no barriers). Kohi is closest to Ember's structure: it is C99, and its Vulkan backend is a shared-library plugin that fills a function-pointer struct.

---

## 1. Direct3D 11 vs Direct3D 12

### Feature sets and complexity

| | Direct3D 11 | Direct3D 12 |
|---|---|---|
| Programming model | The runtime synchronizes CPU and GPU and checks hazards implicitly. `Map(WRITE_DISCARD)` renames buffers for you. Lifetimes are reference-counted ([MS: Important changes 11→12](https://learn.microsoft.com/en-us/windows/win32/direct3d12/important-changes-from-directx-11-to-directx-12)) | The app owns CPU/GPU sync, hazard tracking, dynamic-update buffering, memory residency, resource barriers and object lifetimes ([same page](https://learn.microsoft.com/en-us/windows/win32/direct3d12/important-changes-from-directx-11-to-directx-12)) |
| Work submission | Immediate context plus deferred contexts. Microsoft says deferred contexts "don't map perfectly to hardware" ([MS](https://learn.microsoft.com/en-us/windows/win32/direct3d12/important-changes-from-directx-11-to-directx-12)) | Self-contained command lists, bundles and command queues ([MS](https://learn.microsoft.com/en-us/windows/win32/direct3d12/important-changes-from-directx-11-to-directx-12)) |
| Pipeline state | Independent state objects (input assembler, rasterizer, output merger, shaders), resolved at draw time | Immutable pipeline state objects (PSOs) |
| Resource binding | Views bound to per-stage slots | Descriptor heaps and tables plus root signatures. Shaders can index resources dynamically |
| Minimum hardware | Feature levels 9_1 through 11_1 have shader profiles ([MS: compiler targets](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/specifying-compiler-targets)) | "All Direct3D 12 drivers will be Feature Level 11_0 or better"; "The DirectX 12 API only goes down to feature level 11_0" ([MS: feature levels](https://learn.microsoft.com/en-us/windows/win32/direct3d12/hardware-feature-levels)) |
| Shader bytecode | SM 4.x/5.0 profiles (`vs_5_0`, `cs_5_0`, …) as DXBC from `D3DCompile` (FXC) ([MS](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/specifying-compiler-targets)). "DirectX 11 hardware only supports up to shader model 5.0" ([MS](https://learn.microsoft.com/en-us/windows/win32/direct3d12/hardware-feature-levels)) | SM 5.1, "introduced with D3D12" ([MS](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/shader-model-5-1)), plus SM 6.x as DXIL from DXC. SM 6.0 wave ops are "requirements for DirectX Feature Level 12.0" ([MS](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/hlsl-shader-model-6-0-features-for-direct3d-12)). SM 6.9 is retail in Agility SDK 1.619; SM 6.10 is in preview ([Agility SDK page](https://devblogs.microsoft.com/directx/directx12agility/)) |
| Newer rendering features | Direct3D 11.3 shares conservative rasterization, volume tiled resources, ROVs and shader-specified stencil reference with D3D12 ([MS: What is D3D12](https://learn.microsoft.com/en-us/windows/win32/direct3d12/what-is-directx-12-)) | DirectX 12 Ultimate adds DXR 1.1, variable rate shading, mesh shaders and sampler feedback ([DirectX blog, 2020-03-19](https://devblogs.microsoft.com/directx/announcing-directx-12-ultimate/)). Later: enhanced barriers (Agility SDK 1.608), work graphs (1.613), fence barriers (1.620.1) ([Agility SDK page](https://devblogs.microsoft.com/directx/directx12agility/)) |
| Update channel | Ships with the OS | Agility SDK: a redistributable `D3D12Core.dll` from NuGet. Latest retail is 1.620.1 (2026-10-28); latest preview is 1.722.0 ([Agility SDK page](https://devblogs.microsoft.com/directx/directx12agility/)). Needs Windows 10 1909 or later and is "not supported on OSes before Windows 10" ([Getting started](https://devblogs.microsoft.com/directx/gettingstarted-dx12agility/)) |
| Microsoft's guidance | "Direct3D 11 continues to be a viable option alongside Direct3D 12" ([MS](https://learn.microsoft.com/en-us/windows/win32/direct3d12/what-is-directx-12-)) | "designed for advanced graphics programmers … calls for significant graphics expertise". It has about 200 functions. "Driver support and ease of debugging remain on a par with Direct3D 11" ([MS](https://learn.microsoft.com/en-us/windows/win32/direct3d12/what-is-directx-12-)) |
| Use from C | Not checked in this survey | `d3d12.h` ships a generated C interface: `CINTERFACE`, `lpVtbl` and `COBJMACROS` macros such as `ID3D12SDKConfiguration1_CreateDeviceFactory(...)` ([DirectX-Headers d3d12.h](https://github.com/microsoft/DirectX-Headers/blob/75edcf75f4c0942dd0d39e75fdc5c42ca6f285fd/include/directx/d3d12.h#L2019-L2128), MIT) |

### Tooling and debugging

| Tool | D3D11 | D3D12 | Source |
|---|---|---|---|
| Debug layer | Yes | Yes | [MS GBV page](https://learn.microsoft.com/en-us/windows/win32/direct3d12/using-d3d12-debug-layer-gpu-based-validation) |
| GPU-based validation (GBV) | — | Catches bad or stale descriptors, incompatible resource states and out-of-bounds heap indexing at GPU time. "can slow things down a lot" | [MS GBV](https://learn.microsoft.com/en-us/windows/win32/direct3d12/using-d3d12-debug-layer-gpu-based-validation) |
| DRED (Device Removed Extended Data) | — | Automatic breadcrumbs (costs 2–5%, off by default) and GPU page-fault reporting | [MS DRED](https://learn.microsoft.com/en-us/windows/win32/direct3d12/use-dred) |
| PIX | — | "for … developers using DirectX 12"; GPU captures are for D3D12 | [PIX intro](https://devblogs.microsoft.com/pix/introduction/) |
| RenderDoc v1.47 (2026-10-31) | Yes | Yes. It also supports Vulkan and GL 3.2–4.6 core on Windows/Linux. Metal: N/A | [RenderDoc README](https://github.com/baldurk/renderdoc/blob/2835a735cde545f5f6c7098f0ecf02db09350381/README.md) |

### How each maps onto Vulkan and Metal

Khronos's "decoder ring" maps Vulkan terms to DirectX and Metal terms ([Vulkan-Guide decoder_ring.adoc](https://github.com/KhronosGroup/Vulkan-Guide/blob/b02c0f42e821b6f9fb99acd850c5f3a9f8936ca2/chapters/decoder_ring.adoc); the file is © 2019–2021, so it predates Metal 4). The D3D11 and Metal 4 columns come from the cited Microsoft and Apple pages.

| Concept | D3D11 | D3D12 | Vulkan | Metal (classic) | Metal 4 (macOS 26) |
|---|---|---|---|---|---|
| Command recording | immediate/deferred context | command list + command allocator | command buffer + command pool | command buffer (from queue) + encoders | `MTL4CommandBuffer` + `MTL4CommandAllocator`, reusable; queues commit buffers in groups ([Apple](https://developer.apple.com/documentation/metal/understanding-the-metal-4-core-api)) |
| Pipeline state | separate state objects | PSO | pipeline | pipeline state | pipeline state |
| Binding layout | fixed per-stage slots | root signature / root parameters | pipeline layout / set-layout bindings, push descriptors | arguments in shader parameter list | — |
| Bound resource groups | views in slots | descriptor table in descriptor heap | descriptor set from descriptor pool | argument buffer; per-encoder `set*` binding | argument tables bound to stages; encoders have no per-resource bind methods ([Apple](https://developer.apple.com/documentation/metal/understanding-the-metal-4-core-api)) |
| Hazards | implicit (runtime) | resource barriers, resource states | pipeline barriers, image layouts | tracked by default for resources created with `MTLHazardTrackingMode.tracked` ([Apple](https://developer.apple.com/documentation/metal/mtlhazardtrackingmode)) | "the framework considers all resources untracked"; explicit barriers ([Apple](https://developer.apple.com/documentation/metal/understanding-the-metal-4-core-api)) |
| GPU→CPU sync | implicit | `ID3D12Fence` | fence / timeline semaphore | completed handler / event | `MTLSharedEvent` / events |
| Render targets | render target views | collection of resources | framebuffer / dynamic rendering | `MTLRenderPassDescriptor` | render encoders can suspend and resume a pass across command buffers |
| Amplification / mesh | — | amplification / mesh shader | task / mesh shader | object / mesh shader | object / mesh shader |
| Tessellation | hull / domain | hull / domain | tessellation control / evaluation | tessellation compute kernel + post-tessellation vertex shader | `MTL4RenderCommandEncoder` drops the tessellation methods and suggests mesh shaders instead ([Apple](https://developer.apple.com/documentation/metal/understanding-the-metal-4-core-api)) |

Related: Vulkan's new `VK_EXT_descriptor_heap` (proposal © 2021–2026) moves descriptor management into app-managed memory, similar to D3D12 heaps. It also lets you create descriptors from images without image views ([proposal](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/proposals/VK_EXT_descriptor_heap.adoc); [Vulkan-Guide chapter](https://github.com/KhronosGroup/Vulkan-Guide/blob/b02c0f42e821b6f9fb99acd850c5f3a9f8936ca2/chapters/descriptor_heap.adoc)).

### Loading D3D12 from a backend DLL

- **The classic Agility SDK setup is per-process.** The app exports `D3D12SDKVersion` and `D3D12SDKPath`, and "these parameters need to be exported from the main .exe for the process" ([Getting started](https://devblogs.microsoft.com/directx/gettingstarted-dx12agility/)).
- **The "Independent Devices" spec adds a per-component path.** `ID3D12SDKConfiguration1::CreateDeviceFactory(SDKVersion, SDKPath, …)` "allows loading a side-by-side D3D12Core … This API does not require developer mode" ([DirectX-Specs: Independent Devices](https://microsoft.github.io/DirectX-Specs/d3d/IndependentDevices.html)). The interface exists in the current `d3d12.h` ([L34868](https://github.com/microsoft/DirectX-Headers/blob/75edcf75f4c0942dd0d39e75fdc5c42ca6f285fd/include/directx/d3d12.h#L34868)).

### What other projects chose

- **SDL3 GPU dropped its D3D11 backend before release** ([PR #11456, merged 2024-11-12](https://github.com/libsdl-org/SDL/pull/11456)). The reasons given: deferred contexts "with … spotty hardware support", forced feature compromises, and a small, shrinking share of hardware without D3D12 or Vulkan. The same change lets D3D12 accept DXBC SM 5.1, so apps can use the lighter `D3DCompile` instead of DXC.
- **The Forge removed DirectX 11 "with the retirement of Windows 10"** (release 1.61, 2025-02-13). Earlier, in release 1.60 (2024-10-11), it stopped officially supporting Vulkan on Windows: "the best option to ship a PC / Windows game is DirectX 12". The reasons given were QA effort and reliability ([The Forge README](https://github.com/ConfettiFX/The-Forge/blob/cd5046893faba2dc7869243873bf01f02a6f0df9/README.md#L214-L263)).
- **bgfx still ships both D3D11 and D3D12; sokol_gfx ships D3D11 only** (see §5).

---

## 2. Vulkan on macOS: MoltenVK, KosmicKrisp and native Metal

| | MoltenVK 1.4.2 | KosmicKrisp | Native Metal |
|---|---|---|---|
| What it is | Layered Vulkan implementation on Metal by Khronos/Brenwill ([README](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/README.md)) | LunarG's Mesa driver layered on Metal 4 ([LunarG product page](https://www.lunarg.com/products/kosmickrisp/); [Mesa docs](https://docs.mesa3d.org/drivers/kosmickrisp.html)) | Apple's API |
| Version / status | Released 2026-07-20 ([Whats_New](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/Docs/Whats_New.md); [GitHub release](https://github.com/KhronosGroup/MoltenVK/releases/tag/v1.4.2)). Vulkan 1.4 support arrived in v1.4.0 (2025-08-20) ([release](https://github.com/KhronosGroup/MoltenVK/releases/tag/v1.4.0)). 1.4.3 is unreleased | Vulkan 1.3 conformant on 2025-10-30 ([LunarG](https://www.lunarg.com/lunarg-achieves-vulkan-1-3-conformance-with-kosmickrisp-on-apple-silicon/)), then Vulkan 1.4 conformant in Sept 2026 ([LunarG](https://www.lunarg.com/kosmickrisp-achieves-vulkan-1-4-conformance-on-apple-silicon/); the post body says "2025" but the header, the [product page](https://www.lunarg.com/products/kosmickrisp/) and the SDK notes say 2026). The source reports API version 1.4 ([kk_physical_device.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/db6c407405a4353a1029704a344b565cf7f1fa48/src/kosmickrisp/vulkan/kk_physical_device.c#L33-L41)) | Metal 4 on macOS 26 ([Apple](https://developer.apple.com/documentation/metal/mtl4commandqueue)) |
| Conformance | "an almost-complete subset of … Vulkan 1.4"; "not fully compliant" ([README §compliance](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/README.md#L382)) | Khronos-conformant Vulkan 1.4 ([LunarG](https://www.lunarg.com/kosmickrisp-achieves-vulkan-1-4-conformance-on-apple-silicon/)). LunarG warns that "conformance does not guarantee application compatibility" ([LunarG 1.3 post](https://www.lunarg.com/lunarg-achieves-vulkan-1-3-conformance-with-kosmickrisp-on-apple-silicon/)) | — |
| OS and hardware | macOS 12+ at runtime ([User Guide](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/Docs/MoltenVK_Runtime_UserGuide.md)). Intel, AMD and Apple GPUs | Apple silicon only, macOS 26+ (needs Metal 4). No Intel Macs; iOS "not enabled or supported" ([LunarG](https://www.lunarg.com/kosmickrisp-achieves-vulkan-1-4-conformance-on-apple-silicon/)) | Metal 3: macOS 13 on M1+, AMD Vega/5000/6000, Intel Iris Pro and UHD 630 ([Apple](https://developer.apple.com/documentation/metal/mtlgpufamily/metal3)). Metal 4: "Apple silicon (M1 or later)" ([Apple Metal page](https://developer.apple.com/metal/)) |
| Distribution | Vulkan SDK, or bundled as a dylib or XCFramework ([User Guide](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/Docs/MoltenVK_Runtime_UserGuide.md)) | Vulkan SDK 1.4.363.0 (released 2026-09-29) lists it as "a Vulkan 1.4-conformant ICD in the SDK" ([SDK notes](https://www.lunarg.com/lunarg-releases-vulkan-sdk-1-4-363-0/)). Also upstream Mesa | System framework |
| Portability | Exposes `VK_KHR_portability_subset`. The loader lists it only if the instance sets `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR` ([User Guide](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/Docs/MoltenVK_Runtime_UserGuide.md#L456-L462)) | `VK_KHR_portability_subset` is not in its device extension table ([source](https://gitlab.freedesktop.org/mesa/mesa/-/blob/db6c407405a4353a1029704a344b565cf7f1fa48/src/kosmickrisp/vulkan/kk_physical_device.c#L46-L210)) | — |
| Geometry shaders | `geometryShader` is never set, so false ([MVKDevice.mm initFeatures](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/MoltenVK/MoltenVK/GPUObjects/MVKDevice.mm#L2795-L2865)) | `.geometryShader = true` ([source L236](https://gitlab.freedesktop.org/mesa/mesa/-/blob/db6c407405a4353a1029704a344b565cf7f1fa48/src/kosmickrisp/vulkan/kk_physical_device.c#L236)) | No geometry stage. Slang's Metal target lists vertex, mesh, task, tessellation and fragment stages ([Slang targets](https://github.com/shader-slang/slang/blob/ae4e57957a0adad3061b069b0e0eec8217779338/docs/user-guide/09-targets.md)) |
| Tessellation | Yes on Apple3+ and Mac1 GPUs ([MVKDevice.mm](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/MoltenVK/MoltenVK/GPUObjects/MVKDevice.mm#L2838-L2862)) | Yes ([source](https://gitlab.freedesktop.org/mesa/mesa/-/blob/db6c407405a4353a1029704a344b565cf7f1fa48/src/kosmickrisp/vulkan/kk_physical_device.c#L261)) | Compute kernel + post-tessellation vertex function ([decoder ring](https://github.com/KhronosGroup/Vulkan-Guide/blob/b02c0f42e821b6f9fb99acd850c5f3a9f8936ca2/chapters/decoder_ring.adoc)) |
| Mesh shaders | No `VK_EXT_mesh_shader` in the extension list ([User Guide](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/Docs/MoltenVK_Runtime_UserGuide.md#L232)) | Not in the extension table | Yes, macOS 13+ ([Apple](https://developer.apple.com/documentation/metal/mtlrendercommandencoder/drawmeshthreadgroups(_:threadsperobjectthreadgroup:threadspermeshthreadgroup:))) |
| Ray tracing | No `VK_KHR_ray_*` | Not in the extension table | `MTLAccelerationStructure`, macOS 11+ ([Apple](https://developer.apple.com/documentation/metal/mtlaccelerationstructure)) |
| Other known gaps | No pipeline-statistics queries. `VkAllocationCallbacks` ignored. PVRTC needs host-mapped upload ([§Limitations](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/Docs/MoltenVK_Runtime_UserGuide.md#L681)). Primitive restart always on; no `VK_POLYGON_MODE_POINT` ([extension notes](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/Docs/MoltenVK_Runtime_UserGuide.md#L232)). `logicOp` and `wideLines` only in builds that use private Metal APIs ([MVKDevice.mm](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/MoltenVK/MoltenVK/GPUObjects/MVKDevice.mm#L2802-L2806)) | Some features sit behind `MESA_KK_EXPERIMENTAL`: custom border colour, image-view min LOD ([Mesa docs](https://docs.mesa3d.org/drivers/kosmickrisp.html)) | — |
| Shader path | SPIR-V converted to MSL at `vkCreateShaderModule` via SPIRV-Cross. The pipeline cache can store converted MSL to speed loading ([User Guide §shaders](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/Docs/MoltenVK_Runtime_UserGuide.md)) | "Shader translation through the Mesa compiler stack into Metal Shading Language" ([product page](https://www.lunarg.com/products/kosmickrisp/)) | MSL source at runtime, or a precompiled metallib (§4) |
| Validation and debugging | Validation layers through the Vulkan SDK loader; Metal frame capture ([User Guide](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/Docs/MoltenVK_Runtime_UserGuide.md)). GPU-assisted validation and Debug Printf "do not work with MoltenVK" ([SDK 1.4.363.0](https://www.lunarg.com/lunarg-releases-vulkan-sdk-1-4-363-0/)) | GPU-assisted validation and Debug Printf work ([SDK 1.4.363.0](https://www.lunarg.com/lunarg-releases-vulkan-sdk-1-4-363-0/)); Metal GPU capture through `MESA_KK_GPU_CAPTURE` ([Mesa docs](https://docs.mesa3d.org/drivers/kosmickrisp.html)) | Xcode Metal tools; RenderDoc has no Metal or macOS support ([RenderDoc](https://github.com/baldurk/renderdoc/blob/2835a735cde545f5f6c7098f0ecf02db09350381/README.md)) |
| License | Apache 2.0 | MIT ([product page](https://www.lunarg.com/products/kosmickrisp/)) | proprietary |

Notes:

- **The pinned User Guide already lists 1.4.3 work.** It includes extensions the unreleased 1.4.3 adds, such as `VK_EXT_multi_draw`, `VK_EXT_nested_command_buffer` and `VK_EXT_depth_clip_enable` ([Whats_New](https://github.com/KhronosGroup/MoltenVK/blob/5189f8de86ec365860f63bf1da736815cfaea5cd/Docs/Whats_New.md)).
- **LunarG's January 2026 guidance:** use MoltenVK for "older Intel Macs", iOS and older OSes, and KosmicKrisp for Apple silicon on macOS 26+. It also recommends bundling tested loader and driver libraries rather than relying on a system-wide loader ([LunarG, Jan 2026](https://www.lunarg.com/the-state-of-vulkan-on-apple-jan-2026/)).
- **Apple's stated goal for Metal 4:** it "makes it easier to adapt your apps and games from other platforms, such as DirectX and Vulkan" ([Apple](https://developer.apple.com/documentation/metal/understanding-the-metal-4-core-api)).

---

## 3. OpenGL

### What macOS's OpenGL 4.1 lacks

The version column comes from the GL registry, which records which version made each entry point core ([gl.xml](https://github.com/KhronosGroup/OpenGL-Registry/blob/6af574a14089ccfee87efe230ebcdd8742859813/xml/gl.xml#L32794)).

| Capability | Core since | Example entry points | On macOS (4.1 core) |
|---|---|---|---|
| Separable programs, program binaries | 4.1 | `glGenProgramPipelines`, `glGetProgramBinary` | yes |
| Image load/store, memory barriers, immutable texture storage, base instance | 4.2 | `glBindImageTexture`, `glMemoryBarrier`, `glTexStorage2D` | not core. `ARB_texture_storage` was listed as an extension in Apple's tables |
| **Compute shaders, SSBOs, debug output (`KHR_debug`), object labels, texture views, separate vertex-attribute binding, multi-draw indirect, image copy** | 4.3 | `glDispatchCompute`, `glShaderStorageBlockBinding`, `glDebugMessageCallback`, `glObjectLabel`, `glTextureView`, `glBindVertexBuffer`, `glMultiDrawElementsIndirect`, `glCopyImageSubData` | no |
| Persistent/immutable buffer storage, multi-bind | 4.4 | `glBufferStorage`, `glBindTextures` | no |
| **Direct state access (DSA)**, clip control (0..1 depth) | 4.5 | `glCreateBuffers`, `glNamedBufferStorage`, `glCreateTextures`, `glClipControl` | no |
| SPIR-V shaders, indirect count | 4.6 | `glSpecializeShader`, `glMultiDrawArraysIndirectCount` | no |

- **Apple's version ceiling:** its support page lists OpenGL 4.1 as the highest version on every Mac it covers. The list covers Intel-era models through 2019 ([Apple Support 101525](https://support.apple.com/en-us/101525)).
- **Debugging extensions on macOS:** Apple's capability tables for macOS 10.9 core profiles list only `EXT_debug_label` and `EXT_debug_marker`. They show no `KHR_debug`/`ARB_debug_output`, compute or SSBO extensions ([Apple OpenGL Capabilities Tables, dated 2017-6-5](https://developer.apple.com/opengl/OpenGL-Capabilities-Tables.pdf)).
- **Effect on prior art:** sokol_gfx documents that storage buffers are "*NOT* supported on … macOS+GL (because storage buffers require GL 4.3, while macOS only goes up to GL 4.1)" ([sokol_gfx.h L1741-L1744](https://github.com/floooh/sokol/blob/f3a42297dfe64edb390deaa5883728a558499eb5/sokol_gfx.h#L1741-L1744)). bgfx's OpenGL backend now requires "OpenGL 4.3+" ([bgfx README](https://github.com/bkaradzic/bgfx/blob/d7d83dc16188e7348fcc8a48b05c27427c72ab2b/README.md#L43-L53)).

### Apple's deprecation status

- **macOS Mojave 10.14 release notes:** "The APIs in the OpenGL and OpenCL frameworks are deprecated and remain present for compatibility purposes. Transition to Metal" ([Apple](https://developer.apple.com/documentation/macos-release-notes/macos-mojave-10_14-release-notes)).
- **`NSOpenGLView`:** deprecated in macOS 10.14 with the note "Please use MTKView instead" ([Apple](https://developer.apple.com/documentation/appkit/nsopenglview)).

### OpenGL drivers on Windows and Linux

| Platform | State | Source |
|---|---|---|
| Windows (Microsoft's own GL) | "The version of OpenGL described in this section is 1.1". Newer versions come from GPU vendors' ICDs | [MS Learn: OpenGL](https://learn.microsoft.com/en-us/windows/win32/opengl/opengl) |
| Windows on Arm | Microsoft's "OpenCL and OpenGL Compatibility Pack" is a Mesa GL ICD mapped onto D3D12. When announced it supported "OpenGL version 3.3 and earlier" | [DirectX blog, 2020-11-17](https://devblogs.microsoft.com/directx/announcing-the-opencl-and-opengl-compatibility-pack-for-windows-10-on-arm/) |
| Mesa (Linux; also the D3D12 mapping driver) | GL 4.6 done for radeonsi, iris, crocus (gen7+), r600 (evergreen+), zink (GL on Vulkan), d3d12, asahi, virgl and llvmpipe. nvc0 (nouveau GL) stops at 4.5. `KHR_debug` and DSA are "DONE (all drivers)" | [Mesa docs/features.txt](https://gitlab.freedesktop.org/mesa/mesa/-/blob/6384fd6037871b44784d78bf63369bdf05708965/docs/features.txt#L143-L231) |
| Vendor drivers (NVIDIA/AMD/Intel on Windows; NVIDIA proprietary on Linux) | Not verified: the Khronos conformance list was unreachable from this environment | see Open questions |

---

## 4. Shaders: authoring and cross-compilation

### What each backend consumes

| Backend | Accepts | Source |
|---|---|---|
| Vulkan (incl. MoltenVK and KosmicKrisp) | SPIR-V; no signing required | [Slang targets doc](https://github.com/shader-slang/slang/blob/ae4e57957a0adad3061b069b0e0eec8217779338/docs/user-guide/09-targets.md#L230-L240) |
| D3D12 | DXIL (SM 6.x, must be signed by the validator unless developer mode is on), or DXBC SM 5.1 | [Slang targets L149-L157](https://github.com/shader-slang/slang/blob/ae4e57957a0adad3061b069b0e0eec8217779338/docs/user-guide/09-targets.md#L149-L157); [SDL PR #11456](https://github.com/libsdl-org/SDL/pull/11456) |
| D3D11 | DXBC (SM ≤ 5.0). "The only supported way to generate DXBC is by compiling HLSL using the fxc compiler" | [Slang targets L101-L109](https://github.com/shader-slang/slang/blob/ae4e57957a0adad3061b069b0e0eec8217779338/docs/user-guide/09-targets.md#L101-L109) |
| Metal | MSL source compiled at runtime (`makeLibrary(source:options:)`, macOS 10.11+), or a precompiled `.metallib` built with the `metal`/`metal-ar` tools. "The Metal command-line tools for Windows use the same options" | [Apple: makeLibrary](https://developer.apple.com/documentation/metal/mtldevice/makelibrary(source:options:)); [Apple: precompiling](https://developer.apple.com/documentation/metal/building-a-shader-library-by-precompiling-source-files) |
| OpenGL 4.1–4.5 | GLSL source. GL 4.6 adds SPIR-V via `glSpecializeShader` | [gl.xml](https://github.com/KhronosGroup/OpenGL-Registry/blob/6af574a14089ccfee87efe230ebcdd8742859813/xml/gl.xml#L33787) |

### Tools

| Tool (version, date) | Input → output | Offline / runtime | C-callable API | License | Notes |
|---|---|---|---|---|---|
| **DXC** (v1.9.2609, 2026-09-29) | HLSL → DXIL and SPIR-V. Can also emit metallib (`-metal`) if built against Metal Shader Converter ([README](https://github.com/microsoft/DirectXShaderCompiler/blob/2630a1873de4f41354b4449f6861957a838265f4/README.md)) | `dxc` CLI; `dxcompiler.[dll\|so]` library ([README](https://github.com/microsoft/DirectXShaderCompiler/blob/2630a1873de4f41354b4449f6861957a838265f4/README.md)) | No. `dxcapi.h` declares C++ structs that inherit from `IUnknown`; only `DxcCreateInstance` is `extern "C"` ([dxcapi.h L74-L83, L151](https://github.com/microsoft/DirectXShaderCompiler/blob/2630a1873de4f41354b4449f6861957a838265f4/include/dxc/dxcapi.h#L74-L83)). SDL_shadercross, written in C, hand-declares the vtables ([SDL_shadercross.c L110-L230](https://github.com/libsdl-org/SDL_shadercross/blob/1ff05bec573988a98ef9e0260b4da44f512b8367/src/SDL_shadercross.c#L110-L230)) | University of Illinois/NCSA ([LICENSE.TXT](https://github.com/microsoft/DirectXShaderCompiler/blob/2630a1873de4f41354b4449f6861957a838265f4/LICENSE.TXT)) | Release assets: Windows zip and Linux tarball ([release](https://github.com/microsoft/DirectXShaderCompiler/releases/tag/v1.9.2609)). Also in the Windows SDK (DXIL only) and the Vulkan SDK. SPIR-V codegen is described as a "community contribution" |
| **FXC / `D3DCompile`** | HLSL → DXBC (SM 2–5.x) | Runtime via `d3dcompiler_47.dll` | Yes, a plain C function returning `ID3DBlob` ([MS](https://learn.microsoft.com/en-us/windows/win32/api/d3dcompiler/nf-d3dcompiler-d3dcompile)) | Windows component | Windows only. SDL_shadercross says `d3dcompiler_47` "is shipped with Windows" ([README.txt](https://github.com/libsdl-org/SDL_shadercross/blob/1ff05bec573988a98ef9e0260b4da44f512b8367/README.txt)) |
| **glslang** | GLSL/ESSL → SPIR-V. Its HLSL front end was "deprecated as of April 2026" and will be removed at the next major version, with at least 18 months' notice ([README](https://github.com/KhronosGroup/glslang/blob/83f75db6ada23801a65eed1cde4763a0a5bfbda6/README.md#L5-L33)) | CLI + library (now also as a shared library) | Yes: `glslang_c_interface.h` ([README L417-L425](https://github.com/KhronosGroup/glslang/blob/83f75db6ada23801a65eed1cde4763a0a5bfbda6/README.md#L419-L425)) | Mix of BSD-3, BSD-2, MIT and Apache-2.0 ([LICENSE.txt](https://github.com/KhronosGroup/glslang/blob/83f75db6ada23801a65eed1cde4763a0a5bfbda6/LICENSE.txt)) | |
| **shaderc** | GLSL → SPIR-V (HLSL "deprecated" because glslang's is) | `glslc` + `libshaderc` | Yes: `shaderc.h` is `extern "C"` | Apache-2.0 | Wraps glslang and SPIRV-Tools ([README](https://github.com/google/shaderc/blob/4f0683ab2419ecbb9e18c9edfd4ffd70c2ba8cc6/README.md#L1-L20)) |
| **SPIRV-Cross** | SPIR-V → GLSL/ESSL, MSL, HLSL, JSON reflection; reflection API | CLI + library. MoltenVK uses it at runtime | Yes: a C89 wrapper, "fully stable, both API and ABI-wise", and the only interface supported for shared builds ([README L132-L140](https://github.com/KhronosGroup/SPIRV-Cross/blob/57e4efcc84112abcd426dc5ec248b5339bee04f2/README.md#L132-L140)) | Apache-2.0; C API files "Apache-2.0 OR MIT" ([spirv_cross_c.h](https://github.com/KhronosGroup/SPIRV-Cross/blob/57e4efcc84112abcd426dc5ec248b5339bee04f2/spirv_cross_c.h)) | |
| **Slang** (v2026.21, 2026-11-04) | Slang (most HLSL compiles as-is; GLSL compatibility module) → SPIR-V (direct), DXIL/DXBC (via downstream `dxc`/`fxc`), HLSL, GLSL, MSL (experimental), WGSL (experimental), CUDA, CPU ([README L103-L121](https://github.com/shader-slang/slang/blob/ae4e57957a0adad3061b069b0e0eec8217779338/README.md#L103-L121); [compiling doc](https://github.com/shader-slang/slang/blob/ae4e57957a0adad3061b069b0e0eec8217779338/docs/user-guide/08-compiling.md#L258-L320)) | `slangc` + library; modules can be compiled to IR offline and linked at runtime | No: a COM-style C++ API (`slang::IGlobalSession`, [slang.h L6102](https://github.com/shader-slang/slang/blob/ae4e57957a0adad3061b069b0e0eec8217779338/include/slang.h#L6102)). The older `sp*` API is deprecated and uses C++ default arguments ([slang-deprecated.h](https://github.com/shader-slang/slang/blob/ae4e57957a0adad3061b069b0e0eec8217779338/include/slang-deprecated.h)) | Apache-2.0 WITH LLVM-exception ([LICENSE](https://github.com/shader-slang/slang/blob/ae4e57957a0adad3061b069b0e0eec8217779338/LICENSE)) | Khronos-hosted since 2024-11-21 ([Khronos](https://www.khronos.org/news/press/khronos-group-launches-slang-initiative-hosting-open-source-compiler-contributed-by-nvidia)). In the Vulkan SDK since 1.3.296.0. On Metal it supports vertex, fragment, compute, task and mesh shaders |
| **SDL_shadercross** | SPIR-V or HLSL → DXBC, DXIL, SPIR-V, MSL, HLSL | CLI + runtime library ("offline ahead-of-time compilation is preferred", [SDL_gpu.h L137](https://github.com/libsdl-org/SDL/blob/f3be7b901e5c5ed5b4860bd89c99fc386be28b9d/include/SDL3/SDL_gpu.h#L137)) | Yes, C ([SDL_shadercross.h](https://github.com/libsdl-org/SDL_shadercross/blob/1ff05bec573988a98ef9e0260b4da44f512b8367/include/SDL3_shadercross/SDL_shadercross.h)) | zlib | Depends on SPIRV-Cross, DXC, and `d3dcompiler_47` or vkd3d-utils; its API uses SDL3 types ([README.txt](https://github.com/libsdl-org/SDL_shadercross/blob/1ff05bec573988a98ef9e0260b4da44f512b8367/README.txt)) |
| **vkd3d-shader** | HLSL, DXIL, DXBC, SM 1–3 → SPIR-V, Direct3D formats; GLSL/MSL are work in progress | Library | Yes: `vkd3d_shader_compile()` | LGPL-2.1+ | Produces DXBC on platforms other than Windows ([README](https://gitlab.winehq.org/wine/vkd3d/-/blob/master/README); [COPYING](https://gitlab.winehq.org/wine/vkd3d/-/blob/master/COPYING)) |
| **Metal Shader Converter 4.0 beta** | DXIL (SM 6.0–6.6, incl. ray tracing and mesh/amplification) → metallib | CLI + `libmetalirconverter`; needs macOS 13 + Xcode 15, or Windows 10 + VS2019. The output needs Argument Buffers Tier 2 and macOS 14 / iOS 17 | Yes: "a C interface" | Apple binary (terms not checked) | [Apple](https://developer.apple.com/metal/shader-converter/) |

### Where the toolchains are heading

- **DirectX will take SPIR-V.** Microsoft (2024-09-19): starting with Shader Model 7, "DirectX 12 will accept shaders compiled to SPIR-V". SPIR-V replaces DXIL as the interchange format over "several years", and HLSL support is being built into upstream Clang ([DirectX blog](https://devblogs.microsoft.com/directx/directx-adopting-spir-v/)).
- **glslang and shaderc are dropping HLSL** (see table above). That leaves DXC, Slang and vkd3d-shader as HLSL front ends.
- **Register-to-descriptor mapping:** Khronos documents how HLSL registers map to Vulkan descriptor types (`b` = uniform buffer, `t` = sampled image, `u` = storage buffer/image, `s` = sampler) and how DXC is used offline and at runtime ([Vulkan-Guide hlsl.adoc L163-L173](https://github.com/KhronosGroup/Vulkan-Guide/blob/b02c0f42e821b6f9fb99acd850c5f3a9f8936ca2/chapters/hlsl.adoc#L163-L173)).

---

## 5. Prior art: multi-backend libraries and engines

| | sokol_gfx | bgfx | SDL3 GPU (3.4.20) | The Forge (1.64) | Kohi |
|---|---|---|---|---|---|
| Language / API | C, single header; ~155 public functions ([sokol_gfx.h](https://github.com/floooh/sokol/blob/f3a42297dfe64edb390deaa5883728a558499eb5/sokol_gfx.h)) | C++ library with a C99 API (~226 functions) ([c99/bgfx.h](https://github.com/bkaradzic/bgfx/blob/d7d83dc16188e7348fcc8a48b05c27427c72ab2b/include/bgfx/c99/bgfx.h)) | C, ~97 functions ([SDL_gpu.h](https://github.com/libsdl-org/SDL/blob/f3be7b901e5c5ed5b4860bd89c99fc386be28b9d/include/SDL3/SDL_gpu.h)) | Runtimes rewritten in C99 (release 1.62, 2025-02-27) ([README](https://github.com/ConfettiFX/The-Forge/blob/cd5046893faba2dc7869243873bf01f02a6f0df9/README.md#L174-L178)); the resource-loader header still uses C++ overloads (`addResource(BufferLoadDesc*)`, `addResource(TextureLoadDesc*)`) ([IResourceLoader.h](https://github.com/ConfettiFX/The-Forge/blob/cd5046893faba2dc7869243873bf01f02a6f0df9/Common_3/Resources/ResourceLoader/Interfaces/IResourceLoader.h#L538-L540)) | C99 ([readme](https://github.com/travisvroman/kohi/blob/05a5193d5ed4890703cfcbbeb5c5e947feefecf6/readme.md)) |
| Backends | GL core, GLES3/WebGL2, D3D11, Metal, WebGPU, Vulkan ("experimental") ([sokol_gfx.h L19-L26](https://github.com/floooh/sokol/blob/f3a42297dfe64edb390deaa5883728a558499eb5/sokol_gfx.h#L19-L26); [README L69](https://github.com/floooh/sokol/blob/f3a42297dfe64edb390deaa5883728a558499eb5/README.md#L69)) | D3D11, D3D12, Metal, GL 4.3+, GLES 3.0+, Vulkan, WebGL 2, WebGPU (Dawn), consoles ([README](https://github.com/bkaradzic/bgfx/blob/d7d83dc16188e7348fcc8a48b05c27427c72ab2b/README.md#L43-L53)) | Vulkan, D3D12, Metal ([SDL_gpu.h L217-L271](https://github.com/libsdl-org/SDL/blob/f3be7b901e5c5ed5b4860bd89c99fc386be28b9d/include/SDL3/SDL_gpu.h#L217-L271)); D3D11 removed ([PR #11456](https://github.com/libsdl-org/SDL/pull/11456)) | D3D12 on Windows; Vulkan on Steam Deck, Android, Quest and Switch; Metal on Apple; consoles. D3D11 and official Vulkan-on-Windows removed ([README](https://github.com/ConfettiFX/The-Forge/blob/cd5046893faba2dc7869243873bf01f02a6f0df9/README.md#L1-L20)) | Vulkan only; Direct3D, Metal, OpenGL and headless plugins are unchecked TODOs ([TODO.md L200-L211](https://github.com/travisvroman/kohi/blob/05a5193d5ed4890703cfcbbeb5c5e947feefecf6/TODO.md#L200-L211)) |
| How the backend is chosen | Compile-time define | Runtime renderer type (`BGFX_RENDERER_TYPE_*`, [L83-L95](https://github.com/bkaradzic/bgfx/blob/d7d83dc16188e7348fcc8a48b05c27427c72ab2b/include/bgfx/c99/bgfx.h#L83-L95)) | Runtime: SDL picks a backend from the shader formats the app says it can provide and the backends available ([SDL_gpu.h L119-L130](https://github.com/libsdl-org/SDL/blob/f3be7b901e5c5ed5b4860bd89c99fc386be28b9d/include/SDL3/SDL_gpu.h#L119-L130)) | Compile time (`#if defined(DIRECT3D12)`, … in [IGraphics.h](https://github.com/ConfettiFX/The-Forge/blob/cd5046893faba2dc7869243873bf01f02a6f0df9/Common_3/Graphics/Interfaces/IGraphics.h#L59-L95)); runtime API switching was removed ([README L263](https://github.com/ConfettiFX/The-Forge/blob/cd5046893faba2dc7869243873bf01f02a6f0df9/README.md#L263)) | **Runtime plugin**: `kohi.plugin.renderer.vulkan` is a .dll/.so/.dylib whose `kohi_plugin_renderer_vulkan_create()` fills a `renderer_backend_interface` struct of function pointers ([plugin main](https://github.com/travisvroman/kohi/blob/05a5193d5ed4890703cfcbbeb5c5e947feefecf6/kohi.plugin.renderer.vulkan/src/vulkan_renderer_plugin_main.c)) |
| Objects exposed | 6 handle types: `sg_buffer`, `sg_image`, `sg_sampler`, `sg_shader`, `sg_pipeline`, `sg_view` ([L2422-L2427](https://github.com/floooh/sokol/blob/f3a42297dfe64edb390deaa5883728a558499eb5/sokol_gfx.h#L2422-L2427)) | Handles: static and dynamic vertex/index buffers, texture, frame buffer, shader, program, uniform, indirect buffer, occlusion query, vertex layout ([L543-L567](https://github.com/bkaradzic/bgfx/blob/d7d83dc16188e7348fcc8a48b05c27427c72ab2b/include/bgfx/c99/bgfx.h#L543-L567)) | Device, buffer, transfer buffer, texture, sampler, shader, graphics pipeline, compute pipeline, command buffer, render/compute/copy pass, fence ([L414-L601](https://github.com/libsdl-org/SDL/blob/f3be7b901e5c5ed5b4860bd89c99fc386be28b9d/include/SDL3/SDL_gpu.h#L414-L601)) | Renderer, queue, command pool, cmd, buffer, texture, render target, sampler, shader, pipeline, pipeline cache, descriptor set, fence, semaphore, swapchain, query pool, resource heap, work graph ([IGraphics.h L2819-L2961](https://github.com/ConfettiFX/The-Forge/blob/cd5046893faba2dc7869243873bf01f02a6f0df9/Common_3/Graphics/Interfaces/IGraphics.h#L2819-L2961)) | 90 function pointers covering frame begin/submit/present, dynamic-state setters, `begin_rendering`/`end_rendering`, textures, shaders with "binding sets", samplers, "renderbuffers" and a GPU profiler ([renderer_types.h L290-L899](https://github.com/travisvroman/kohi/blob/05a5193d5ed4890703cfcbbeb5c5e947feefecf6/kohi.runtime/src/renderer/renderer_types.h#L290-L899)) |
| Command buffers | Not exposed: `sg_begin_pass` … `sg_end_pass`, `sg_commit` | Not exposed. Encoders feed *views*; draws get 64-bit sort keys and are radix-sorted on a render thread ([internals.rst](https://github.com/bkaradzic/bgfx/blob/d7d83dc16188e7348fcc8a48b05c27427c72ab2b/docs/internals.rst)) | Exposed (`SDL_AcquireGPUCommandBuffer`, submit, fences) | Exposed (`initCmd`, `beginCmd`, `queueSubmit`) | Not exposed (`frame_commands_begin`/`end`) |
| Render passes | Explicit pass begin/end | Declarative views (target, clear, rect) | Explicit render/compute/copy passes | `cmdBindRenderTargets` | `begin_rendering` / `end_rendering` |
| Binding model | `sg_bindings` holds vertex/index buffers, views and samplers; plus `sg_apply_uniforms` ([L3601-L3610](https://github.com/floooh/sokol/blob/f3a42297dfe64edb390deaa5883728a558499eb5/sokol_gfx.h#L3601-L3610)) | `bgfx_set_texture(stage, sampler_uniform, texture)`, uniforms, 64-bit state flags (`bgfx_set_state`) | Per-stage slots with a **fixed shader layout convention**. SPIR-V: set 0 = vertex resources, 1 = vertex uniforms, 2 = fragment resources, 3 = fragment uniforms. HLSL: `space0`–`space3`. Four uniform slots per stage ([L2915-L2975](https://github.com/libsdl-org/SDL/blob/f3be7b901e5c5ed5b4860bd89c99fc386be28b9d/include/SDL3/SDL_gpu.h#L2915-L2975)) | Descriptor sets (`addDescriptorSet`, `updateDescriptorSet`, `cmdBindDescriptorSet`) with graphics/compute root signatures | Binding-set instances per shader (`shader_acquire_binding_set_instance`, `shader_apply_binding_set`) |
| Barriers / sync | Hidden | Hidden | Hidden. Writes take a "cycle" flag so you "don't have to worry about complex state tracking and synchronization" ([L326-L332](https://github.com/libsdl-org/SDL/blob/f3be7b901e5c5ed5b4860bd89c99fc386be28b9d/include/SDL3/SDL_gpu.h#L326-L332)) | Explicit `cmdResourceBarrier` (buffer, texture, render-target barriers), fences and semaphores | Hidden (`texture_prepare_for_sampling`, `colour_texture_prepare_for_present`) |
| Shaders | sokol-shdc: annotated Vulkan-style GLSL → GLSL 300es/310es/410/430, HLSL4/5, MSL, WGSL, via glslang, SPIRV-Tools, SPIRV-Cross and Tint ([sokol-shdc.md](https://github.com/floooh/sokol-tools/blob/8ddcfe3bc18545b0e660848d941439d9166a140b/docs/sokol-shdc.md)) | Own `shaderc`: a GLSL-like language run through a C preprocessor ([tools.rst](https://github.com/bkaradzic/bgfx/blob/d7d83dc16188e7348fcc8a48b05c27427c72ab2b/docs/tools.rst)) | The app supplies the backend's format (SPIR-V, DXBC, DXIL, MSL, metallib); SDL_shadercross is offered | FSL, "a superset of HLSL", translated per platform ([README](https://github.com/ConfettiFX/The-Forge/blob/cd5046893faba2dc7869243873bf01f02a6f0df9/README.md)) | GLSL compiled at runtime with shaderc inside the Vulkan plugin ([vulkan_backend.c L10-L14, L349](https://github.com/travisvroman/kohi/blob/05a5193d5ed4890703cfcbbeb5c5e947feefecf6/kohi.plugin.renderer.vulkan/src/vulkan_backend.c#L349)) |
| Advanced features | Compute and storage buffers (not on macOS GL) | Compute, indirect | "no immediate plans" for ray tracing or mesh shaders; "If you need cutting-edge features … this API is probably not for you" ([L150, L185](https://github.com/libsdl-org/SDL/blob/f3be7b901e5c5ed5b4860bd89c99fc386be28b9d/include/SDL3/SDL_gpu.h#L150)) | Ray tracing, work graphs, `cmdExecuteIndirect`, pipeline statistics | Compute is a TODO |
| Self-description | "simple 3D API wrapper" | "declarative API" ([internals.rst](https://github.com/bkaradzic/bgfx/blob/d7d83dc16188e7348fcc8a48b05c27427c72ab2b/docs/internals.rst)) | "a relatively thin layer over the underlying graphics API" ([L208](https://github.com/libsdl-org/SDL/blob/f3be7b901e5c5ed5b4860bd89c99fc386be28b9d/include/SDL3/SDL_gpu.h#L208)) | — | "The frontend only interacts via this structure and has no knowledge of the way things actually work on the backend" ([renderer_types.h L290-L297](https://github.com/travisvroman/kohi/blob/05a5193d5ed4890703cfcbbeb5c5e947feefecf6/kohi.runtime/src/renderer/renderer_types.h#L290-L297)) |
| License | zlib | BSD-2-Clause | zlib | Apache-2.0 (consoles commercial) | Apache-2.0 |

A related reference is the standardized C header **`webgpu.h`** (BSD-3). It is implemented by Dawn and Emdawnwebgpu; wgpu-native "does not yet implement the stable version" ([webgpu-headers README](https://github.com/webgpu-native/webgpu-headers/blob/a1dc77409bb2b8bd80540cbf4b8db219e123e909/README.md)). Its objects: instance, adapter, device, queue, buffer, texture, texture view, sampler, bind group, bind group layout, pipeline layout, render/compute pipeline, shader module, command encoder, render/compute pass encoder, command buffer, render bundle, query set, surface ([webgpu.h](https://github.com/webgpu-native/webgpu-headers/blob/a1dc77409bb2b8bd80540cbf4b8db219e123e909/webgpu.h)).

---

## Implications for Ember

These are options and trade-offs only; none is a decision.

### "Is OpenGL a v1 backend?"

- **Option A: no OpenGL.**
  - Matches SDL3 GPU and The Forge, neither of which has a GL backend (§5).
  - macOS is covered by Metal and MoltenVK/KosmicKrisp.
  - Linux and Windows are covered by Vulkan; Windows also by D3D12.
  - Cost: no fallback for machines without Vulkan or D3D12.
- **Option B: GL 4.1 core as the floor, so it can run on macOS.**
  - Compute, SSBOs, image load/store, `KHR_debug` output, DSA and SPIR-V are all unavailable (§3).
  - The backend interface would need a lower capability tier, or feature flags. sokol does this by disabling storage buffers on macOS GL.
  - The API has been deprecated on macOS since 2018.
  - RenderDoc does not run on macOS, and Apple's own GL tables stop at 2017.
- **Option C: GL 4.3+ or 4.6 on Windows and Linux only.**
  - Mesa provides GL 4.6 on most Linux drivers.
  - Windows depends on vendor ICDs (status not verified here). Windows on Arm's mapping layer was 3.3 when announced.
  - bgfx sets its GL floor at 4.3.
- **Cost for any GL option:**
  - GL's implicit-state, implicit-sync model needs a different backend shape from Vulkan, D3D12 and Metal 4 (§1 mapping table).
  - Shaders need GLSL output: via SPIRV-Cross, or SPIR-V directly on 4.6.

### "Direct3D 11 or 12?"

- **D3D12 matches Vulkan concept for concept** (§1 mapping table). An explicit backend interface built for Vulkan carries over. D3D12 also reaches SM 6.x, DXR, mesh shaders and work graphs.
  - Costs: explicit synchronization, residency, barriers and descriptor heaps; FL 11_0 minimum; Windows 10+.
  - DLL-specific: the Agility SDK normally needs exports from the host `.exe`. A backend DLL would need the host to export them, or would use `CreateDeviceFactory` (§1).
  - Shaders: DXIL from DXC (C++ API; official binaries for Windows and Linux), or DXBC SM 5.1 from FXC (SDL's route).
  - Tooling: PIX, GBV, DRED and RenderDoc.
- **D3D11 is simpler: implicit synchronization and slot binding.**
  - Its model is closer to classic Metal and GL than to Vulkan, so it pulls the backend interface toward a thicker shape.
  - Shaders are capped at SM 5.0 DXBC, which only FXC (Windows-only) or vkd3d-shader (LGPL) produce.
  - No SM6, ray tracing or mesh shaders.
  - Tooling: RenderDoc and the debug layer; no PIX.
  - SDL3 GPU and The Forge both dropped it (2024–25); bgfx and sokol keep it.
- **Both:** two Windows backends to maintain. bgfx is the only surveyed library that ships both.
- **Neither (Vulkan on Windows):** possible, since Vulkan is already planned. The Forge reports dropping Vulkan on Windows for QA and reliability reasons. That is their experience, not a measured fact.

### "Shading language and shader compilation"

- **HLSL + DXC as the front end.**
  - DXC emits DXIL for D3D12 and SPIR-V for Vulkan and the macOS layers.
  - Metal: SPIRV-Cross → MSL (C API), or DXIL → Metal Shader Converter → metallib (C API, Apple binary).
  - D3D11 would add an FXC step (or SPIRV-Cross → HLSL → FXC, which is SDL_shadercross's round trip).
  - DXC's API is C++/COM, so a C engine needs a C++ shim, hand-written vtables (as SDL_shadercross does), or offline-only use of the CLI.
  - Microsoft's SM 7 SPIR-V plan points the same way long-term.
- **GLSL + glslang or shaderc (C APIs) → SPIR-V, then SPIRV-Cross for MSL, HLSL and GLSL.**
  - Everything can be called from C. sokol-shdc and Kohi work this way.
  - D3D12 then consumes HLSL that SPIRV-Cross generated: compiled by DXC to DXIL, or by FXC to DXBC SM 5.1.
  - glslang's HLSL input is deprecated, so this route means writing GLSL.
- **Slang.**
  - One language to every target. Khronos-hosted, permissive license.
  - The API is C++/COM, and DXIL/DXBC output still needs DXC/FXC downstream.
  - Its Metal target is labelled experimental.
- **Offline vs runtime.**
  - Every compiler above runs both ways.
  - MoltenVK converts SPIR-V to MSL at runtime regardless (cacheable through the pipeline cache).
  - Metal can compile MSL at runtime, or load metallibs built offline on macOS or Windows.
  - Runtime compilation means shipping compiler libraries in the engine or in each backend (Kohi ships shaderc in its Vulkan plugin).
- **Resource-binding convention.** Pick one that maps to Vulkan sets, D3D register spaces and Metal argument tables. SDL3 GPU's fixed set/space convention is a worked example.

### "The renderer's interface to its backends"

- **Thin, explicit interface** (The Forge, `webgpu.h`): command buffers, barriers, descriptor sets/bind groups, pipelines and passes.
  - Maps nearly 1:1 onto Vulkan, D3D12 and Metal 4.
  - D3D11, GL and classic Metal backends would have to emulate barriers and descriptor sets.
  - More engine-side responsibility.
- **Middle interface** (SDL3 GPU): explicit command buffers and passes, but no barriers. Hazards are handled by "cycling", and the binding layout is fixed.
  - Smaller surface. The backend does hazard tracking.
  - SDL deliberately leaves out ray tracing and mesh shaders.
- **Thick or declarative interface** (bgfx views and sort keys; sokol passes with no command buffers; Kohi's ~90 function pointers around shaders, binding sets and renderbuffers).
  - Easiest to put an implicit API behind.
  - Backends get larger and duplicate more logic.
  - Kohi is the closest structural precedent: C99, and a renderer backend as a shared library that fills a function-pointer struct at load time.
- **What the facts constrain, whichever shape is chosen:**
  - The boundary between modules is a C ABI (Kohi uses a function-pointer table).
  - Shader formats differ per backend (§4), so the interface has to say which bytecode each backend accepts (SDL negotiates this at device creation).
  - Feature tiers differ: MoltenVK has no geometry or mesh shaders; GL 4.1 has no compute; native Metal and D3D12 have mesh shaders and ray tracing.
  - The D3D12 Agility SDK has a per-process loading quirk (§1).

---

## Open questions (not verified)

- **Windows vendor OpenGL drivers:** current GL 4.6 status for NVIDIA, AMD and Intel on Windows, and for NVIDIA's proprietary Linux driver. The Khronos conformant-products page refused connections from this environment.
- **What OpenGL exposes on Apple-silicon Macs today:** version and extension strings. Apple's support page lists only Intel-era models, and its capability tables are a 2017 snapshot that stops at macOS 10.9.
- **How KosmicKrisp implements geometry shaders, and how it performs:** whether they are emulated, what the cost is, and how fast it is in real workloads compared with MoltenVK. LunarG only compares it with its own earlier versions ("tripled").
- **Which driver the macOS Vulkan SDK picks by default** when both MoltenVK and KosmicKrisp are installed.
- **Other Vulkan-on-Metal layers:** none beyond MoltenVK and KosmicKrisp was surveyed.
- **Apple's own statement that macOS 26 is the last release for Intel Macs.** Only LunarG's page ("discontinued … as of macOS 27") was found.
- **`ID3D12SDKConfiguration1::CreateDeviceFactory`:** the minimum Windows/D3D12 runtime build that supports it, and whether it works on retail machines without developer mode. The spec says it does not require developer mode.
- **Metal Shader Converter license terms** for redistributing `libmetalirconverter` with an engine.
- **Whether `d3d11.h` offers the same C interface as `d3d12.h`.** Only `d3d12.h` was inspected.
- **DXC binaries for macOS:** the official GitHub release has Windows and Linux assets only. The README says DXC is in the Vulkan SDK; its presence in the macOS SDK was not checked.
- **D3D11-only hardware share:** the SDL PR's "very small and shrinking" is an assertion, not data. No primary market-share data was gathered.
