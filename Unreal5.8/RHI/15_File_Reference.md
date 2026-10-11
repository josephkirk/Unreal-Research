# File Reference — UE 5.8 RHI

## Scope

Inventory of the files the rest of this set cites, with what each is responsible for. Paths relative to `UE_5.8/Engine/Source`.

## Runtime/RHI/Public

| File | Responsibility |
|---|---|
| `RHI.h` | umbrella include |
| `RHIFwd.h` | forward declarations (`L44-94`), `RHI_WANT_RESOURCE_INFO` (`L8`), `RHI_ENABLE_RESOURCE_INFO` (`L17`) |
| `RHIDefinitions.h` | enums and flags: `EBufferUsageFlags` (`L885-970`), `ETextureCreateFlags` (`L1087`), `EUniformBufferBindingFlags` (`L676`), `EUniformBufferValidation` (`L527`) |
| `RHIAccess.h` | `ERHIAccess` (`L10-85`) and access predicates (`L88-120`) |
| `RHITransition.h` | `FRHISubresourceRange` (`L20-101`), `FRHITransitionInfo` (`L118-226`), `FRHICommitResourceInfo` (`L112`) |
| `RHIDescriptorRange.h` | `FRHIDescriptorRangeDesc` (`L45`), `FRHIDescriptorRange` (`L103`), capability notes (`L15`, `L26`) |
| `RHIResources.h` | resource hierarchy (`FRHIResource` `L53` … `FRHITexture` `L2167`), `FGraphicsPipelineStateInitializer` (`L4655`), `FRHIRenderPassInfo` (`L5346`), `ERenderTargetActions` (`L5229`), `ESubpassHint` (`L4609`), `FRHIShaderBundle` (`L3990`) |
| `RHICommandList.h` | command classes (`L347-450`), command-list classes (`L453`, `L2493`, `L3590`, `L4388`, `L5035`), executor (`L5047`), flags (`L1182-1187`) |
| `RHICommandList.inl` | `FScopedRHIThreadStaller` (`L13`), breadcrumb allocator (`L183-201`) |
| `RHICommandListCommandExecutes.inl` | `ExecuteAndDestruct` bodies, `RHISTAT` (`L89`) |
| `RHIContext.h` | `IRHIComputeContext` (`L254`), `IRHIUploadContext` (`L570`), `IRHICommandContext` (`L686`), `IRHICommandContextPSOFallback` (`L877`), `ShouldEmitBreadcrumbs()` (`L555-560`) |
| `DynamicRHI.h` | `FDynamicRHI` vtable, `FDynamicRHIPSOFallback` (`L1003`), `PlatformCreateDynamicRHI()` (`L1483`), `GetSelectedDynamicRHIModuleName` (`L1487`) |
| `RHIGlobals.h` | `FRHIGlobals`/`GRHIGlobals` (`L805`), capability bits, deprecated macro aliases (`L810-828`), `GetFeatureLevelShaderPlatform` (`L1016`) |
| `RHIFeatureLevel.h` | `ERHIFeatureLevel::Type` (`L15`) |
| `RHIShaderPlatform.h` | `EShaderPlatform` (`L10`), `FStaticShaderPlatform` (`L60`) |
| `DataDrivenShaderPlatformInfo.h` | platform capability table (`L910`, `954`, `960`, `1052`, `1076`, `1299`) |
| `RHIStaticShaderPlatformNames.h` | platform name strings |
| `PipelineStateCache.h` | PSO cache API (`L163-248`) |
| `PipelineFileCache.h` | serialized PSO format (`FPipelineCacheFileFormatPSO` `L106`), manager (`L314`) |
| `PsoLruCache.h` | `TPsoLruCache` (`L20`) |
| `BoundShaderStateCache.h` | fallback PSO cache (`L12`) |
| `RHIShaderBindingLayout.h` | `FRHIShaderBindingLayout` (`L71`) |
| `RHIShaderFormatDefinitions.inl` | per-platform format definitions |
| `RHIShaderParameters.h` | shader parameter types |
| `RHIShaderLibrary.h` | shader library (`L14`) |
| `RHIBreadcrumbs.h` | breadcrumb gate (`L26-39`), node (`L136`), allocator (`L258`), event macros (`L1300-1357`) |
| `GPUProfiler.h` | `FQueue` (`L20-57`), `FGPUStat` (`L145-173`), frame history (`L579-614`) |
| `GpuProfilerTrace.h` | `UE_TRACE_GPU_PROFILER_ENABLED` (`L12-14`), metadata serializer `FMetadataSerializer` (`L22`, `AppendValue` `L28-62`), `Stats()` (`L170`) |
| `RenderTracing.h` | trace channels (`L10-39`) |
| `RHIStats.h` | stat declarations (`L108-179`) |
| `RHILockTracker.h` | `FRHILockTracker` (`L8`) |
| `RHITransientResourceAllocator.h` | `FRHITransientAllocationFences` (`L22`), `IRHITransientResourceAllocator` (`L535`) |
| `GPUDefragAllocator.h` | `FGPUDefragAllocator` (`L42`) |
| `RHIAllocators.h` | `FCommandListBaseBlockAllocationTag` (`L12`), 64 KB `BlockSize` (`L14`), `FAlignedAllocator` (`L20`), `FRHICmdListBaseArrayAllocator` / `...SetAllocator` (`L23-24`) |
| `RHIResourceCollection.h` | resource collections (`L45`) |
| `RHIResourceReplace.h` | resource replacement (`L11`, `L54`) |
| `RHIMemoryLayout.h` | memory layout helpers |
| `RHIBufferInitializer.h` / `RHITextureInitializer.h` | upload/initialize helpers |
| `RHITextureReference.h` | texture-reference indirection (`L7`); `SetReferencedTexture` (`L48`) called only from `FDynamicRHI::RHIUpdateTextureReference` — no tile residency |
| `RHIGPUReadback.h` | readback; legacy `Lock`/`Unlock` deprecated (`L99-112`) |
| `RHIUtils.h` / `RHIResourceUtils.h` / `RHIUtilities.h` / `RHITypes.h` / `RHIStrings.h` | helpers |
| `RHISurfaceDataConversion.h` | pixel format conversion |
| `RHIImmutableSamplerState.h` | deprecated since 5.8 (`L8`) |
| `RHIUniformBufferLayoutInitializer.h` | UB layout init |
| `RHITextureUtils.h` | texture utility |
| `MultiGPU.h` | `FRHIGPUMask` (`L32`), `WITH_MGPU` (`L19`) |
| `RHIValidation.h` / `RHIValidationContext.h` / `RHIValidationCommon.h` / `RHIValidationUtils.h` / `RHIValidationTransientResourceAllocator.h` | validating RHI layer |
| `TextureProfiler.h` | deprecated in 5.7 (`L7`), `TEXTURE_PROFILER_ENABLED 0` (`L17`) |
| `Tests/RHIUnitTests.h` | `IRHITestModule::RunAllTests` (`L6-11`) |
| `Windows/IDXGISwapchainProvider.h` | swapchain provider interface |
| `IOS/*.inl`, `Android/*.inl` | per-platform static feature level / shader platform / platform info |

