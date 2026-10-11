# 19 — File Reference

Authoritative file counts (`find | wc -l`) for UE 5.8 at `UE_5.8`, plus the files that actually carry the behaviour described in the other chapters.

## Top-level counts

| Root | Files |
|---|---|
| `Engine/Source/Runtime/Renderer/Private` | 705 |
| `Engine/Source/Runtime/Renderer/Public` | 50 |
| `Engine/Shaders/Private` | 1,146 |
| `Engine/Shaders/Public` | 22 |
| `Engine/Shaders/Shared` | 51 |

`Engine/Shaders/Compute` does not exist. `Renderer/Private` has 261 files at its root and 444 in 24 subdirectories.

## `Renderer/Private` by subdirectory

| Directory | Files | Directory | Files |
|---|---|---|---|
| PostProcess | 106 | VirtualShadowMaps | 9 |
| Lumen | 64 | InstanceCulling | 9 |
| Nanite | 38 | HeterogeneousVolumes | 9 |
| VT | 36 | Substrate | 8 |
| RayTracing | 34 | SparseVolumeTexture | 8 |
| HairStrands | 33 | MaterialCache | 8 |
| Shadows | 13 | CompositionLighting | 8 |
| Skinning | 12 | SceneCulling | 6 |
| MegaLights | 12 | VariableRateShading | 5 |
| StateStream | 10 | StochasticLighting | 5 |
| — | — | Froxel | 4 |
| — | — | OIT | 3 |
| — | — | Tests | 2 |
| — | — | Renderer | 2 |

## `Engine/Shaders/Private` by subdirectory

| Directory | Files | Directory | Files |
|---|---|---|---|
| Lumen | 95 | ScreenSpaceDenoise | 20 |
| HairStrands | 79 | TemporalSuperResolution | 23 |
| RayTracing | 64 | Tools | 11 |
| Nanite | 58 | Bloom | 11 |
| Substrate | 25 | MaterialCache | 9 |
| VirtualShadowMaps | 37 | ACES | 8 |
| InstanceCulling | 7 | SMAA | 7 |
| MegaLights | 32 | SSRT | 6 |
| VT | 36 | SceneCulling | 3 |
| — | — | LocalFogVolumes | 3 |
| — | — | GPUScene | 2 |
| — | — | PostProcessing | 2 |
| — | — | OIT | 1 |

## Files that carry the behaviour

