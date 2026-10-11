# 13 — File Reference

Paths relative to `G:/Epic/UE_5.8/Engine/`.

| File | Responsibility | Key symbols |
|---|---|---|
| `Source/Runtime/RenderCore/Public/Shader.h` | Shader instance base, type system, registration macros | `FShader` 816; virtuals 842-858; `SHADER_TYPE_LIST` 1219-1226; `FShaderType` 1234; `EShaderTypeForDynamicCast` 1238-1242; lookups 1270-1279; `FShaderTypeRegistration` 1600-1612; `DECLARE_SHADER_TYPE` 1706; `IMPLEMENT_SHADER_TYPE` 1744-1764; `IMPLEMENT_SHADER_TYPE_CONSTRUCTOR` 1862, 1918 |
| `Source/Runtime/RenderCore/Private/Shader.cpp` | Type registration, key generation, compile CVars | CVars 89-119; `FShaderType` ctor 290-295; `FShaderTypeRegistration` ctor 315; `CommitAll` 328-336; `ShaderMapAppendKey` 1711, 1723-1732; key tokens 2749-2778 |
| `Source/Runtime/RenderCore/Public/GlobalShader.h` | Global shader family | `IMPLEMENT_SHADER_TYPE_CONSTRUCTOR` 93; `DECLARE_GLOBAL_SHADER` 431; `DECLARE_EXPORTED_GLOBAL_SHADER` 432; `IMPLEMENT_GLOBAL_SHADER` 433 |
| `Source/Runtime/RenderCore/Private/ShaderCore.cpp` | Virtual shader directories, debug CVars, compile stats | `STATGROUP_ShaderCompiling` 102-109; `ResetAllShaderSourceDirectoryMappings` 4213; `AddShaderSourceDirectoryMapping` 4218-4231; debug CVars 212-236; `r.ForceDebugViewModes` 570-574; `PreprocessShaderInternal` 1816 |
| `Source/Runtime/RenderCore/Private/ShaderDiagnostics.cpp` | Warning/development-mode CVars | 8-12, 14-18, 20-24 |
| `Source/Runtime/RenderCore/Public/ShaderParameterStruct.h` | Parameter struct macros | 58, 62, 69, 75 |
| `Source/Runtime/RenderCore/Public/ShaderParameterMacros.h` | Parameter struct begin/end and resource parameter types | `BEGIN_SHADER_PARAMETER_STRUCT` 1411, `END_SHADER_PARAMETER_STRUCT` 1414, `SHADER_PARAMETER_RDG_TEXTURE` 1697, `SHADER_PARAMETER_RDG_TEXTURE_UAV` 1733 |
| `Source/Runtime/RenderCore/Public/ShaderSource.h` | Source buffer with SIMD padding | `FShaderSource` 19, `SetLen` 22-27, deprecated macros 12-14 |
| `Source/Runtime/RenderCore/Public/ShaderCompilerJobTypes.h` | Compile job types | `FShaderCompileJob` 402, `Key` 407, `ShaderParameters` 413 |
| `Source/Runtime/RenderCore/Private/ShaderResource.cpp` | Shader map resource assembly | `FShaderMapResourceCode::AddShaderCompilerOutput` 262 |
| `Source/Runtime/RenderCore/Internal/ShaderLibrary/EditorShaderCodeArchive.h` | Cooked archive format | `SaveToDisk` doc 156-162 |
| `Source/Runtime/RenderCore/Public/ShaderStableKeyDebugInfoReader.h` | `.shk` stable key reader | `FShaderStableKeyDebugInfoReader` 17-26 |
| `Source/Runtime/RenderCore/Private/ShaderLibrary/ShaderCodeArchive.cpp` | Archive load CVars | 45-51, 53-59, 61-67, 69-76 |
| `Source/Runtime/RenderCore/Private/ShaderLibrary/ShaderCodeLibraryUtilities.cpp` | Load behaviour | 50-56, 58-64 |
| `Source/Runtime/RenderCore/Private/ShaderBundles.cpp` | Shader bundles | `r.ShaderBundle.MaxSize` 8-12; `FDispatchShaderBundleCS` 15; gates 17-20, 22-25 |
| `Source/Runtime/RenderCore/Private/ShaderPipelineCache.cpp` | PSO precache | CVars 55-126 |
| `Source/Runtime/RenderCore/Public/ShaderPipelineCache.h` | Pipeline cache contract | 34-38, 54-58 |
| `Source/Runtime/RHI/Private/PipelineFileCache.cpp` | `r.ShaderPipelineCache.Enabled` | 112-116 |
| `Source/Runtime/RenderCore/Public/RenderGraphDefinitions.h` | Pass flags | `ERDGPassFlags` 129-160 |
| `Source/Runtime/RenderCore/Public/RenderGraphBuilder.h` | Pass registration | `RegisterExternalTexture` 79-87, `CreateBuffer` 110, `AddPass` 221/225/233 |
| `Source/Runtime/RenderCore/Public/RenderGraphUtils.h` | Compute dispatch helpers | `FComputeShaderUtils` 333, `kGolden2DGroupSize` 336, `GetGroupCount` 339 |
| `Source/Runtime/RenderCore/Private/RenderGraphPrivate.cpp` | RDG debug CVars | 18-373 |
| `Source/Runtime/RenderCore/Public/RenderGraphEvent.h` | Profiling scopes | `FRDGScope_RHI::BeginCPU` 166-176, `RDG_EVENT_*` 477-484 |
| `Source/Runtime/RenderCore/Public/VertexFactory.h` | Vertex factory types | `IMPLEMENT_VERTEX_FACTORY_PARAMETER_TYPE` 217, `FVertexFactoryType` 314, `IMPLEMENT_VERTEX_FACTORY_TYPE` 572 |
| `Source/Runtime/RenderCore/Public/Animation/MeshDeformerProvider.h` | Deformer modular feature | `IMeshDeformerProvider` 17, note 12-16, name 22, `IsSupported` 27, `FDefaultMeshDeformerSetup` 30-34, `GetDefaultMeshDeformer` 36-38 |
| `Source/Runtime/RenderCore/Public/IRenderCaptureProvider.h` | Capture modular feature | 10, 18-22, 29-34 |
| `Source/Runtime/RenderCore/Public/DumpGPU.h` | Frame dump API | 28, 31, 34, 36-63 |
| `Source/Runtime/RenderCore/Private/GPUDebugCrashUtils.cpp` | Crash-time compute shader | 36, 39-44 |
| `Source/Runtime/RenderCore/Private/VisualizeTexture.cpp` | Visualize texture | `r.VisualizeTexture.AllowBlinking` 602-605 |
| `Source/Runtime/RenderCore/Private/RenderUtils.cpp` | Substrate settings interface | `CVarSubstrate` 2068-2072; CVars 2084-2217; `namespace Substrate` 2219-2581; `IsSubstrateEnabled` 2221-2224; `IsSubstrateBlendableGBufferEnabled` 2226-2233; `IsStochasticLightingEnabled` 2235-2238; `GetBytePerPixel` 2240-2262 |
| `Source/Runtime/RenderCore/Private/GBufferInfo.cpp` | GBuffer target mapping | Substrate UAV path 265-276 |
| `Source/Runtime/Engine/Public/MaterialDomain.h` | Material domains | `EMaterialDomain` 12-30 |
| `Source/Runtime/Engine/Public/MaterialShared.h` | Custom output gathering | 2941, 2944 |
| `Source/Runtime/Engine/Public/MaterialShaderType.h` | Material shader family | `IMPLEMENT_MATERIAL_SHADER_TYPE` 13-20; `FMaterialShaderTypes` 192-207, `AddShaderType` 200 |
| `Source/Runtime/Engine/Public/Materials/MaterialExpressionCustomOutput.h` | Custom output base | 13-14, 21, 23, 24, 29 |
| `Source/Runtime/Engine/Public/Materials/MaterialExpressionCustom.h` | Custom node output types | `ECustomMaterialOutputType` 16-24 |
| `Source/Runtime/Engine/Public/Materials/Material.h` | Usage flags (deprecated) | 701-703, 861-863, 1160-1162 |
| `Source/Runtime/Engine/Private/Materials/MaterialShared.cpp` | Translation dispatch | 108-123, 3541-3555, 3696-3703 |
| `Source/Runtime/Engine/Private/Materials/HLSLMaterialTranslator.cpp` | Translator | 90-94, 173-178, 180-184, 186-190, `TranslateMaterial` 1324 |
| `Source/Runtime/Engine/Private/Materials/MaterialExpressions.cpp` | Substrate topology error | 2461 |
| `Source/Runtime/Engine/Private/ShaderCompiler/ShaderCompiler.cpp` | Compiler, debug CVars | `GRetryShaderCompilation` 321; 132-136, 138-143, 323-334, 337-341 |
| `Source/Runtime/Engine/Private/ShaderCompiler/ShaderCompilerJobCache.cpp` | Per-shader DDC | 51-55, 57-62 |
| `Source/Runtime/Engine/Private/GPUSkinCache.cpp` | Skin cache | `STATGROUP_GPUSkinCache` 39-45; exec 49-76; CVars 78-220 |
| `Source/Runtime/Engine/Public/GPUSkinCache.h` | Skin cache contract | 9-21, `DynamicMeshBoundsModifyCompilationEnvironment` 114 |
| `Source/Runtime/Engine/Public/DebugViewModeInterface.h` | Debug view mode interface | 27, 72, 78, 81, 86 |
| `Source/Runtime/Engine/Private/DebugViewModeHelpers.cpp` | Debug view mode dispatch | 89, 345 |
| `Source/Runtime/Engine/Public/SceneViewExtension.h` | Plugin render hooks | `ISceneViewExtension` 117; hooks 175-222; `FSceneViewExtensions::NewExtension` 328-334 |
| `Source/Runtime/Engine/Public/Rendering/CustomRenderPass.h` | Custom pass base | `FCustomRenderPassBase` 49, hooks 143-146 |
| `Source/Runtime/Engine/Public/SceneInterface.h` | Custom pass input | 809, 812-813 |
| `Source/Runtime/Engine/Classes/Engine/EngineTypes.h` | Substrate enums | `ESubstrateStorageFormat` 780-791, `ESubstrateClosureConfig` 796-807, `ESubstrateShadingModel` 814-836 |
| `Source/Runtime/Engine/Classes/Engine/RendererSettings.h` | Substrate and skin cache settings | `bEnableSubstrate` 1254-1261, storage format 1263-1265, skin cache 1534-1543 |
| `Source/Runtime/Engine/Classes/Engine/BlendableInterface.h` | Blendable locations | `EBlendableLocation` 14-83, `IBlendableInterface` 95-101 |
| `Source/Runtime/Engine/Classes/Engine/Scene.h` | Post process settings | `WeightedBlendables` 2574-2575 |
| `Source/Runtime/Engine/Classes/Components/PostProcessComponent.h` | Post process component | 23-53 |
| `Source/Runtime/Renderer/Private/SceneExtensions.h` | Scene extension interfaces | `ISceneExtension` 23-41, `ISceneExtensionUpdater` 45-65, `ISceneExtensionRenderer` 70-111, `FSceneExtensionRegistry` 132-140, macros 403-412 |
| `Source/Runtime/Renderer/Public/MaterialShader.h` | Material shader base | `FMaterialShader` 55 |
| `Source/Runtime/Renderer/Public/MeshMaterialShader.h` | Mesh material shader base | `FMeshMaterialShader` 67, `GetElementShaderBindings` 102 |
| `Source/Runtime/Renderer/Private/PostProcess/PostProcessMaterial.cpp` | Post process material shaders | 41-45, 49-55, 57-61, 244-282, 342-343, 345-362 |
| `Source/Runtime/Renderer/Public/PostProcess/PostProcessMaterialInputs.h` | Post process inputs | 49-85 |
| `Source/Runtime/Renderer/Private/PostProcess/PostProcessing.h` | Post process pass API | `FPostProcessVS` 47, `AddPostProcessingPasses` 49-63 |
| `Source/Runtime/Renderer/Private/PostProcess/PostProcessing.cpp` | Subscription call, post process CVars | 189-199, 830-851 |
| `Source/Runtime/Renderer/Private/PostProcess/PostProcessAA.cpp` | FXAA quality | 10-14 |
| `Source/Runtime/Renderer/Private/ScreenSpaceRayTracing.cpp` | SSR quality | 44-46 |
| `Source/Runtime/Renderer/Private/MeshDrawCommands.cpp` | On-demand shader creation | 45-52 |
| `Source/Runtime/RHI/Public/RHICommandList.h` | Dispatch | 2699, 2709 |
| `Source/Runtime/RHI/Private/RHICommandList.cpp` | `r.RHICmdBypass` | 56-60 |
| `Source/Runtime/RHI/Private/GPUProfiler.cpp` | GPU profiler | CVars 35-116, `ProfileGPU` command 2197-2205 |
| `Source/Runtime/RHI/Public/RHIDefinitions.h` | Profiler gates | `RHI_NEW_GPU_PROFILER` 71, `HAS_GPU_STATS` 73-75 |
| `Source/Runtime/RHI/Public/GpuProfilerTrace.h` | Trace gate | 12-14 |
| `Source/Runtime/RHI/Public/DataDrivenShaderPlatformInfo.h` | Platform capability flags | `GetSupportsAdaptiveGBuffer` (used at `RenderUtils.cpp:2228`) |
| `Source/Runtime/Core/Public/Misc/Build.h` | Build gates | `WITH_PROFILEGPU` 410-411, `ALLOW_DUMPGPU_IN_TEST` 413-415, `ALLOW_DUMPGPU_IN_SHIPPING` 417-419 |
| `Source/Runtime/Core/Private/HAL/ConsoleManager.cpp` | `DumpGPU` command, `r.BloomQuality` | 3813-3815, 4081-4085 |
| `Source/Runtime/Engine/Private/SceneView.cpp` | Auto exposure defaults | 182-186, 189-193, 196-199, 201-205 |
| `Source/Editor/UnrealEd/Private/ShaderStatsCollector.cpp` | Editor compile stats | 9-20 |
| `Source/Developer/TargetPlatform/Public/Interfaces/IShaderFormat.h` | Shader format interface | `IShaderFormat` 22; resolved via `FindShaderFormat` (`ShaderCompiler.cpp:3819`, `4346`) |
| `Source/Developer/ShaderCompilerCommon/` | Shader format plumbing: ISAParser, SPIR-V, SDCE, minifier, cross-compiler, symbol export | `ShaderCompilerCommon.cpp`, `ISAParser/RDNA_Common.h` |
| `Source/Developer/MaterialBaking/Private/Tests/ExportMaterialProxyTest.cpp` | Only shader-adjacent automation test | 24-27 |
| `Source/Programs/` | No `ShaderCompileWorker` source in this install; binary at `Engine/Binaries/Win64/ShaderCompileWorker.exe` | — |
| `Shaders/Private/` | 502 entries of HLSL source | — |
| `Shaders/Shared/SubstrateDefinitions.h` | BSDF ids, tile size, packing | 31, 33-40, 41, 44 |
| `Plugins/Developer/ShaderToolkit/Source/ShaderAuditCore/Public/IShaderAuditExtension.h` | Editor audit extension point | 42 |
| `Plugins/FX/Niagara/Source/NiagaraShader/Private/NiagaraShaderModule.cpp` | Plugin shader directory + compile queue delegate | 13-21, 23-28 |
| `Plugins/FX/Niagara/Source/Niagara/Private/NiagaraGpuComputeDispatch.cpp` | Compute dispatch | `DispatchStage` 1475 |
