# 06 — Base Pass, GBuffer and Shading Models

The base pass writes material attributes to a GBuffer; deferred lighting then evaluates a BRDF per pixel per light. Two implementations exist and are selected per platform: the clustered deferred path (`ClusteredDeferredShadingPixelShader.usf`) and the mobile path (`MobileBasePassPixelShader.usf` + `MobileDecodeGBuffer`). Forward shading is a third path where the base pass integrates lighting directly.

## GBuffer contract

`FGBufferData` (`Engine/Shaders/Private/DeferredShadingCommon.ush:379`) is the shader-side contract every shading model must fill:

| Field | Line | Notes |
|---|---|---|
| `WorldNormal` | `:382` | |
| `WorldTangent` | `:384` | valid only when `HAS_ANISOTROPY_MASK` is set (`ShadingCommon.ush:39`) |
| `DiffuseColor` | `:386` | shading-model dependent |
| `SpecularColor` | `:388` | |
| `BaseColor` | `:390` | |
| `Metallic` / `Specular` | `:392` / `:394` | |
| `CustomData` | `:396` | shading-model specific payload |
| `Roughness` | `:406` | |
| `GBufferAO` | `:410` | |
| `ShadingModelID` | `:412` | 4 low bits of an 8-bit channel |
| `SelectiveOutputMask` | `:414` | 4 high bits of the same channel |
| `PerObjectGBufferData` | `:417` | |
| `Velocity` | `:428` | |

The 4-bit split is the hard constraint: `SHADINGMODELID_MASK 0xF` (`ShadingCommon.ush:35`) means at most 16 shading models (`SHADINGMODELID_NUM 14`, `:34`), and the flags `HAS_ANISOTROPY_MASK` (`1<<4`), `SKIP_PRECSHADOW_MASK` (`1<<5`), `ZERO_PRECSHADOW_MASK` (`1<<6`), `IS_FIRST_PERSON_MASK` (`1<<6`, shared bit — only valid when `ALLOW_STATIC_LIGHTING` is 0), `SKIP_VELOCITY_MASK` (`1<<7`) (`:39-43`) live in the same byte. A new shading model that needs a flag bit cannot get one.

## Shading models

| ID | Value | Line |
|---|---|---|
| `UNLIT` | 0 | `ShadingCommon.ush:20` |
| `DEFAULT_LIT` | 1 | `:21` |
| `SUBSURFACE` | 2 | `:22` |
| `PREINTEGRATED_SKIN` | 3 | `:23` |
| `CLEAR_COAT` | 4 | `:24` |
| `SUBSURFACE_PROFILE` | 5 | `:25` |
| `TWOSIDED_FOLIAGE` | 6 | `:26` |
| `HAIR` | 7 | `:27` |
| `CLOTH` | 8 | `:28` |
| `EYE` | 9 | `:29` |
| `SINGLELAYERWATER` | 10 | `:30` |
| `THIN_TRANSLUCENT` | 11 | `:31` |
| `SUBSTRATE` | 12 | `:32` — "Temporary while we convert everything to Substrate" |
| `SUBSTRATE_TOON` | 13 | `:33` |

Dispatch is a compile-time-permutation switch, not a runtime branch: `GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM` is a macro specifically so `ScreenSpaceData.GBuffer.ShadingModelID = ShadingModelId` is a compile-time constant (`ClusteredDeferredShadingPixelShader.usf:289-302`), instantiated once per model (`:423-432`). `UNLIT` exits before any lighting (`:409`). Substrate replaces the per-model switch with a tile-based BSDF dispatch (see [11_Substrate.md](11_Substrate.md)); its BSDF type list reserves 7 for custom (`SubstrateDefinitions.h:33-40`).

## Packing

`EGBufferSlot` (`RenderCore/Public/GBufferInfo.h:12-43`) enumerates every attribute and its storage width — `GBS_SceneColor` 11.11.10, `GBS_WorldNormal` 10.10.10, `GBS_ShadingModelId` 4 bits, `GBS_Velocity` RG float16, `GBS_CustomData` RGBA8 uncompressed. `EGBufferCompression` (`:47-80`) and `EGBufferType` (`:83-95`) define how slots map to render targets; `EGBufferLayout` (`:106-112`) has exactly two layouts, `GBL_Default` and `GBL_ForceVelocity`. Base-pass shader permutations are generated per (light-map policy × layout) pair (`IMPLEMENT_BASEPASS_PIXELSHADER_TYPE` `BasePassRendering.cpp:186`, `TBasePassPS` template `:441-460`), with a 128-bit render-target special case only for `LMP_NO_LIGHTMAP` + `GBL_Default` (`:454`).