| Area | File | Why |
|---|---|---|
| Frame entry | `Renderer/Private/SceneRendering.cpp` | `RenderViewFamily_RenderThread` `:5215`, `Renderer->Render` `:5229`, `FlushCrossGPUFences` `:5232` (def `:4108`) |
| Frame order | `Renderer/Private/DeferredShadingRenderer.cpp` | `Render` `:1823`; the 50-row order in chapter 05 |
| View types | `Engine/Source/Runtime/Engine/Public/SceneView.h` | `FSceneView` `:1481`, `FSceneViewFamily` `:2320`, `FSceneViewFamilyContext` `:2777` |
| View data management | `Renderer/Private/ViewData.cpp` | `FRendererViewDataManager` `:6`, `AllocateViews` `:74`, `FlushRegisteredViews` `:113` |
| Scene extensions | `Renderer/Private/SceneExtensions.h` / `.cpp` | interfaces `:23`, `:44`, `:68`; registry `:132`; macros `:403`-`:418` |
| Mesh passes | `Renderer/Public/MeshPassProcessor.h` | `EMeshPass` `:56`-`:108`, `FMeshDrawCommand` `:1281`, `FMeshPassProcessor` `:2258` |
| Draw submission | `Renderer/Private/SimpleMeshDrawCommandPass.h` | `SubmitDraw` `:38`, `AddPass` `:98` |
| Instance culling | `Renderer/Public/InstanceCulling/InstanceCullingContext.h` | `FInstanceCullingContext` `:75`, static slot `:95` |
| GPUScene | `Renderer/Private/GPUScene.h` | `FGPUScene` `:26`, parameter structs `:34`-`:73` |
| GBuffer | `RenderCore/Public/GBufferInfo.h` | slot/compression/type/checker/layout enums `:12`-`:115` |
| GBuffer shader | `Engine/Shaders/Private/DeferredShadingCommon.ush` | `FGBufferData` `:379`, encode `:968`, decode `:1076` |
| Shading models | `Engine/Shaders/Private/ShadingCommon.ush` | IDs `:20`-`:34`, masks `:39`-`:43` |
| Shadows | `Renderer/Private/Shadows/ShadowScene.cpp`, `VirtualShadowMaps/VirtualShadowMapArray.cpp`, `VirtualShadowMapCacheManager.cpp` | extension `:44`, page asserts `:1012`-`:1172`, extension `:645` |
| Nanite | `Renderer/Private/Nanite/NaniteCullRaster.cpp`, `NaniteShading.cpp`, `NaniteRayTracing.cpp` | bit-field asserts `:46`-`:49`, bin passes `:1221`-`:1293`, CVars `:115`-`:250` |
| Lumen | `Renderer/Private/Lumen/LumenSceneData.h`, `LumenSceneLighting.cpp`, `LumenSceneCardCapture.cpp` | card types `:43`-`:307`, update policy `:17`-`:64`, asserts `:259`-`:910` |
| MegaLights | `Renderer/Private/MegaLights/MegaLights.cpp`, `MegaLightsInternal.h`, `MegaLightsRayTracing.cpp` | gates `:19`-`:143`, `FMegaLightsViewContext` `:375`, frozen-data asserts `:491`-`:543` |
| Substrate | `Renderer/Private/Substrate/Substrate.cpp`, `Substrate.h`, `Engine/Public/Rendering/SubstrateMaterialShared.h`, `Engine/Shaders/Shared/SubstrateDefinitions.h` | constants `:21`-`:65`, tile enum `:68`-`:79`, classification `:261` |
| Compute infra | `Renderer/Private/GPUWorkGroupLoadBalancer.h`, `Engine/Shaders/Private/WorkGroupLoadBalancer.ush`, `WorkDistribution.ush`, `ThreadGroupPrefixSum.ush`, `WaveOpUtil.ush` | bit packing `GPUWorkGroupLoadBalancer.h:18`-`:61`; `#error` guard `ThreadGroupPrefixSum.ush:24`; `WavePrefixSum` `WaveOpUtil.ush:71` |
| Post process | `Renderer/Private/PostProcess/PostProcessing.cpp`, `Renderer/Public/PostProcess/PostProcessMaterialInputs.h` | execution order `:412`-`:1676`, input aliases `:21`-`:38` |
| Screen space | `Renderer/Private/SceneTextureReductions.cpp`, `ScreenSpaceRayTracing.cpp`, `ScreenSpaceReflectionTiles.cpp`, `ScreenSpaceDenoise.cpp` | HZB `:16`-`:378`, static asserts `:134`-`:145` |
| Translucency | `Renderer/Private/TranslucentRendering.cpp`, `OIT/OIT.cpp`, `OIT/OIT.h` | bucket `:172`-`:181`, pool `:72`-`:74` |
| Volumetrics | `Renderer/Private/VolumetricFog.cpp`, `Froxel/Froxel.h`, `LocalFogVolumeRendering.cpp`, `HeterogeneousVolumes/HeterogeneousVolumes.cpp` | grid `:118`-`:239`, froxel geometry `:12`-`:15`, gates `:18`-`:123` |
| SVT | `Engine/Source/Runtime/Engine/Private/SparseVolumeTexture/SparseVolumeTextureData.cpp`, `Renderer/Private/SparseVolumeTexture/` | bounds asserts `:73`-`:343`, viewer only in Renderer |
| Hair | `Renderer/Private/HairStrands/HairStrandsVisibility.cpp`, `HairStrandsData.h`, `HairStrandsInterface.cpp` | sample budgets `:83`-`:118`, structures `:60`-`:240`, support gate `:175` |
| Decals | `Renderer/Private/DecalRenderingCommon.h`, `DecalRenderingShared.h` | stage/target enums `:35`-`:72`, write mask `:24`-`:30` |
| Virtual texturing | `Renderer/Private/VT/VirtualTextureSystem.h`, `AdaptiveVirtualTexture.cpp` | system `:27`, CVars `:79`-`:147`, asserts `:173`-`:279` |
| Scaling | `RenderCore/Public/DynamicRenderScaling.h`, `Renderer/Private/VariableRateShading/` | heuristic defaults `:35`-`:41`, VRS enums `:36`-`:64` |
| RDG | `RenderCore/Public/RenderGraphUtils.h`, `RenderCore/Private/RenderGraphPrivate.h` | group sizing `:336`-`:402`, counters `:164`-`:190` |