## Runtime/RHI/Private

`RHI.cpp`, `RHICommandList.cpp`, `RHIContext.cpp`, `DynamicRHI.cpp`, `RHIGlobals.cpp`, `RHIResources.cpp`, `RHITransition.cpp`, `RHIDescriptorRange.cpp`, `RHILockTracker.cpp`, `RHITransientResourceAllocator.cpp`, `RHIGPUReadback.cpp`, `RHIBreadcrumbs.cpp`, `GPUProfiler.cpp`, `GpuProfilerTrace.cpp`, `RenderTracing.cpp`, `RHITracing.cpp` + `RHITracing.h`, `RHIStats.cpp`, `RHIStrings.cpp`, `RHIShaderPlatformConfig.cpp`, `RHIStaticShaderPlatformNames.cpp`, `RHITextureInitializer.cpp`, `RHIBufferInitializer.cpp`, `RHITextureReference.cpp`, `RHIResourceCollection.cpp`, `RHIUtilities.cpp`, `RHIValidation.cpp`, `BoundShaderStateCache.cpp`, `DataDrivenShaderPlatformInfo.cpp`, `GPUDefragAllocator.cpp`, `MultiGPU.cpp`, `PipelineFileCache.cpp`, `PipelineStateCache.cpp` — the command-list allocators are header-only (`RHIAllocators.h`).

Per-platform: `Private/Windows/WindowsDynamicRHI.cpp` (`L1407`), `Private/Linux/LinuxDynamicRHI.cpp` (`L43`), `Private/Android/AndroidDynamicRHI.cpp` (`L135`), `Private/Apple/AppleDynamicRHI.cpp` (`L156`), `Private/Apple/Platform/IOS/IOSPlatformDynamicRHI.h`, `Private/Apple/Platform/Mac/MacPlatformDynamicRHI.h`.

## Backend modules

