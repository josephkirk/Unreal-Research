# 20 — Testing

The render pipeline has almost no unit tests. Enforcement is by runtime asserts and by validation that checks API inputs and lifecycle, not graph decisions. Verification of behaviour is observational: event scopes, visualizers, and stats.

## Automation tests

`Renderer/Private/Tests/` contains exactly 2 files, both gated by `#if WITH_DEV_AUTOMATION_TESTS || WITH_EDITOR`:

| Test | Test name | Flags | Anchor |
|---|---|---|---|
| `FRayTracingTestbed` | `System.Renderer.RayTracing.BasicRayTracing` | `EditorContext \| HighPriority \| EngineFilter` | `Tests/RayTracingTestbed.cpp:40` |
| `FVirtualTextureTestbed` | `System.Renderer.VirtualTexture` | `EditorContext \| ClientContext \| EngineFilter` | `Tests/VirtualTextureTestbed.cpp:11` |

Both are `IMPLEMENT_CUSTOM_SIMPLE_AUTOMATION_TEST` on `FAutomationTestBase` (`RayTracingTestbed.cpp:11`, `VirtualTextureTestbed.cpp:11`). `RayTracingTestbed.cpp` defines its own `FTestRaygenShader : public FGlobalShader` (`:321`) — the test compiles a permutation and dispatches it, so it exercises the shader-permutation path, not the pipeline. No test covers Nanite, Lumen, MegaLights, Substrate, shadows, post process, or instance culling.

## What RDG validation actually checks

`FRDGUserValidation` (`RenderCore/Public/RenderGraphValidation.h:40`) validates API inputs and lifecycle only:

- resource creation inputs — `ValidateCreateTexture` `:48`, `ValidateCreateBuffer` `:49`, SRV `:50`-`:51`, UAV `:52`-`:53`, uniform buffer `:54`
- external resource registration — `:57`-`:63`, `ValidateRegisterExternalTexture` `:65`-`:68`, `:75`
- upload and commit — `ValidateUploadBuffer` `:78`-`:82`, `ValidateCommitBuffer` `:84`
- extraction and conversion — `:87`-`:91`
- pass addition — `ValidateAddPass` `:98`-`:100`, with `bSkipPassAccessMarking` documented at `:94`-`:96` for debug-injected passes
- pass state before/after execution — `:103`, `:104`
- graph state before/after execution — `:107`, `:108`
- clobber tracking — `TryMarkForClobber` `:111`
- pooled resource access — `:113`, `:114`

It does not validate whether a pass *should* have run, whether an access mask was semantically correct for the shader that read it, or whether a resource was live at the point a subsystem assumed it was. Those are enforced by the subsystem's own asserts.

## Assert inventory (the real enforcement)

| Mechanism | Example | Anchor |
|---|---|---|
| `static_assert` on bit-field width | `FCandidateNode.x` field limits | `Nanite/NaniteCullRaster.cpp:46`-`:48`, `NANITE_MAX_INSTANCES <= MAX_INSTANCE_ID` `:49` |
| `static_assert` on capacity | `VSM_MAX_SINGLE_PAGE_SHADOW_MAPS <= VSM_PAGE_SIZE * VSM_PAGE_SIZE` | `VirtualShadowMaps/VirtualShadowMapArray.cpp:2647` |
| `static_assert` on enum/array parity | `EPass` vs `PassNames` | `PostProcess/PostProcessing.cpp:620` |
| `static_assert` on batch size vs buffer count | `IScreenSpaceDenoiser::kMaxBatchSize <= kMaxBufferProcessingCount` | `ScreenSpaceDenoise.cpp:145` |
| `static_assert` on type size | `sizeof(FLumenCardId) == sizeof(uint64)` | `Lumen/LumenSceneData.h:162` |
| `checkf` on lifecycle misuse | "Pipeline state ID used after release" | `MeshPassProcessor.h:655` |
| `checkf` on unregistered processor | "never registered" | `MeshPassProcessor.h:2417` |
| `checkf` on unimplemented branch | "Unknown culling type: %d" | `NaniteCullRaster.cpp:4496` |
| `ensureMsgf` on permutation absence | "Missing permutation for FScreenProbeCompositeTracesWithScatterCS" | `Lumen/LumenScreenProbeFiltering.cpp:465` |
| `ensureMsgf` on unsupported resolution | "Unsupported gather resolution" | `LumenScreenProbeFiltering.cpp:497` |
| `checkf` on alignment contract | "Viewport rect must be %d-pixel aligned" | `StochasticLighting/StochasticLighting.cpp:370` |
| `checkf` on frozen-data bounds | `GlobalFrozenDataBytes + NumLights * LightFrozenDataBytes <= FrozenLightData.Num()` | `MegaLights/MegaLightsVisualizeLightComplexity.cpp:491` |
| `ensure` on capacity overflow | `(NumSinglePageShadowMaps + Count) <= VSM_MAX_SINGLE_PAGE_SHADOW_MAPS` | `VirtualShadowMapArray.cpp:1092` |
| `ensure` on instance-run packing | `InstanceRuns.Num() + 2 <= InstanceRuns.Max()` | `LumenSceneCardCapture.cpp:910`, `WaterInfoTextureRendering.cpp:751` |
| `ensure` on geometry invariant | "Card has..." OBB orientation | `LumenSceneCardCapture.cpp:722` |
| `ensure` on aspect ratio | "Aspect ratio is too big for desired grid size" | `VT/AdaptiveVirtualTexture.cpp:279` |
| `checkSlow` on bit packing bounds | `NumItems - 1 < (1U << NumItemBits)` | `GPUWorkGroupLoadBalancer.h:41`-`:61` |
| `checkNoEntry` on unreachable mode | light complexity target | `MegaLightsVisualizeLightComplexity.cpp:140` |
| `#error` in HLSL | "NUM_THREADS_PER_GROUP must be defined, and must be equal to the thread group size of the caller" | `Engine/Shaders/Private/ThreadGroupPrefixSum.ush:24` |
| `check` on RHI capability range | VRS tile min width in `[8, 64]` | `VariableRateShading/FoveatedImageGenerator.cpp:200` |
| `check` on dispatch dimension | `GRHIMaxDispatchThreadGroupsPerDimension.X >= WrappedGroupStride` | `RenderGraphUtils.h:390` |
| `check` on render target count | `ColorTextures.Num() <= MaxSimultaneousRenderTargets` | `RenderGraphUtils.h:123`, `:160` |
| `check` on thread affinity | `check(IsInRenderingThread())` | `DecalRenderingShared.cpp:155`, `:166`; `DynamicRenderScaling.cpp:176` |

