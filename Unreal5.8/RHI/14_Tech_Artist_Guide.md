# Tech Artist Guide — UE 5.8 RHI

## Scope

What a tech artist needs to know about the RHI to make content decisions that survive profiling. Everything here is grounded in the chapters 01-13.

## What actually costs

| Cost | Source | What changes it |
|---|---|---|
| Draw calls | `STAT_MeshDrawCalls` (`RenderCore/Public/RenderCore.h:38`) | instancing merge, `StateBucketId` grouping |
| State changes | `FMeshDrawCommandStateCache` (`Renderer/Public/MeshPassProcessor.h:905-960`) | sorting by `SortKey` then `StateBucketId` (`L1836-1839`) |
| PSO creation | `GetAndOrCreateGraphicsPipelineState` (`RHI/Public/PipelineStateCache.h:189`) | material/shader permutation count, render-target format, load/store actions |
| Transitions | `FRHITransitionInfo` (`RHI/Public/RHITransition.h:118`) | access states, RDG merge eligibility |
| Per-instance data | GPUScene instance slots (`Renderer/Private/GPUScene.h:244`, `251`) | instance count, dynamic primitive data |
| GPU instance culling | `InstanceCullingContext` dispatches | `bForceInstanceCulling`, occlusion queries |

A material that changes render-target format or load action creates a new PSO (`RHIResources.h:4748`, `4828`). A material that binds a UAV in a vertex shader needs `bSupportsVertexShaderUAVs` (`DataDrivenShaderPlatformInfo.h:102`).

## Instancing guidance

Content level (`Engine/Source/Runtime/Engine/Public/MeshBatch.h:231-333`):

- `NumInstances` (`L276`) becomes `FMeshDrawCommand.NumInstances` (`Renderer/Public/MeshPassProcessor.h:1302`), which becomes `InstanceCount` in the indirect args (`InstanceCullingContext.cpp:278`).
- `InstanceRuns` (`L258`) + `bIsInstanceRuns` (`L287`) let one batch cover multiple disjoint instance ranges; `GetNumPrimitives()` (`L307`) accounts for them.
- `InstancedLODIndex:4` and `InstancedLODRange:4` (`L284-285`) — 4 bits each, so a batch can only describe a LOD and a LOD range within 16 LODs.
- `MinScreenSize`/`MaxScreenSize` (`L281-282`) are packed as fixed-point 3.10 into `FMeshDrawCommandCullingPayload` (`MeshPassProcessor.h:1672`, `LodIndex:4`, `MinScreenSize:13`, `MaxScreenSize:13`, `PackScreenSize` `L1688`). Screen sizes outside the packable range lose precision in GPU culling.
- `bForceInstanceCulling` (`L288`), `bPreserveInstanceOrder` (`L289`), `bFetchInstanceCountFromScene` (`L290`) change whether culling runs and whether order is kept — order-preserving instancing blocks some merges.

LOD selection for an ISM group uses the instance sphere radius: `ComputeLODForMeshes(..., InstanceSphereRadius, ...)` (`Engine/Private/SceneManagement.cpp:1168`) computes a LOD *range* from the nearest and farthest instance (`L1187-1191`). A group with a large radius spans multiple LODs, which means multiple batches and fewer merged instances. Tight instance grouping is therefore both a memory and a draw-call decision.

Merge rule (`Renderer/Private/MeshDrawCommands.cpp:506-569`): commands with the same `StateBucketId` are merged by incrementing `NumInstances` (`L514-521`, capped by `MaxInstances` `L508`, `L520`). A new command is only emitted when the bucket has more than one draw and `PrimitiveIdStreamIndex >= 0` and `NumInstances == 1` (`L534-540`). Indirect draws fall back to direct when `NumPrimitives > 0 && PrimitiveIdStreamIndex < 0` (`L930-936`).

Practical consequence: two instances of the same mesh with the same material and same LOD are one draw. Same material but different LOD, or a vertex factory without a primitive-id stream, is not.

## Shader and permutation guidance

- Permutations are per shader platform, not per feature level (`RHIShaderPlatform.h:10`, `RHIGlobals.h:1016`). Vertex factory type and source hash are part of the cache key (`RenderCore/Public/VertexFactory.h:593`, `605`).
- `ShouldCompilePermutation` (`VertexFactory.h:566`) is the lever for permutation explosion; a VF that compiles for every platform pays on every platform.
- PSO file cache is off in editor (`r.ShaderPipelineCache.Enabled` = `!WITH_EDITOR`, `PipelineFileCache.cpp:112`), on in game (`L119` logging, `L159` user cache in Shipping). Editor hitches are expected; shipping misses come from combinations never logged.
- `r.pso.evictiontime` (60, `PipelineStateCache.cpp:188`) and `r.PSO.RuntimeCreationHitchThreshold` (20 ms, `L195`) — a hitch above the threshold is logged with the breadcrumb that caused it (`PipelineStateCache.cpp:290-297`, `315-336`).

## Bindless, virtual textures, streaming