| Module | Path | Notable files |
|---|---|---|
| `D3D12RHI` | `Runtime/D3D12RHI/Private` (92 files) | `D3D12RHIPrivate.h` (`L158`, `L701`), `D3D12CommandContext.cpp`, `D3D12CommandList.cpp`, `D3D12Submission.cpp`, `D3D12DirectCommandListManager.cpp`, `D3D12EnhancedBarriers.cpp`, `D3D12LegacyBarriers.cpp`, `D3D12ExplicitDescriptorCache.cpp`, `D3D12BindlessDescriptors.cpp`, `D3D12DescriptorRange.cpp`, `D3D12Pipelinestate.cpp`, `D3D12RootSignature.cpp`, `D3D12Query.cpp`, `D3D12PoolAllocator.cpp`, `D3D12TransientResourceAllocator.cpp`, `D3D12Residency.h`, `D3D12WorkGraph.cpp`, `D3D12RayTracing.cpp`, `D3D12Tracing.cpp`, `D3D12Stats.cpp`, `D3D12StateCache.cpp` |
| `VulkanRHI` | `Runtime/VulkanRHI/Private` (78 files) | `VulkanDynamicRHI.h` (`L66`, `L469`), `VulkanContext.cpp`, `VulkanCommands.cpp`, `VulkanSubmission.cpp`, `VulkanQueue.cpp`, `VulkanCommandBuffer.cpp`, `VulkanBarriers.cpp`, `VulkanSynchronization.cpp`, `VulkanPipeline.cpp`, `VulkanPipelineState.cpp`, `VulkanRenderpass.cpp`, `VulkanFramebuffer.cpp`, `VulkanDescriptorSets.cpp`, `VulkanBindlessDescriptorManager.cpp`, `VulkanShaderObjectManager.cpp`, `VulkanQuery.cpp`, `VulkanLayers.cpp`, `VulkanDebug.cpp`, `VulkanChunkedPipelineCache.cpp`, per-platform `Private/Linux/VulkanLinuxPlatform.cpp`, `Private/Android/VulkanAndroidPlatform.cpp`, `Private/Windows/VulkanWindowsPlatform.cpp` |
| `MetalRHI` | `Runtime/Apple/MetalRHI/Private` | `MetalDynamicRHI.h` (`L73`), `MetalDynamicRHIModule.h` (`L16`), `MetalSubmission.cpp`, `MetalCommandBuffer.h`, `MetalCommandQueue.h`, `MetalStateCache.cpp`, `MetalBindlessDescriptors.cpp`, `MetalProfiler.cpp` |
| `OpenGLDrv` | `Runtime/OpenGLDrv/Private` (33 files) | `OpenGLDrv.h` (`L40`, `L614`), `OpenGLState.cpp`, `OpenGLCommands.cpp`, `OpenGLProgramBinaryFileCache.cpp`, `OpenGLES.cpp`, `OpenGL3.cpp`, `OpenGL4.cpp`, per-platform EGL/function loaders |
| `D3D11RHI` | `Runtime/Windows/D3D11RHI/Private` (22 files) | `D3D11RHI.cpp`, `D3D11State.cpp`, `D3D11StateCache.cpp`, `D3D11Commands.cpp`, `D3D11Device.cpp`, `D3D11Query.cpp`, `D3D11NvidiaAftermath.cpp`, `D3D11RHIPrivate.h`; the dynamic-RHI interface `ID3D11DynamicRHI.h` is in `Public/` |
| `NullDrv` | `Runtime/NullDrv` (5 files) | `Public/NullRHI.h` (`L15`), `Public/NullDrv.h` (`L12`), `Private/NullRHI.cpp`, `Private/NullDrv.cpp` |

## RenderCore

| File | Responsibility |
|---|---|
| `RenderGraphBuilder.h` | `FRDGBuilder` (`L50`), resource creation (`L104`, `L110`), dependencies (`L249`), hints (`L252`), `Execute()` (`L412`) |
| `RenderGraphDefinitions.h` | `ERDGPassFlags` (`L129-161`), `ERDGBufferFlags` (`L164`), `RDG_ENABLE_DEBUG` (`L17`, engine variant `L27`) |
| `RenderGraphPass.h` | `FRDGPass` (`L217-241`), merge predicates (`L273-291`), `FRDGDispatchPass` (`L719`), `FRDGSentinelPass` (`L825`) |
| `RenderGraphResources.h` | `FRDGSubresourceState` (`L74-128`), `FAccessModeState` (`L417-437`), view caches (`L551-552`), `FRDGBufferDesc` factories (`L944-1016`) |
| `RenderGraphEvent.h` | event macros (`L477-503`) |
| `Private/RenderGraphUtils.cpp` | `FDepthStencilBinding` and load/store actions (`L800`, `L807`, `L817`) |
| `Private/RenderGraphPrivate.cpp` | prologue/epilogue (`L598`, `L1793`), clobber (`L5002`), access modes (`L898-948`) |
| `Private/RenderGraphTrace.cpp` | RDG trace event names (`L516-540`) |
| `RenderGraphValidation.h` | RDG validation |
| `RenderCore.h` | `STATGROUP_SceneRendering` stats (`L37-41`) |
| `RenderResource.h` | `FRenderResource` (`L36`), `TGlobalResource` (`L445`) |
| `Shader.h` | `FShader` and shader map |
| `ShaderParameterMacros.h` | `IMPLEMENT_STATIC_UNIFORM_BUFFER_SLOT` (`L1588`) |
| `ShaderParameterStruct.h` | `SHADER_PARAMETER_STRUCT` machinery |
| `ShaderBundles.h` | shader bundles |
| `VertexFactory.h` | `FVertexFactoryShaderParameters` (`L172`), `FVertexFactoryType` (`L314`), `FVertexFactory` (`L655`) |
| `RHIStaticStates.h` | `TStaticStateRHI` (`L20-53`), `GetRHI()` `checkSlow` (`L28`), `TGlobalResource` in `EInitPhase::Pre` (`L49`), `GetStaticRasterizerState` (`L99-102`) — see chapter 17 |
| `RenderTargetPool.h` / `PooledRenderTarget.h` | external RDG resources |
| `DumpGPU.h` | GPU dump |
| `ProfilingDebugging/RealtimeGPUProfiler.h` | ProfileGPU UI |