Encode/decode is paired and platform-specific: `EncodeGBuffer` (`DeferredShadingCommon.ush:968`) delegates to `MobileEncodeGBuffer` (`:592`) on mobile, and `DecodeGBufferData` (`:1076`) pairs with `MobileDecodeGBuffer` (`:717`). Any consumer that decodes with the wrong variant reads garbage — the mobile path packs into four `half4` targets, the desktop path uses six plus velocity (`:1246`, `:1337`).

## Clustered lighting

`ClusteredDeferredShadingPass.cpp` asserts the preconditions it depends on:

- `check(ShouldUseClusteredDeferredShading(ViewFamily.GetShaderPlatform()))` (`:339`) — the runtime gate requires SM6 **and** Virtual Shadow Map support (`:64`). The hair-strands custom vertex shader permutation allows SM5 (`:89`); the pixel shader predicate requires VSM support (`:146`) and SM6 (`:151`).
- `check(SortedLightsSet.ClusteredSupportedEnd > 0)` (`:199`) — a light set that is entirely unsupported produces an empty cluster.
- `check(SubstrateSceneData)` (`:228`, `:356`) — both are conditional, not unconditional: `:228` fires only when opaque rough refraction is enabled (`:226`), `:356` only when Substrate is enabled (`:353`).
- Event name carries the actual configuration: `RDG_EVENT_NAME("Light::ClusteredDeferredShading(%s,Lights:%d%s%s%s%s)", ...)` (`:246`) reports HairStrands, Substrate and light counts.

Light selection is per-view and per-cluster; `r.UseClusteredDeferredShading` is deprecated in favour of `r.ClusteredDeferredShading.EnableForProject` (`:36`, `:40`, `:49`), gated on feature level (`:64`).

## Light evaluation inputs

`LightRendering.cpp` holds the per-light-type ray-traced shadow switches: directional `:121`, point `:127`, spot `:133`, rect `:139`; denoiser `:97`; batch sizes `:106`, `:111`; contact-shadow length/intensity overrides `:158-180`; light function atlas parameters `:58` (`IMPLEMENT_TYPE_LAYOUT(FLightFunctionSharedParameters)`). Simple lights (`:77`) and depth-bounds test (`:70`) are separate gates.

Indirect lighting (`IndirectLightRendering.cpp`) declares **project-default** CVars, not scalability settings (`:89` "This is the project default dynamic global illumination, NOT the scalability setting", `:100` same for reflections): GI method `:92`, reflection method `:103`, half-res `:110`, rays per pixel `:115`, denoiser `:120`, SSR denoiser `:125`. The tracing viewport is asserted, not clamped — `ensure(... TracingViewportSize <= kMaxTracingResolution)` (`:969-970`) reports a violation without changing the dimensions. A light-map policy must have been found (`check(BestPolicyId != -1)` `:963`).

## Design constraints

1. **Shading model IDs are a 4-bit namespace.** Adding one beyond 15 requires widening the channel, which changes every GBuffer layout and every decode path.
2. **Permutations are generated per layout.** A material that compiles for `GBL_Default` does not automatically compile for `GBL_ForceVelocity` (`BasePassRendering.cpp:496` switch).
3. **`r.BasePassOutputsVelocityDebug` mode 2** (`:420`-`:421`) changes the render-state write mask (`:423` instead of `:427`), so velocity debug output is a blend-state change, not a shader permutation.
4. **Substrate scene data is required by the clustered pass only when Substrate or opaque rough refraction is enabled** (`:226`-`:228`, `:353`-`:356`). The clustered runtime gate itself requires SM6 plus Virtual Shadow Map support (`:64`).

## Tech artist notes

- Shading-model changes are permutation changes: `r.SupportAllShaderPermutations` (`:145`) and `r.SupportLowQualityLightmaps` (`:139`) change which permutations exist, which is why a material can compile in editor and fail at runtime.
- `r.ClearGBufferDBeforeBasePass` (`:157-160`) is the only way to guarantee GBuffer D is clean for models that do not write it; leaving it off means stale data from the previous frame.
- `r.SelectiveBasePassOutputs` (`:111`) is the debug channel that lets a material write to `SelectiveOutputMask` — the same byte as the shading model ID.

## Graph facts

- `PostProcess/PostProcessGBufferHints.cpp` is the highest-degree render-scope file at 14,890 edges — the GBuffer-hint visualization path is referenced from nearly every render-scope node, which is why the debug view modes live in the post-process directory.
- `Engine/Private/Materials/MaterialExpressions.cpp` (1,481 nodes) and `Engine/Public/MaterialCompiler.h` (722) are the material-side half of the shading model contract; `HLSLMaterialTranslator.h` (585) is the translation layer.
- Renderer-module node counts: PostProcess 1,519, Substrate 172, CompositionLighting 191, Shadows 187.
- Shader files are not graph nodes; the `FGBufferData` and `SHADINGMODELID_*` facts above come from direct source reads, not graph queries.