## Paths that are not where you would expect

- `SceneView.h` is in `Engine/Source/Runtime/Engine/Public/`, not `Renderer/Public/`.
- `NaniteResources.h` and `NaniteStreamingManager.h` are in `Engine/Public/Rendering/`, not `Renderer/Private/Nanite/`.
- `SubstrateMaterialShared.h` is in `Engine/Public/Rendering/`.
- `CapsuleShadowRendering.cpp` is in `Renderer/Private` root, not `Renderer/Private/Shadows/`.
- `PostProcessMaterialInputs.h` is in `Renderer/Public/PostProcess/`, not `Private/`.
- `DynamicRenderScaling` is in `RenderCore`, not `Renderer`.
- `SparseVolumeTextureData.cpp` is in `Engine/Private/SparseVolumeTexture/`; `Renderer/Private/SparseVolumeTexture/` is the viewer only.
- `LocalFogVolumeSceneProxy.cpp` is in `Engine/Private/`, not `Renderer/Private/`.
- `ThreadGroupPrefixSum.ush`, `WorkGroupLoadBalancer.ush`, `WorkDistribution.ush` are at `Engine/Shaders/Private/` root, not in `Tools/`.
- `Renderer/Internal/` exists (e.g. `Renderer/Internal/PostProcess/SubpixelMorphologicalAA.h`); those subsystems are not callable from outside the Renderer module.

## Graph facts

- Render-scope nodes 274,332 of 2,688,593 total; links 5,061,968. Shader files are not graph nodes (0 shader nodes), so the graph documents C++ structure only.
- Top render-scope files by node count: `Engine/Private/Materials/MaterialExpressions.cpp` 1481, `Renderer/Private/ScenePrivate.h` 862, `RHI/Public/RHIResources.h` 840, `Engine/Private/UnrealEngine.cpp` 726, `Engine/Public/MaterialCompiler.h` 722, `HLSLMaterialTranslator.h` 585, `Renderer/Private/SceneRendering.h` 548, `MaterialShared.h` 544, `Scene.h` 451, `RHICommandList.h` 446, `Lumen/LumenSceneData.h` 409, `RenderCore/Public/Shader.h` 379, `ShaderCompiler.h` 376.
- Relations in render scope: references 370,567; calls 150,141; contains 129,319; defines 109,759; imports 86,623; method 36,403; inherits 10,058.
- Cross-module edges touching Renderer: → unknown 21,820; → RenderCore 4,633; → Core 4,137; Experimental → Renderer 2,352; Runtime → Renderer 1,649; → Engine 1,441; Editor → Renderer 1,407; → CoreUObject 1,322; Developer → Renderer 1,320; Engine → Renderer 1,076; → RHI 1,070; VirtualProduction → Renderer 996; Online → Renderer 912; Media → Renderer 597.
