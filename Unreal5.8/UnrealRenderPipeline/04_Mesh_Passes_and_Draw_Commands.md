# 04 — Mesh Passes and Draw Commands

## Two-phase design

Mesh passes are not executed when they are visited in code. They are **collected** during visibility and **dispatched** later:

1. A scene proxy emits `FMeshBatch` per pass during `FSceneRenderer` visibility.
2. A `FMeshPassProcessor` (`Renderer/Public/MeshPassProcessor.h:2258`, implements `IPSOCollector`) turns each batch into an `FMeshDrawCommand` via `AddMeshBatch` (`:2284`).
3. Dispatch runs the command set against the RHI, using instance-culling indirect args when GPUScene is active (`FInstanceCullingDrawParams`, forward-declared `:47`).

The split matters because the processor is where material, vertex factory and pass compatibility are resolved. A batch that fails compatibility never becomes a draw command — it is dropped silently, which is why "the mesh is not drawn" is usually a processor decision, not a dispatch bug.

## Key types

| Type | Line | Contents |
|---|---|---|
| `FMeshDrawCommand` | `MeshPassProcessor.h:1281` | `ShaderBindings` (`:1288`), `VertexStreams` (`:1289`), `IndexBuffer` (`:1290`), `CachedPipelineId` (`:1295`), `FirstIndex`/`NumPrimitives`/`NumInstances` (`:1300-1302`), and a union of direct vertex params (`:1306-1311`) or `IndirectArgs` (`:1316`) |
| `FMeshDrawShaderBindings` | `:966` | per-command parameter bytes; ordering is enforced by `checkf(CheckedDataOffset == DataOffset, TEXT("GetSingleShaderBindings was not called in the order of ShaderFrequencies"))` (`:1047`) |
| `FMeshDrawCommandPrimitiveIdInfo` | `:1632` | scene primitive id, persistent primitive index, instance scene data offset — the GPUScene addressing triple |
| `FVisibleMeshDrawCommand` | `:1762` | the visible form produced by culling; carries `PrimitiveIdInfo` (`:1801`) |
| `FMeshPassProcessorRenderState` | `:2177` | blend, depth-stencil, rasterizer state for the pass |
| `FMeshPassDrawListContext` | `:1729` | draw list; dynamic (`:1857`) vs cached (`:2023`) variants |
| `FMeshDrawCommandStatsBudget` | `Engine/Public/MeshDrawCommandStatsSettings.h:12` | `PrimitiveBudget` (`:24`), `VertexBudget` (`:27`); budgets collected under `r.MeshDrawCommands.Stats` (`:60-65`) |

Cached vs dynamic: `FCachedPassMeshDrawListContext` stores commands across frames for static meshes; `FDynamicPassMeshDrawListContext` rebuilds them per frame. A dynamic command cannot be addressed through GPUScene static instance data, which is why occlusion culling asserts `!bIsDynamicPrimitive` (`InstanceCullingOcclusionQuery.cpp:329`, see [03_Instance_Culling_and_GPUScene.md](03_Instance_Culling_and_GPUScene.md)).

## Pass catalog

`EMeshPass::Type` (`MeshPassProcessor.h:58`) is a `uint8` with `NumBits = 6` (`:108`) and `static_assert(EMeshPass::Num <= (1 << EMeshPass::NumBits))` (`:111`) — the catalog is capped at 64 pass types, so adding a pass is a bounded resource decision.

| Pass | Line | Notes |
|---|---|---|
| `DepthPass` | `:60` | depth prepass |
| `SecondStageDepthPass` | `:61` | second-stage depth (stochastic rendering) |
| `BasePass` | `:62` | GBuffer / forward shading |
| `AnisotropyPass` | `:63` | anisotropic mask pass |
| `SkyPass` | `:64` | sky / background |
| `SingleLayerWaterPass` | `:65` | water surface |
| `SingleLayerWaterDepthPrepass` | `:66` | water depth |
| `CSMShadowDepth` | `:67` | cascaded shadow map |
| `VSMShadowDepth` | `:68` | virtual shadow map page render |
| `OnePassPointLightShadowDepth` | `:69` | single-pass point shadow |
| `Distortion` | `:70` | translucent distortion |
| `Velocity` | `:71` | opaque velocity |
| `TranslucentVelocity` | `:72` | translucent velocity |
| `TranslucentVelocityClippedDepth` | `:73` | pixels below the opaque threshold |
| `TranslucencyStandard` | `:74` | main translucent pass |
| `TranslucencyStandardModulate` | `:75` | modulate variant |
| `TranslucencyAfterDOF` | `:76` | translucent after DOF |
| `TranslucencyAfterDOFModulate` | `:77` | modulate variant |
| `TranslucencyAfterMotionBlur` | `:78` | translucent after motion blur |
| `TranslucencyHoldout` | `:79` | holdout, infers background visibility |
| `TranslucencyAll` | `:80` | used outside the main renderer (e.g. `FRendererModule::DrawTile`) |
| `LightmapDensity` | `:81` | debug view mode |
| `DebugViewMode` | `:82` | any `EDebugViewShaderMode` |
| `CustomDepth` | `:83` | custom depth / custom stencil |
| `MobileBasePassCSM` | `:84` | mobile base pass with CSM |
| `VirtualTexture` | `:85` | VT page rendering |
| `LumenCardCapture` | `:86` | Lumen mesh card capture |
| `LumenCardNanite` | `:87` | Lumen cards for Nanite geometry |
| `LumenTranslucencyRadianceCacheMark` | `:88` | radiance cache marking |
| `FrontLayerTranslucencyGBuffer` | `:89` | front-layer translucency GBuffer |
| `DitheredLODFadingOutMaskPass` | `:90` | used by ray-traced shadows |
| `NaniteMeshPass` | `:91` | Nanite raster/shading |
| `MeshDecal_DBuffer` | `:92` | DBuffer decals |
| `MeshDecal_SceneColorAndGBuffer` | `:93` | deferred decals |
| `MeshDecal_SceneColorAndGBufferNoNormal` | `:94` | deferred decals without normal |
| `MeshDecal_SceneColor` | `:95` | modulated decals |
| `MeshDecal_AmbientOcclusion` | `:96` | decal AO |
| `WaterInfoTextureDepthPass` | `:97` | water info depth |
| `WaterInfoTexturePass` | `:98` | water info texture |
| `HitProxy`, `HitProxyOpaqueOnly`, `EditorLevelInstance`, `EditorSelection` | `:101-104` | `#if WITH_EDITOR` only |