- Descriptor ranges require `GRHIGlobals.bSupportsDescriptorRange` and `DescriptorTypeGroups` (`RHIGlobals.h:718`), surfaced to shaders as `PLATFORM_SUPPORTS_DESCRIPTOR_RANGE` (`RHIDescriptorRange.h:15`, `26`); per-platform support via `RHIGetStaticShaderBindingLayoutSupport` (`DataDrivenShaderPlatformInfo.h:1299`).
- `FRHITextureReference` (`RHI/Public/RHITextureReference.h:7`) is only an *indirection*: `SetReferencedTexture` (`L48`) is called from `FDynamicRHI::RHIUpdateTextureReference` (`RHI/Public/DynamicRHI.h:434`) and swaps which whole `FRHITexture` the reference points at. It implements no tile residency, so a cached pointer to contents is invalid across frames. Residency lives in the VT system: `FVirtualTexturePhysicalSpace` holds the physical texture plus the page pool that tracks residency (`Renderer/Private/VT/VirtualTexturePhysicalSpace.h:88-89`), and residency is a mapping of virtual to physical *page addresses*, not tile pixels — `FTexturePage` packs a 24-bit virtual address (Morton order relative to mip 0) plus an 8-bit log size (`Renderer/Private/VT/TexturePageMap.h:25-27`), which is what the header's "4k x 4k virtual pages / 256 x 256 physical pages" comment means (`L14-15`): address-space capacity, not a fixed virtual-to-physical ratio. `FPhysicalSpaceIDAndAddress` packs a 16-bit physical address with a 16-bit space ID (`L40-41`), and mappings must go through `FTexturePagePool` (`L57`). `IAllocatedVirtualTexture` exposes the page-table textures and page-table indirection texture (`RenderCore/Public/VirtualTexturing.h:407-408`), per-layer physical textures (`L410`), and page-table format `UInt16`/`UInt32` (`L363`, `L463`).
- Defragmentation can move a buffer (`RHI/Public/GPUDefragAllocator.h:42`, `DefragmentMemory` `L613`, `Tick` `L623`) — never cache a resource address across frames.

## Profiling hooks a tech artist should use

| Hook | Macro | Where |
|---|---|---|
| RDG pass name | `RDG_EVENT_SCOPE` / `RDG_EVENT_SCOPE_STAT` | `RenderCore/Public/RenderGraphEvent.h:479`, `480` |
| Conditional name | `RDG_EVENT_SCOPE_CONDITIONAL_NAME` | `L486` |
| Draw event | `SCOPED_DRAW_EVENTF`, `RHI_BREADCRUMB_EVENT_CONDITIONAL_F` | `InstanceCullingContext.cpp:1787` example |
| Quick cycle counter | `QUICK_SCOPE_CYCLE_COUNTER` | `InstancedStaticMesh.cpp:1104` example |
| Breadcrumb color | deterministic from FNV-1a of the name | `RHIBreadcrumbs.h:84` |

Breadcrumb names become ProfileGPU tree nodes (`GPUProfiler.cpp:1636-1646`, `1681-1686`), so naming a scope well is the difference between a readable frame and an unreadable one.

## CVars worth knowing

| CVar | Default | Effect |
|---|---|---|
| `r.RHICmdBypass` | 0 | direct execution, no command list; disables parallel (`RHICommandList.h:5087`) |
| `r.RHICmdWidth` | 8 | task granularity (`RHICommandList.cpp:63-66`) |
| `r.RHICmdMinDrawsPerParallelCmdList` | 64 | below this, no parallel command lists (`SceneRendering.cpp:512-515`) |
| `r.RHIThread.Enable` | via `GPendingRHIThreadMode` (`RHICommandList.h:106`) | RHI thread mode |
| `r.GPUCrashDebugging.Breadcrumbs` | 1 | crash context breadcrumbs (`RHI/Private/RHI.cpp:2133-2143`) |
| `r.ProfileGPU.Root` / `ThresholdPercent` | `*` / 0 | ProfileGPU tree filter (`GPUProfiler.cpp:35-115`) |
| `r.Trace.Channels` | Submission | RenderTraceInsights channels (`RenderTracing.cpp:62-68`) |
| `r.RDG.ClobberResources` | off | debug fill, depth `0.123456789f`, stencil `123` (`RenderGraphPrivate.cpp:196-203`) |
| `r.ShaderPipelineCache.*` | see chapter 06 | PSO file cache |

Build flags: `RHI_WANT_RESOURCE_INFO` for non-Shipping (`RHI.Build.cs:32-35`), `WITH_MGPU` Windows desktop only (`L23`), `RDG_ENABLE_DEBUG` excludes Shipping/Test (`RenderGraphDefinitions.h:17`), `WITH_RHI_BREADCRUMBS` gate (`RHIBreadcrumbs.h:26`).

## Pitfalls

1. **Instancing that does not instance.** Different LOD per instance, or a VF without a primitive-id stream (`VertexFactory.h:744`, `751`), prevents merge.
2. **Dynamic instancing is not available on the `SubmitMeshDrawCommandsRange` path** (`MeshPassProcessor.cpp:1660` `checkSlow(!bDynamicInstancing)`).
3. **Screen-size precision.** 13-bit fixed-point packing (`MeshPassProcessor.h:1688`) means very small or very large screen sizes round.
4. **Format changes create PSOs.** A post-processing pass that changes format invalidates the pipeline cache for every material in it.
5. **Access-state mistakes are silent on some RHIs** and fatal on others; validation asserts them (`RHIValidationContext.h:934-968`).
6. **Shipping loses resource names** (`RHI_WANT_RESOURCE_INFO`), so memory profiling in Shipping needs a non-Shipping config or `bTrackRHIResourceInfoForTest`.
