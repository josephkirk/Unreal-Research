# 18 — Tech Artist Guide

Practical control surface: which knob changes which stage, what each knob actually gates, and what a symptom means in the pipeline. Every line reference is to UE 5.8 source under `UE_5.8`.

## What a knob actually gates

There are three distinct kinds of control, and confusing them causes most "I changed the CVar and nothing happened" reports:

| Kind | Meaning | Example |
|---|---|---|
| Support gate | Whether the feature can exist for the project at all, independent of platform | `CVarMegaLightsSupported` (`MegaLights/MegaLights.cpp:19`), comment `:22` "Whether MegaLights is supported at all for the project, regardless of platform. This can be used to avoid compiling MegaLights shaders" |
| Permutation gate | Whether the shader is compiled at all (`ShouldCompilePermutation`, default `return true` at `RenderCore/Public/Shader.h:848`) | `FSubstrateTileType : SHADER_PERMUTATION_INT("SUBSTRATE_TILETYPE", 4)` (`Substrate/Substrate.h:320`) |
| Runtime gate | Whether the pass runs this frame | `PassSequence.SetEnabled(...)` (`PostProcess/PostProcessing.cpp:624`-`:685`), `IsHairStrandsSupported()` (`HairStrands/HairStrandsInterface.cpp:175`) |

A show flag is a runtime gate. `PostProcessing.cpp:785` documents the exception: eye adaptation is not gated by its show flag — "If disabled, the auto exposure pass is still executes but performs a...".

## Symptom → knob

| Symptom | Where to look | Verified anchor |
|---|---|---|
| Shading looks wrong on a specific material | Shading model ID is 4 bits; the model must be in the 14-value set | `Engine/Shaders/Private/ShadingCommon.ush:20`-`:34`, `SHADINGMODELID_MASK 0xF` `:35` |
| Custom data channel reads zero | Slot packing differs per compression mode | `RenderCore/Public/GBufferInfo.h:31` (CustomData RGBA8 uncompressed), `EGBufferCompression` `:47`-`:80` |
| Velocity missing on some materials | `SKIP_VELOCITY_MASK (1<<7)` | `ShadingCommon.ush:43` |
| First-person shading only works in dynamic lighting | `IS_FIRST_PERSON_MASK` shares bit 6 and "only works if `ALLOW_STATIC_LIGHTING` is 0" | `ShadingCommon.ush:41`, `:42` |
| Nanite mesh detail lower than expected | `r.Nanite.MaxPixelsPerEdge` | `Nanite/NaniteCullRaster.cpp:133`; primary raster scaling `:332` |
| Nanite raster cost spikes | Binning technique and shade binning mode | `NaniteShading.cpp:202`, `:210`, validation `:233` |
| Hair cost scales with screen percentage unexpectedly | `r.HairStrands.Visibility.ResolutionScaleAware` | `HairStrands/HairStrandsVisibility.cpp:100` |
| Hair sample budget too aggressive | Per-model sample CVars (MSAA `:88`, compute `:93`, PPLL `:114`) | `HairStrandsVisibility.cpp:88`-`:115` |
| Substrate tiles allocate too much memory | Allocation mode, clamped 0-2 | `Substrate/Substrate.cpp:61`, `:165`, `:167` |
| Substrate resolution looks blocky | Tile size is fixed at 8; the 8-alignment contract is upheld by `QuantizeSceneBufferSize()`, not by the assert | `Engine/Shaders/Shared/SubstrateDefinitions.h:44`, `Substrate.cpp:202`-`:203` (tile-count division), comment `:211`, assert `:213` checks only `& 0x3` (multiple of 4) |
| Lumen card capture stalls | Card update factor and feedback | `Lumen/LumenSceneLighting.cpp:41`, `:25`, `:32` |
| Lumen reflections disappear at grazing angles | Roughness fade length and max roughness to trace | `Lumen/LumenReflections.cpp:93`, `:108` |
| Lumen noise at distance | Radiance cache supersample distance and downsample distance | `Lumen/LumenRadianceCache.cpp:101`, `:109` |
| MegaLights flickers on newly visible lights | History distance threshold and min frames accumulated | `MegaLights/MegaLightsDenoising.cpp:24`, `:31` |
| MegaLights misses a light entirely | Min sample weight and light index scalarization threshold | `MegaLightsSampling.cpp:14`, `MegaLightsResolve.cpp:66` |
| Screen-space reflections vanish | Min specular clamped to `[-0.001, 1.001]` | `ScreenSpaceReflectionTiles.cpp:18`, `:183` |
| Post process material does not use a blend mode | `r.PostProcess.AllowBlendModes` | `PostProcess/PostProcessMaterial.cpp:49`, `:131`; stencil `:41`, `:67` |
| Auto exposure does not change when show flag is off | Show flag does not gate the pass | `PostProcess/PostProcessing.cpp:785`, `:786` |
| Bloom quality change reallocates textures | Downsample chain stage count must equal the quality enum | `PostProcess/PostProcessBloomSetup.cpp:199` |
| Resolution pumping frame to frame | Change threshold and increase amortization | `RenderCore/Public/DynamicRenderScaling.h:39`, `:41` |
| Shading rate map does not match expectation | Foveation level clamped `[0,4]`; tile size sanity check | `VariableRateShading/FoveatedImageGenerator.cpp:151`, `:200` |
| Instance culling does nothing | `r.CullInstances` and `r.OcclusionCullInstances` | `InstanceCulling/InstanceCullingContext.cpp:28`, `:34`-`:35`; gates `:211`-`:213`, `:216`-`:218` |
| Draw calls higher than expected | Mesh draw command stats and budgets | `Engine/Public/MeshDrawCommandStatsSettings.h:60`, `:62`, `:65` |
| Ray tracing passes absent | `r.RayTracing` plus platform support | `DeferredShadingRenderer.cpp:277`, `:693`; `DataDrivenShaderPlatformInfo.h:1204`, `:1222`, `:1230` |