## Observability surfaces

| Surface | Where | Anchor |
|---|---|---|
| RDG event scopes and names | every pass | `RenderGraphDefinitions.h:75`-`:89` usage examples |
| Stats groups | `STATGROUP_RHI`, `STATGROUP_RDG` | `Core/Public/Stats/GlobalStats.inl:61`, `:62` |
| GPU stats | `Postprocessing`, `CustomRenderPasses` | `Renderer/Private/SceneRendering.h:200`, `:201` |
| RDG counters and memory watermark | pass counts, cycle stats | `RenderCore/Private/RenderGraphPrivate.h:164`-`:190` |
| Mesh draw command stats | `r.MeshDrawCommands.Stats` | `Engine/Public/MeshDrawCommandStatsSettings.h:60`, budgets `:62`, `:65` |
| VT CSV stats | `r.VT.CsvStats` | `Renderer/Private/VT/VirtualTextureSystem.cpp:140`-`:146` |
| Lumen stats | `r.Lumen.*Stats` | `Lumen/LumenSceneLighting.cpp:57` |
| Nanite material performance warnings | `r.Nanite.EmitMaterialPerformanceWarnings` | `Nanite/NaniteFeedback.cpp:15`, `:100`, `:169` |
| Critical path stall | `CriticalPathStall.AfterInitViews` | `DeferredShadingRenderer.cpp:1388`-`:1391`, applied `:2128`-`:2132` |
| Draw event visibility | `r.Nanite.ShowDrawEvents` | `NaniteShading.cpp:96` |

## Verifying a claim in this document set

Three rules, applied during the read audit:

1. Every `file:line` citation was opened with `read` or `grep` against `UE_5.8`; scout-reported line numbers were re-verified before being cited, because scouts were off by a few lines.
2. Claims that could not be observed in source are marked `[INFERENCE]`.
3. Graph facts come from the graphify graph (`graphify-out/graph.json`, scan root `UE_5.8`, CLI 0.9.80, `GRAPHIFY_MAX_GRAPH_BYTES=6GB`), not from reading files. Shader files are not graph nodes, so no graph claim about shader structure is possible.

Engine-wide `grep` times out at 30 seconds; every scope claim in these chapters is limited to the directories actually searched.

Mermaid blocks are validated with `bun` against mermaid 12.1.0 (`C:/Users/nguye/AppData/Local/Temp/rp_mermaid_check.mjs`, siblings `rdg_mermaid_check.mjs`, `rhi_mermaid_check.mjs`). Constraints that the checker enforces: edge labels containing `(` must be quoted; `stateDiagram-v2` labels reject `&gt;`; sequence-diagram message text rejects `;`; `participant Actor` is invalid because `actor` is reserved.

## What is not testable here

- No reference-image or golden-image harness exists in `Renderer/Private/Tests`; the two testbeds are functional, not visual.
- Denoiser quality, temporal stability, and visualizer output cannot be asserted from source; they require a rendered frame.
- Permutation coverage is only observable through `ShouldCompilePermutation` (`RenderCore/Public/Shader.h:848`) and `ModifyCompilationEnvironment` (no-op default `:842`), which run at shader-map build time, not at test time.
