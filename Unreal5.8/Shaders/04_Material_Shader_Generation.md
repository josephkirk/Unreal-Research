# 04 — Material Shader Generation

A material is not a shader; it is a graph that the translator turns into HLSL plus a parameter metadata table. The permutation identity is `material × shader type × vertex factory × platform × usage flags`.

## Material domains

`UENUM EMaterialDomain : int` (`Runtime/Engine/Public/MaterialDomain.h:12-30`):

| Domain | Line | Consumer |
|---|---|---|
| `MD_Surface` | 15 | Base pass, `FMeshMaterialShader` |
| `MD_DeferredDecal` | 17 | Decal pass |
| `MD_LightFunction` | 19 | Light function shaders |
| `MD_Volume` | 21 | Volumetric fog / media |
| `MD_PostProcess` | 23 | Post process material shaders — see [12](12_Post_Process_Shaders_and_Control.md) |
| `MD_UI` | 25 | Editor and runtime UI |
| `MD_RuntimeVirtualTexture` | 27 (`Hidden`) | Runtime virtual texture painter (deprecated) |
| `MD_MAX` | 29 | Sentinel |

Domain selection changes which shader types the material is compiled for. A `MD_PostProcess` material is never compiled as a surface shader.

## Translation path

`FMaterial::Translate` branches between two implementations (`Runtime/Engine/Private/Materials/MaterialShared.cpp:3696-3703`): the new generator is used when `InShaderMapId.bUsingNewHLSLGenerator && !bEnableRestrictiveMaterialMode`. Note this is **not** gated by `FMaterialIRModule::IsEnabled` — the CVar that selects it is `r.Material.Translator.EnableNew`, default **false** (`MaterialShared.cpp:108-113`), read through `IsUsingNewMaterialTranslatorPrototype()` (`120-123`). `r.Material.UseShaderCompilationParameters` is 1 and `ReadOnly` (`114-118`).

The legacy path is `Translate_Legacy` (`3541-3555`): it constructs `FHLSLMaterialTranslator` with `&ShaderMapId.SubstrateCompilationConfig` (`3548`) and can return `EHLSLMaterialTranslatorResult::RetryWithoutDDC` (`3552`) — a translation failure is not always fatal, it can re-run without cache.

Translator CVars (`Runtime/Engine/Private/Materials/HLSLMaterialTranslator.cpp`):

| CVar | Default | Line |
|---|---|---|
| `r.Material.EnableTranslationLogFile` | false | 90-94 — writes `ShaderDebugInfo/MaterialTranslationLog-X.csv` |
| `r.MaterialEditor.LWCEnabled` | true | 173-178 |
| `r.Material.PedanticErrorChecksEnabled` | false | 180-184 |
| `r.Material.CullIntermediateUniformExpressions` | true | 186-190 |
| `TranslateMaterial()` | — | 1324 |

The translation log is the cheapest way for a tech artist to see what the graph produced: enable the CVar, recompile the material, read the CSV.

## Custom outputs and custom interpolators

Both gather functions are editor-only (`WITH_EDITORONLY_DATA`): `GatherCustomOutputExpressions` (`Runtime/Engine/Public/MaterialShared.h:2941`) and `GatherExpressionsForCustomInterpolators` (`2944`). The abstract base is `MaterialExpressionCustomOutput` (`Runtime/Engine/Public/Materials/MaterialExpressionCustomOutput.h:13-14`) with `GetNumOutputs` (21), `GetMaxOutputs` (23), `GetFunctionName` (PURE_VIRTUAL, 24), `HasCustomSourceOutput` (29). A plugin adds a custom output by subclassing this UObject — that is a data-driven extension, no shader code required.

`MaterialExpressionCustom` (the Custom HLSL node) is limited to `ECustomMaterialOutputType`: `CMOT_Float1`…`CMOT_Float4`, `CMOT_MaterialAttributes` (`Runtime/Engine/Public/Materials/MaterialExpressionCustom.h:16-24`). It cannot emit engine-internal types (see the Substrate consequence in [09](09_Substrate.md)).

## Usage flags

Usage flags are deprecated in 5.8 in favour of `GetUsageByFlag()`: `bUsedWithSkeletalMesh` (`Runtime/Engine/Public/Materials/Material.h:701-703`), `bUsedWithNanite` (`861-863`), `bUsedWithNeuralNetworks` (`1160-1162`). They still gate permutation compilation — a material used with a feature whose usage flag is off fails at bind time, not at compile time.

## Parameter binding

`SHADER_ROOT_PARAMETERS_SIGNATURE` (`RenderCore/Public/ShaderParameterStruct.h:58`), `SHADER_USE_PARAMETER_STRUCT` (`62`), `SHADER_USE_PARAMETER_STRUCT_WITH_LEGACY_BASE` (`69` — disables the bind-everything validation for the legacy `FMaterialShader`/`FMeshMaterialShader` bases), `SHADER_USE_PARAMETER_STRUCT_MIXED` (`75`). Resource parameters come from `ShaderParameterMacros.h` (e.g. `SHADER_PARAMETER_RDG_TEXTURE` at `1697`).

## Tech artist vs code

| Action | Surface |
|---|---|
| Choose domain, shading model, usage flags | Material asset |
| Add a Custom output node | `MaterialExpressionCustomOutput` subclass (C++, UObject) |
| Add a Custom HLSL node | `MaterialExpressionCustom`, float outputs only |
| Add a vertex factory | `IMPLEMENT_VERTEX_FACTORY_TYPE` (`VertexFactory.h:572`) + `IMPLEMENT_VERTEX_FACTORY_PARAMETER_TYPE` (`217`) |
| Force a permutation to exist | `ShouldCompilePermutation` in a material shader type (C++) |
| Inspect generated HLSL | `r.Material.EnableTranslationLogFile`, `r.DumpShaderDebugInfo` |
