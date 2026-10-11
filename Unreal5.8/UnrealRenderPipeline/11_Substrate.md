# 11 — Substrate

Substrate replaces "one shading model per material" with a per-tile material graph: each 8×8 pixel tile holds a tree of BSDF nodes and operators, evaluated once per tile. The consequence is that material cost becomes a tile-classification problem — the renderer decides per tile which path to run, and the tile type determines which shader permutation exists.

## Constants that define the representation

`Engine/Shaders/Shared/SubstrateDefinitions.h`:

| Constant | Value | Line | Consequence |
|---|---|---|---|
| `SUBSTRATE_TILE_SIZE` | 8 | `:44` | every Substrate resolution is rounded up to a multiple of 8 (`Substrate.cpp:202-203`, `:211`, `:480-481`) |
| `SUBSTRATE_MAX_CLOSURE_COUNT` | aliased to `..._FOR_CLOSUREOFFSET` | `:21` | comment at `:20`: "can only ever use ... for Lumen" — the closure budget is shared with Lumen |
| `SUBSTRATE_MAX_OPERATOR_COUNT` | 15 | `:22` | runtime tree size |
| `SUBSTRATE_COMPILATION_OUTPUT_MAX_OPERATOR` | 24 | `Engine/Source/Runtime/Engine/Public/Rendering/SubstrateMaterialShared.h:163` | compiler output can exceed the runtime limit; the array is sized to 24 (`:236`) |
| `SUBSTRATE_MAX_SHAREDLOCALBASES_REGISTERS` | 4 | `:25` | register budget for shared bases |
| `SUBSTRATE_FULLY_SIMPLIFIED_NUM_UINTS` | `32/4` | `:31` | the fast path packs a simplified tile into 32 bits |
| `SUBSTRATE_BASE_PASS_MRT_OUTPUT_COUNT` | 3 | `:55` | Substrate base pass writes 3 MRTs |
| `SUBSTRATE_LAYER_DEFAULT_THICKNESS_CM` | 0.01 | `:48` | default layer thickness |
| `SUBSTRATE_MIN_GGX_ROUGHNESS` / `SUBSTRATE_MIN_FUZZ_ROUGHNESS` | 0.02 | `:52` / `:53` | lower bounds; a material below them is clamped, not rejected |

BSDF types (`:33-40`): `SLAB` 0, `VOLUMETRICFOGCLOUD` 1, `UNLIT` 2, `HAIR` 3, `SINGLELAYERWATER` 4, `EYE` 5, `TOON` 6, 7 reserved for custom. Operators (`:59-65`): `WEIGHT` 0, `VERTICAL` 1, `HORIZONTAL` 2, `ADD` 3, `SELECT` 4, `BSDF` 5, `BSDF_LEGACY` 6.

## Tile types

`ESubstrateTileType` (`SubstrateMaterialShared.h:68-81`) maps to the shader defines: `ESimple` `:70`, `ESingle` `:71`, `EComplex` `:72`, `EComplexSpecial` `:73`, plus rough-refraction variants (`:75`, `:76`) and decal variants (`:77-79`). The base-pass permutation is `SHADER_PERMUTATION_INT("SUBSTRATE_TILETYPE", 4)` (`Substrate.h:320`) — four permutations for the four base types, and the special paths get their own passes.

Tile-type queries are per view: `GetSubstrateUsesTileType` (`Substrate.cpp:223`, with `check(TileType <= ESubstrateTileType::EComplexSpecial)` `:228`), `GetSubstrateUsesComplexSpecialPath` (`:237`), `GetSubstrateUsesAnisotropy` (`:248`). A material that needs the complex-special path forces that permutation to exist for the whole view.

## Frame setup

`InitialiseSubstrateFrameSceneData` (`Substrate.cpp:460`) is called from `DeferredShadingRenderer.cpp:2401`, immediately after `EndInitViews` — the comment at `:2399` states it must run after `EndInitViews()` to ensure view relevance computation is complete. Substrate is not a scene extension; `FSubstrateSceneData` (`Substrate.h:93`) and `FSubstrateViewData` (`:153`) are frame/view state, and `check(SceneData.MaterialTextureArray != nullptr)` (`Substrate.cpp:291`) with the comment "Sanity check: the scene data should already exist" (`:288`) means the scene data must be initialized before any view pass runs.

Indirect dispatch buffers are created per view (`:356-358`): closure-tile per-thread, closure-tile, and closure-tile ray-tracing variants, sized by `GetMaxDownsampleFactor()` (`:137`), with `check(InDownsampleFactor <= GetMaxDownsampleFactor())` (`:149`) and `check(TotalTileCount > 0)` (`:337`).

Material classification is its own pass: `AddSubstrateMaterialClassificationPass` (`Substrate.h:261`), event scope `Substrate` (`DeferredShadingRenderer.cpp:3171`).

## Stochastic lighting coupling

Substrate and stochastic lighting share the tile classification stage (`StochasticLighting/StochasticLighting.cpp`):