## Renderer draw path

| File | Responsibility |
|---|---|
| `Renderer/Public/MeshPassProcessor.h` | `FMeshDrawCommand` (`L1281`), `FMeshDrawCommandStateCache` (`L905-960`), `FMeshDrawShaderBindings` (`L966-1175`), `FVisibleMeshDrawCommand` (`L1762`), `FCompareFMeshDrawCommands` (`L1826`), `EMeshPass::Type` (`L56-100`), `FDynamicPassMeshDrawListContext` (`L1954`) |
| `Renderer/Private/MeshPassProcessor.cpp` | submission (`L1248-1386`), sorting loop (`L1646-1677`), bucket assignment (`L2100`, `L2145`, `L2175`) |
| `Renderer/Private/MeshDrawCommands.cpp` | merge (`L506-569`), indirect fallback (`L930-936`), sort keys (`L276-346`), parallel draw tasks (`L1673-1723`) |
| `Renderer/Private/MeshDrawCommands.h` | `FParallelMeshDrawCommandPass` (`L113`), setup task context (`L54`), `FPrimitiveIdVertexBufferPool` (`L31`) |
| `Renderer/Public/SceneRendering.h` | `FSceneRenderer`, deprecated `FParallelCommandListSet` (`L550`) |
| `Renderer/Private/SceneRendering.cpp` | parallel command list impl (`L919-972`), `r.RHICmdMinDrawsPerParallelCmdList` (`L512-515`) |
| `Renderer/Private/DeferredShadingRenderer.cpp` | frame flow (`L2116-2465`) |
| `Renderer/Private/BasePassRendering.h` | per-pass RDG parameters with `FInstanceCullingDrawParams` (`L162-163`) |
| `Renderer/Private/GPUScene.h` | `FGPUScene` (`L26`) |
| `Renderer/Private/InstanceCulling/` | instance culling |
| `Renderer/Public/InstanceCulling/InstanceCullingContext.h` | culling context (`L23-255`) |
| `Renderer/Private/InstanceCulling/InstanceCullingContext.cpp` | indirect args (`L253-281`), compaction (`L431-461`), clear pass (`L1363-1391`), submission (`L1758-1803`) |
| `Renderer/Private/SceneVisibilityPrivate.h` | visibility |

## Engine instancing

| File | Responsibility |
|---|---|
| `Engine/Public/MeshBatch.h` | `FMeshBatchElement` (`L231-333`), `FMeshBatch` (`L364-558`) |
| `Engine/Public/SceneManagement.h` | `FMeshElementCollector::DrawBatchedElements` (`L1382`) |
| `Engine/Private/SceneManagement.cpp` | LOD/screen-size selection (`L956-1191`) |
| `Engine/Public/InstancedStaticMeshComponent.h` | `FInstancedStaticMeshInstanceData` (`L40`) |
| `Engine/Private/InstancedStaticMesh.h` | `FInstancedStaticMeshSceneProxy` (`L435`) |
| `Engine/Private/InstancedStaticMesh.cpp` | `GetDynamicMeshElements` (`L1102`), LOD selection (`L1119-1127`) |

## Tests

`RHI/Public/Tests/RHIUnitTests.h` (`L6-11`), launched from `Launch/Private/LaunchEngineLoop.cpp` (`L4379-4384`); suite in `Engine/Plugins/Tests/RHITests/Source/RHITests/Private`; shaders in `Engine/Plugins/Tests/RHITests/Shaders/Private`. See chapter 16.