## Reading a frame

The frame order is the ground truth; chapter `05_Pass_Catalog_and_Frame_Order.md` lists it with RDG event-scope names. The names are the strings to search for in RenderDoc or the GPU profiler. Examples verified in source: `Light::ClusteredDeferredShading(%s,Lights:%d%s%s%s%s)` (`ClusteredDeferredShadingPass.cpp:246`), `ScreenSpaceShadowing %ux%u` (`Shadows/ScreenSpaceShadows.cpp:339`), `TiledCapsuleShadowing` (`CapsuleShadowRendering.cpp:597`), `VisualizeGBufferHints` (`PostProcess/PostProcessGBufferHints.cpp:65`), `GenerateSamples DownsampleFactor:%dx%d SamplesPerPixel:%dx%d TileType:%s` (`MegaLightsSampling.cpp:434`), `Histogram %dx%d (CS)` (`PostProcess/PostProcessHistogram.cpp:289`), `FFTBloom SurveyKernel(TexelDiameter=%f)` (`PostProcessFFTBloom.cpp:422`), `ResampleLumenCards` (`LumenSceneRendering.cpp:1886`), `MarkRadianceProbes(Visualize)` (`LumenRadianceCache.cpp:292`), `Substrate::VisualizeMaterial(Print)` (`Substrate/SubstrateVisualize.cpp:1059`), `WaterInfoTexture(Terrain)` (`WaterInfoTextureRendering.cpp:921`), `SSR::TileCategorisationMarkTiles` (`ScreenSpaceReflectionTiles.cpp:188`), `InitIndirectArgs1D` (`RenderGraphUtils.cpp:895`).

## Visualizers worth using

