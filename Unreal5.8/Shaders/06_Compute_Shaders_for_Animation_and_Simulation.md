# 06 — Compute Shaders for Animation and Simulation

**`FComputeShader` is not a class in 5.8.** Grep for `class FComputeShader` in `Runtime/RenderCore` returns no matches. A compute shader is an `FGlobalShader` registered with `SF_Compute`; the RHI-side type is `FRHIComputeShader` (`Runtime/RHI/Public/RHI.h`). `FDispatchGraph` and `FDispatchParams` do not exist.

## Dispatch surface

| Layer | Call | File |
|---|---|---|
| RHI | `DispatchComputeShader` | `Runtime/RHI/Public/RHICommandList.h:2699` |
| RHI indirect | `DispatchIndirectComputeShader` | `RHICommandList.h:2709` |
| RDG helper | `FComputeShaderUtils::AddPass` | `RenderCore/Public/RenderGraphUtils.h:333` |
| Group sizing | `FComputeShaderUtils::GetGroupCount` | `RenderGraphUtils.h:339` |
| Golden 2D size | `kGolden2DGroupSize = 8` | `RenderGraphUtils.h:336` |

`RenderGraphUtils.h:336` documents why 8×8: it fills one GCN wavefront or two Nvidia warps. Group size is a compile-time define in the shader source; the CPU side only computes the dispatch count.

## Animation and simulation consumers

| System | Compute path | Evidence |
|---|---|---|
| Niagara | `FNiagaraGpuComputeDispatch::DispatchStage(FRDGBuilder&, ...)` | `Engine/Plugins/FX/Niagara/Source/Niagara/Private/NiagaraGpuComputeDispatch.cpp:1475` |
| Niagara shader family | `FNiagaraShaderType`, `SF_Compute` entries | `NiagaraShaderType.h:82`, `NiagaraShader.cpp:31` |
| GPU skin cache | compute dispatches for bone transforms and tangent recomputation | `Runtime/Engine/Private/GPUSkinCache.cpp`: `FGPUSkinCacheCS` (`935`), `FInitDynamicMeshBoundsCS` (`952`), `FRecomputeTangentsPerTriangleCS` (`1172`), `FRecomputeTangentsPerVertexCS` (`1214`); dispatch through `FComputeShaderUtils::Dispatch` (`1035`) and `DispatchComputeShader` (`1258`, `1364`) — not `Renderer/Private/Skinning/` |
| Skin cache scene extension | `SkinningSceneExtension.cpp` | `Runtime/Renderer/Private/Skinning/` |
| Mesh deformation | `IMeshDeformerProvider` modular feature, `FDefaultMeshDeformerSetup`, `GetDefaultMeshDeformer()` | `RenderCore/Public/Animation/MeshDeformerProvider.h:17,30-34,36-38` |
| VectorVM (dataflow/ComputeFramework) | `ShaderFormatVectorVM` bytecode, `FComputeKernelShaderType` | `Developer/ShaderFormatVectorVM/Private/ir_vm_gen_bytecode_visitor.cpp` (188 nodes, degree 467), `ComputeKernelShaderType.h:67` |
| HairStrands, PCG, pressure/velocity grids | plugin compute shader types | `NiagaraDataInterfaceHairStrands.cpp:1000`, `NiagaraDataInterfacePressureGrid.cpp:391`, `NiagaraDataInterfaceVelocityGrid.cpp:606,659` |
| Shader bundles | `FDispatchShaderBundleCS` | `RenderCore/Private/ShaderBundles.cpp:15` |

## Skin cache CVars (`Runtime/Engine/Private/GPUSkinCache.cpp`)

| CVar | Default | Line |
|---|---|---|
| `r.SkinCache.Allow` | true | 78-83 |
| `r.SkinCache.CompileShaders` | 0 | 92-98 |
| `r.SkinCache.Mode` | — | 104-113, delegate `FGPUSkinCache::OnCVarUpdate` |
| `r.SkinCache.DefaultBehavior` | `ESkinCacheDefaultBehavior::Inclusive` | 118-123 |
| `r.SkinCache.RecomputeTangents` | — | 129-136 |
| `NumTangentIntermediateBuffers` | — | 142-146 |
| `Debug` | 1.0 | 151-155 |
| `StoreDuplicatedVerticesForRecomputeTangents` | — | 159-163 |
| `UseDuplicatedVerticesForRecomputeTangents` | — | 168-172 |
| `RecomputeTangentsParallelDispatch` | — | 177-181 |
| `PrintMemorySummary` | — | 187-191 |
| `AsyncCompute` | — | 197-201 |
| `DynamicMeshBounds` | — | 206-210 |
| `Capture` | — | 217-220 |

`r.SkinCache.CompileShaders` is the gate a tech artist must set before a deformer or skin cache permutation exists; the project setting mirrors it (`Runtime/Engine/Classes/Engine/RendererSettings.h:1534-1543`, `SkinningShaderSupport` and `SkinCache.DefaultBehavior`). `DynamicMeshBoundsModifyCompilationEnvironment` is the compile-env hook (`Runtime/Engine/Public/GPUSkinCache.h:114`); the header contract is `GPUSkinCache.h:9-21`. Exec command `list skincacheusage` (`GPUSkinCache.cpp:49-76`), stats group `STATGROUP_GPUSkinCache` (`39-45`).

## Async compute

There is no `AddAsyncComputePass`. Async compute is a pass flag: `ERDGPassFlags::AsyncCompute` (`RenderCore/Public/RenderGraphDefinitions.h:129-160`) passed to `GraphBuilder.AddPass`. `r.RDG.AsyncCompute` defaults to `RDG_ASYNC_COMPUTE_ENABLED` (1) with a sink (`RenderGraphPrivate.cpp:211-229`). Skin cache has its own `r.SkinCache.AsyncCompute` (`GPUSkinCache.cpp:197-201`). See [../RDG/](../RDG/) for the pass model.

## Group size and wave behaviour

`kGolden2DGroupSize = 8` (`RenderGraphUtils.h:336`) is the engine's default heuristic. `[INFERENCE]` — the engine does not enforce wave alignment for arbitrary group sizes; a group size that is not a multiple of the hardware wavefront (64 Nvidia, 64 GCN) wastes lanes, which is why the 8×8 default is chosen.
