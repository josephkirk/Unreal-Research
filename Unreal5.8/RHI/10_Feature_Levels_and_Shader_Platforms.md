# Feature Levels and Shader Platforms — UE 5.8 RHI

## Scope

Two different concepts that are often conflated: the feature level (a capability tier the renderer branches on) and the shader platform (the compile target that decides permutations and binding layout).

## Feature levels

`ERHIFeatureLevel::Type` (`RHI/Public/RHIFeatureLevel.h:15`): `ES2_REMOVED`, `ES3_1`, `SM4_REMOVED`, `SM5`, `SM6`, `Num`. `ES2_REMOVED` and `SM4_REMOVED` are tombstones — code still branches on them for legacy paths.

Feature level is stored per render resource (`RenderCore/Public/RenderResource.h:207`, `SetFeatureLevel` `L154`, `HasValidFeatureLevel` `L156`) and can change at runtime: `FRenderResource::ChangeFeatureLevel` (`L56`) reinitializes every initialized resource. That is why switching a project from SM5 to SM6 rebuilds static resources.

## Shader platforms

`EShaderPlatform` (`RHI/Public/RHIShaderPlatform.h:10`) — values are stable IDs, not sequential (`SP_PCD3D_SM5 = 0` `L12`, `SP_METAL_ES3_1_IOS = 11`, `SP_PCD3D_ES3_1 = 14`, `SP_VULKAN_SM5 = 20`, then dynamic entries `SP_VULKAN_SM5_ANDROID`, `SP_PCD3D_SM6`, `SP_VULKAN_SM6`, `SP_METAL_SM6` up to `SP_NumPlatforms`). The header warns that `LegacyShaderPlatform` must be updated when this changes (`L9`).

`FStaticShaderPlatform` (`RHIShaderPlatform.h:60`) is the compile-time-safe wrapper used by data-driven queries.

Mapping between the two is runtime, not static: `GetFeatureLevelShaderPlatform` (`RHI/Public/RHIGlobals.h:1016`) reads `GRHIGlobals.ShaderPlatformForFeatureLevel`; `GetFeatureLevelShaderPlatform_Checked` (`L1021`) asserts you are not inside a cook commandlet (`L1023`) — cooking for platform A while running on platform B is a real source of wrong permutations.

## Data-driven platform info

`FDataDrivenShaderPlatformInfo` (`RHI/Public/DataDrivenShaderPlatformInfo.h`) is the authoritative capability table, initialized from `.uplugin`/`.ini` data at `FGenericDataDrivenShaderPlatformInfo::Initialize()` (`RHI/Private/DynamicRHI.cpp:300`).

| Query | Line |
|---|---|
| `IsValid(Platform)` | 910 |
| `IsPCPlatform(Platform)` | 954 |
| `IsMobilePlatform(Platform)` | 960 (max feature level == `ES3_1`) |
| `GetMaxSupportedFeatureLevel(Platform)` | 1052 |
| `IsFeatureLevelSupported(Platform, FL)` | 1076 |
| `RHIGetStaticShaderBindingLayoutSupport(Platform)` | 1299 |

Capability bits that change what shaders can do: `bSupportsAsyncPipelineCompilation` (`L100`), `bSupportsVertexShaderSRVs` (`L101`, accessor `L720`), `bSupportsVertexShaderUAVs` (`L102`, needs runtime `GRHIGlobals.SupportsVertexShaderUAVs`, accessor `L723`), `bSupportsTypedBufferSRVs` (`L103`), `bSupportsManualVertexFetch` (`L104`, accessor `L738`), `bRequiresReverseCullingOnMobile` (`L105`), `bOverrideFMaterial_NeedsGBufferEnabled` (`L106`), `bSupportsFFTBloom` (`L107`). Static shader binding layout section lookup: `Private/DataDrivenShaderPlatformInfo.cpp:120`.

`UseGPUScene(ShaderPlatform)` gates the GPUScene path (`Renderer/Public/MeshPassProcessor.h:2076` `bUseGPUScene`, used at `L2108`); when false, primitive data comes from a CPU-built uniform buffer instead of the scene buffer.

## Vertex factories and permutations

`FVertexFactoryType` (`RenderCore/Public/VertexFactory.h:314`) declares the vertex layout, stream bindings, permutation set (`ShouldCompilePermutation` `L566`), and the shader-parameter vtable. `FVertexFactoryShaderParameters` (`L172`) is the interface that binds per-draw parameters and vertex streams (`L187-191`, `L210-214`, macro form `L228-232`).

`FVertexFactory` (`L655`) is an `FRenderResource`; `GetPrimitiveIdStreamIndex` (`L744`) is the hook that decides whether the primitive id comes from a vertex stream, and `SupportsPrimitiveIdStream()` (`L751`) gates instancing. This is the link between content-level instancing and GPUScene: with manual vertex fetch there is no stream, and the primitive id is read from the scene buffer.

`FVertexFactoryTypeDependency` (`L593`) serializes the VF name and source hash (`L605`) — the shader cache key includes the VF, so a VF change invalidates permutations.

## Practical consequences

- A material is compiled per shader platform, not per feature level. Two platforms that share SM5 capability can still have different permutations because binding layout and vertex-fetch support differ.
- `bSupportsManualVertexFetch` false means vertex streams must exist; GPUScene instancing degrades to per-instance stream data.
- Reverse culling on mobile (`bRequiresReverseCullingOnMobile` `L105`) is a real geometry change, not a flag: `FMeshDrawCommandPassSetupTaskContext::bReverseCulling` (`Renderer/Private/MeshDrawCommands.h:66`) carries it into the pass.