| Visualizer | CVar | Anchor |
|---|---|---|
| GBuffer hints | show flag `GBufferHints` | enable `PostProcess/PostProcessing.cpp:668`; text `PostProcess/PostProcessGBufferHints.cpp:80` |
| Substrate material tree | `Substrate::VisualizeMaterial` | `SubstrateVisualize.cpp:1059`, `:1090`, `:1129`, `:1469`, `:1540` |
| MegaLights light complexity | `r.MegaLights.VisualizeLightComplexity.*` | `MegaLightsVisualizeLightComplexity.cpp:10`, `:24`, `:36`, `:43`, `:50` |
| MegaLights mode | `r.MegaLights.Visualize` | `MegaLightsVisualize.cpp:13`, `:58` |
| Lumen visualize | `r.Lumen.Visualize.Mode` | `PostProcessing.cpp:631`, `:671`; `LumenRadianceCache.cpp:292` |
| Hair tile debug | `r.HairStrands.*.Debug` | `HairStrandsVisibility.cpp` debug, `HairStrandsDebug.cpp` |
| VT residency | `r.VT.Residency.Show` / `.Notify` | `VT/VirtualTextureSystem.cpp:128`, `:134` |
| VSM debug | `VirtualShadowMapDebug.usf` | `VirtualShadowMaps/` shader dir (37 files) |
| Nanite draw events | `r.Nanite.ShowDrawEvents` | `NaniteShading.cpp:96` |
| Temporal upscaler inputs | show flag `VisualizeTemporalUpscaler` | `PostProcessing.cpp:675`, inputs `:1188`-`:1191` |
| Motion blur debug | `r.PostProcessMotionBlur.Debug` | `PostProcess/PostProcessMotionBlur.cpp:71`-`:74`, `:78` |
| SMAA edge/weight debug | `r.SMAA.DebugVisualization` | `PostProcess/SubpixelMorphologicalAA.cpp:22` |

## Invariants a tech artist should not violate

1. Shading model IDs occupy the low 4 bits; flags occupy the high 4 (`ShadingCommon.ush:19`). A material cannot use both a model ID ≥ 16 and a flag.
2. `IS_FIRST_PERSON_MASK` and `ZERO_PRECSHADOW_MASK` share bit 6 (`:41`, `:42`); the overlap is only safe under static-lighting-disabled configurations.
3. Substrate tile type must map to the `SUBSTRATE_TILE_TYPE` defines (`Engine/Public/Rendering/SubstrateMaterialShared.h:67`) and stays ≤ `EComplexSpecial` (`Substrate.cpp:228`).
4. Substrate resolution must be a multiple of `SUBSTRATE_TILE_SIZE` (`SubstrateDefinitions.h:44`); the runtime assert is weaker — `Substrate.cpp:213` checks `& 0x3` (multiple of 4) while the comment at `:211` states the 8-alignment is guaranteed by `QuantizeSceneBufferSize()`. A resolution that is 4- but not 8-aligned passes the assert and mis-tiles.
5. View rect must be tile-aligned for stochastic lighting (`StochasticLighting/StochasticLighting.cpp:370`).
6. Lumen card capture cannot bind a bindless shading command (`LumenSceneCardCapture.cpp:259`) and requires exactly 3 render targets (`:547`).
7. A pipeline state ID used after release fails `checkf` (`MeshPassProcessor.h:655`).
8. A mesh pass processor that was never registered fails `checkf` (`MeshPassProcessor.h:2417`).
9. Shader binding order is asserted at draw time (`MeshPassProcessor.h:1047`), and an invalid frequency fails `:1056`.
10. `checkf(false, TEXT("Unknown culling type: %d"))` (`NaniteCullRaster.cpp:4496`) means a culling path was added in C++ without a matching shader permutation.

## Graph facts

- Renderer module: 22,449 nodes; RenderCore 7,571; RHI 5,286; Engine 109,367; Developer 88,087; Core 41,572 — 274,332 render-scope nodes of 2,688,593 total.
- Highest-degree public Renderer file: `Public/MeshPassProcessor.h` at 1,230 edges; `ScenePrivate.h` 2,737; `SceneRendering.h` 1,977; `RendererScene.cpp` 1,433.
- Renderer → RenderCore 4,633 edges, Renderer → RHI 1,070, Renderer → Core 4,137. Shader files are not graph nodes (0 shader nodes), so shader-side relationships are not in the graph.