`GetMeshPassName` (`:114`) has a `checkf(0, TEXT("Missing case for EMeshPass %u"))` fallthrough (`:171`) — a new pass without a name case fails at runtime in the first call, not at compile time.

Processor creation is a jump table indexed by shading path and pass type (`CreateMeshPassProcessor` `:2413`), with `checkf(JumpTable[...][PassType] || DeprecatedJumpTable[...][PassType], TEXT("Pass type %u create function was never registered"))` (`:2417`). A pass type that exists in the enum but has no registered processor for the active shading path is a hard failure.

## RDG-facing dispatch

`FSimpleMeshDrawCommandPass` (`Renderer/Public/SimpleMeshDrawCommandPass.h:24`) is the wrapper that lets a command set run inside a graph pass: `SubmitDraw` (`:38`) records the draw, and the helper `AddPass` (`:98`, `:173`) registers it as an RDG pass with the instance-culling draw params. The implementation asserts that the view is a real `FViewInfo` (`Private/SimpleMeshDrawCommandPass.cpp:13`, `:50`) and that an instanced view exists for GPUScene paths (`:21`).

Pass model, load actions and subpass rules are covered in [../RDG/06_Render_Passes_and_Subpasses.md](../RDG/06_Render_Passes_and_Subpasses.md); the dispatch-pass mechanics in [../RDG/12_Pipeline_Integration.md](../RDG/12_Pipeline_Integration.md); GPUScene addressing in [../RHI/08_Instance_Culling_and_GPUScene.md](../RHI/08_Instance_Culling_and_GPUScene.md) and [../RHI/09_Frame_Flow_and_Mesh_Passes.md](../RHI/09_Frame_Flow_and_Mesh_Passes.md).

## Design constraints

1. **Pipeline state IDs are reference-counted.** `checkf(Result.Value.RefNum && DebugSalt == DebugSalt, TEXT("Pipeline state ID used after release"))` (`:655`) — a command that outlives its PSO is a use-after-release, and the check fires before the GPU does anything.
2. **Shader binding order is fixed by frequency.** `:1047` and `:1056` enforce that bindings are written in shader-frequency order; a processor that writes bindings out of order corrupts the buffer without a visible error until the check fires.
3. **Ray-tracing shader frequency must match.** `checkf(0, TEXT("Requested raytracing shader frequency (%d) doesn't match assigned shader frequency (%d)"))` (`:811`).
4. **Pass count is bounded** by `NumBits = 6` (`:108`, `:112`).
5. **Editor passes do not exist in shipping** (`:100-105`), so hit-proxy code paths must be guarded by `WITH_EDITOR`, not by show flags.

## Tech artist notes

- A material that fails to compile for a pass produces no draw command for that pass; the mesh disappears in that pass only. Checking `r.MeshDrawCommands.Stats` budgets (`MeshDrawCommandStatsSettings.h:12-65`) shows whether the command count is what you expect.
- Static vs dynamic path choice is a material/proxy decision with performance consequences: cached commands survive frames, dynamic commands are rebuilt every frame and cannot use static GPUScene instance data.
- Vertex factory and material compatibility are resolved in the processor; a "works in editor, missing at runtime" mesh is usually a feature-level or shader-platform permutation gap ([../RHI/10_Feature_Levels_and_Shader_Platforms.md](../RHI/10_Feature_Levels_and_Shader_Platforms.md)).

## Graph facts

- `Public/MeshPassProcessor.h` is the highest-degree public Renderer file: 1,230 edges. It contains `FMeshDrawCommand` (`:1281`), `FMeshDrawShaderBindings` (`:966`), `FVisibleMeshDrawCommand` (`:1762`) and `FMeshPassProcessor` (`:2258`).
- Mesh-pass subsystem files sit in the Renderer module root (11,036 nodes), not in a subsystem directory; the pass catalog is therefore part of the "Scene View and Materials" community (18, 2,112 nodes).
- Shader files are not graph nodes; the pass catalog above is C++ only.