- `check(MaterialSource == EMaterialSource::GBuffer)` (`:321`), `check(DepthHistoryUAV && NormalHistoryUAV)` (`:326`), `check(ClosureHistoryUAV)` (`:329`), `check(DownsampledSceneDepth2x1UAV && DownsampledWorldNormal2x1UAV)` (`:335`), `check(DownsampledSceneDepth2x2UAV && DownsampledWorldNormal2x2UAV)` (`:340`), `check(LumenTileBitmaskUAV)` (`:345`), `check(MegaLightsTileBitmaskUAV)` (`:350`).
- `checkf((View.ViewRect.Min.X % TileSize) == 0 && (View.ViewRect.Min.Y % TileSize) == 0, TEXT("Viewport rect must be %d-pixel aligned"))` (`:370`) — a view rect that is not tile-aligned breaks classification, which is why odd resolutions and some XR viewports fail.
- Overflow handling is a separate pass (`TileClassificationMark(Overflow)` `:569`), and `check(!RunConfig.bSubstrateOverflow)` (`:603`) guards the non-overflow path.
- Async compute mode is clamped 0–2 (`:907`); `r.StochasticLighting.FixedStateFrameIndex` (`:14`) makes the state deterministic when ≥ 0 (`:36`, `:46`).
- `check(Desc.Extent == RenderTarget->Desc.Extent)` (`:1131`) — the classification output must match the render target exactly.

Event scope: `StochasticLighting` (`DeferredShadingRenderer.cpp:3202`), gated by `CVarSubstrateStochasticLightingActive` (`Substrate.cpp:73`, `:157`) and `CVarSubstrateUseStochasticLightingClassification` (`:85`).

## Rough refraction

Opaque rough refraction is a separate pipeline (`SubstrateRoughRefraction.cpp`): blur scale `:18`, horizontal blur `:143`, vertical blur `:201`, SSS `:260`, lobe evaluation `:482`, data pass `:537`, GPU readback `:549`, event scope `Substrate::OpaqueRoughRefraction` (`:90`). Shader-print is required for the visualization path: `check(ShaderPrint::IsEnabled(View.ShaderPrintData))` — "One must enable ShaderPrint beforehand using r.ShaderPrint=1" (`:446`).

## Design constraints

1. **Tile size is fixed at 8.** Every Substrate resolution is rounded up (`:480-481`), so a view whose resolution is not a multiple of 8 gets a larger buffer than the view rect.
2. **Closure budget is shared with Lumen** (`SubstrateDefinitions.h:20-21`); raising it changes Lumen's memory layout.
3. **Compiler output limit (24) exceeds the runtime limit (15)** (`SubstrateMaterialShared.h:163` vs `SubstrateDefinitions.h:22`) — a material that compiles can still be rejected at runtime.
4. **Allocation mode is clamped 0–2** (`Substrate.cpp:167`), and tile coordinates can be 8-bit (`CVarSubstrateTileCoord8Bits` `:67`), which caps the tile grid.
5. **Stencil pass stage is clamped 0–2** (`:127`, enum `EStencilPassStage` `Substrate.h:222`), so DBuffer decals and the base pass have a fixed ordering choice.
6. **Substrate scene data is required by the clustered shading pass when Substrate or opaque rough refraction is enabled** (`ClusteredDeferredShadingPass.cpp:226`-`:228`, `:353`-`:356`); the checks are conditional, so a non-Substrate material on a clustered platform does not need the scene data.

## Tech artist notes

- `r.Substrate.DebugPeelLayersAboveDepth` (`:37`) and `r.Substrate.DebugRoughnessTracking` (`:43`) are the layer-debug surfaces; the visualization passes are `Substrate::VisualizeMaterial(Print)` (`SubstrateVisualize.cpp:1059`), `(Display)` (`:1090`), `Substrate::VisualizeSystemInfo` (`:1129`), `Substrate::SubstrateAdvancedVisualization(Print)` (`:1469`), `FMaterialDebugSubstrateTreePS` (`:1494`), all guarded by `check(IsSubstrateEnabled())` (`:1535`).
- `r.Substrate.UseClosureCountFromMaterial` (`:31`) changes the buffer size per view, so a single heavy material can increase memory for the whole view.
- `r.Substrate.ClearMaterialBuffer` (`:79`) and `r.Substrate.UseCmaskClear` (`:25`) are correctness switches, not optimizations: without them, stale tile data from the previous frame is read.

## Graph facts

- Substrate subsystem: 172 nodes in the Renderer module. Top symbols: `Substrate.cpp` 69, `Substrate.h` 43, `InitialiseSubstrateFrameSceneData()` 34, `SubstrateVisualize.cpp` 32, `AddSubstrateMaterialClassificationPass()` 21, `FGlintShadingLUTsStateData` 19.
- Substrate reaches Lumen through the shared closure budget rather than through a call: `FLumenSceneData` (161 edges) and `Substrate.cpp` (69) are in different communities (18 and 37), and the coupling is the `SUBSTRATE_MAX_CLOSURE_COUNT` define, which is not a graph edge.
- 25 Substrate shader files exist (`Engine/Shaders/Private/Substrate/`); shader files are not graph nodes, so the tile-type permutation space is invisible to graph queries.
