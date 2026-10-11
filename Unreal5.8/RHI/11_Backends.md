# Backends — UE 5.8 RHI

## Scope

The concrete `FDynamicRHI` implementations. Each backend is a dynamically loaded module that implements the same vtable with different submission, state, and descriptor models.

## Selection

`GetSelectedDynamicRHIModuleName()` (`RHI/Public/DynamicRHI.h:1487`) — noted as "very slow when called before RHIInit" (`L1486`). `RHIInit()` (`Private/DynamicRHI.cpp:291`) calls `PlatformCreateDynamicRHI()` (`DynamicRHI.h:1483`), implemented per platform: Windows `Private/Windows/WindowsDynamicRHI.cpp:1407`, Linux `Private/Linux/LinuxDynamicRHI.cpp:43`, Android `Private/Android/AndroidDynamicRHI.cpp:135`, Apple `Private/Apple/AppleDynamicRHI.cpp:156`.

Windows path: `LoadDynamicRHIModule(RequestedFeatureLevel, LoadedRHIModuleName)` (`L1413`), then ConfigRules can hotfix the choice using `GRHIDeviceId`/`GRHIVendorId` (`L1420-1430`), then `DynamicRHIModule->CreateRHI(RequestedFeatureLevel)` (`L1433`). `LogTDRSummary()` (`L1437`) reports TDR settings. Driver denylist (`DynamicRHI.cpp:35`) and `GDynamicRHIIgnoredSM5Preference` (`DynamicRHI.cpp:36`) record when the requested feature level was overridden.

Failure falls back to NullDrv (`DynamicRHI.cpp:373`); commandlets and `-nullrhi` use it directly (`L60-71`).

## Backends

| Backend | Dynamic RHI | Module | Submission model | Pipeline model |
|---|---|---|---|---|
| D3D11 | `FD3D11DynamicRHI` | `D3D11RHI` | immediate + deferred contexts | bound shader state (PSO fallback) |
| D3D12 | `FD3D12DynamicRHI` (`D3D12RHIPrivate.h:158`) | `FD3D12DynamicRHIModule` (`L701`) | `FD3D12CommandList` + `FD3D12Queue` (`D3D12Submission.h:18`, `D3D12CommandList.h:19`) | real PSOs, explicit descriptor caches (`D3D12DescriptorCache.h:13`, `D3D12ExplicitDescriptorCache.h:8`) |
| Vulkan | `FVulkanDynamicRHI` (`VulkanDynamicRHI.h:66`) | `FVulkanDynamicRHIModule` (`L469`) | `VkCommandBuffer` via command pools, `FVulkanQueue` (`VulkanQueue.h:16`), sync points/fences (`VulkanSubmission.h:14`, `VulkanQuery.h:18`) | `VkPipeline` via PSO cache, descriptor sets/pools (`VulkanDevice.h:18-20`) |
| Metal | `FMetalDynamicRHI` (`Apple/MetalRHI/Private/MetalDynamicRHI.h:73`) | `FMetalDynamicRHIModule` (`MetalDynamicRHIModule.h:16`) | `FMetalCommandBuffer` + encoders, `FMetalSyncPoint` (`MetalSubmission.h:17`, `L21`) | `MTLRenderPipelineState` |
| OpenGL | `FOpenGLDynamicRHI` (`OpenGLDrv/Private/OpenGLDrv.h:40`) | `FOpenGLDynamicRHIModule` (`L614`) | immediate state machine (`FOpenGLState`), `FOpenGLLinkedProgram` (`OpenGLResources.h:37`) | `IRHICommandContextPSOFallback` (`OpenGLDrv.h:40`) — no real pipeline object |
| Null | `FNullDynamicRHI` (`NullRHI.h:15`) | `FNullDynamicRHIModule` (`NullDrv/Public/NullDrv.h:12`) | no-op | `FDynamicRHIPSOFallback` (`DynamicRHI.h:1003`) |

NullDrv and OpenGL deriving from the PSO-fallback interfaces is the concrete evidence that the pipeline-state model is a capability, not a universal: `FDynamicRHIPSOFallback::RHICreateGraphicsPipelineState` (`DynamicRHI.h:1006`) and `IRHICommandContextPSOFallback` (`RHIContext.h:877`) exist for exactly these RHIs, and `PLATFORM_USE_FALLBACK_PSO` (`RHICommandList.h:3896`) routes them through `FBoundShaderStateCache` (`BoundShaderStateCache.h:12`).

## What differs across backends

- **Barrier granularity.** D3D12 and Vulkan implement `ERHIAccess` as explicit barriers (`RHITransition.h:118`); Metal uses encoder boundaries; OpenGL has none. `GRHIGlobals.NeedsExtraTransitions` (`RHIGlobals.h:778`) and `NeedsShaderUnbinds` (`L760`) capture those differences.
- **Async compute.** Only if `GRHIGlobals.SupportsEfficientAsyncCompute` (`RHIGlobals.h:267`); otherwise `GetPipelineFromFlags` returns Graphics only (`RHICommandList.h` / `RHIPipeline.h:46-50`).
- **Descriptor model.** Bindless and descriptor-range support is per-backend (`GRHIGlobals.bSupportsDescriptorRange`, `DescriptorTypeGroups` `RHIGlobals.h:718`), which is why bindless shaders are gated by `PLATFORM_SUPPORTS_DESCRIPTOR_RANGE`.
- **UAV limits.** `MinGuaranteedSimultaneousUAVs = 8` (`RHIGlobals.h:356`) and the per-backend `MaxSimultaneousUAVs` (`L359`, initialised to the guaranteed minimum); validation reserves the guaranteed count per shader (`RHIValidationCommon.h:958`). A pass that binds more UAVs than the backend allows fails validation, not silently.
- **Indirect draw.** `GRHIGlobals.SupportsDrawIndirect` (`RHIGlobals.h:115`) and `SupportsMultiDrawIndirect` (`L118`); `MultiDrawIndexedPrimitiveIndirect` documents the requirement (`RHIContext.h:782-786`).
- **Reserved/sparse resources.** `GRHIGlobals.ReservedResources.Supported` (`RHITransition.h:110`).
- **Shader bundles.** `GRHIGlobals.ShaderBundles.SupportsDispatch` (`RHIGlobals.h:661`).

## Practical consequences

- Feature-parity bugs are backend-specific: a shader that uses vertex-shader UAVs is fine on D3D12 but requires `bSupportsVertexShaderUAVs` plus a runtime `GRHIGlobals.SupportsVertexShaderUAVs` check (`DataDrivenShaderPlatformInfo.h:102`, `723`).
- The same material can produce different PSO counts per backend because the pipeline key includes formats and flags that some backends ignore.
- `-nullrhi` runs the full render path with no GPU, which is why RDG graph structure and validation bugs reproduce without a GPU.
