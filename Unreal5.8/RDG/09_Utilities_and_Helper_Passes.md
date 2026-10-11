# 09 — Utilities and Helper Passes

**Helper passes are the preferred way to add work** (`RenderGraph.h:54-56`): they declare access correctly, run `ClearUnusedGraphResources`, and produce event names with the parameters that make profiling readable. A hand-written lambda is only justified when no helper covers the work.

## Canonical shape

`AddCopyBufferPass` (`RenderGraphUtils.cpp:386-403`) is the reference form — parameter struct with declared access, `AllocParameters`, `AddPass` with an event name, flag, and lambda:

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FCopyBufferParameters, )
    RDG_BUFFER_ACCESS(SrcBuffer, ERHIAccess::CopySrc)
    RDG_BUFFER_ACCESS(DstBuffer, ERHIAccess::CopyDest)
END_SHADER_PARAMETER_STRUCT()

void AddCopyBufferPass(FRDGBuilder& GraphBuilder, FRDGBufferRef DstBuffer, uint64 DstOffset,
                       FRDGBufferRef SrcBuffer, uint64 SrcOffset, uint64 NumBytes)
{
    check(SrcBuffer);
    check(DstBuffer);

    FCopyBufferParameters* Parameters = GraphBuilder.AllocParameters<FCopyBufferParameters>();
    Parameters->SrcBuffer = SrcBuffer;
    Parameters->DstBuffer = DstBuffer;

    GraphBuilder.AddPass(
        RDG_EVENT_NAME("CopyBuffer(%s Size=%ubytes)", SrcBuffer->Name, SrcBuffer->Desc.GetSize()),
        Parameters,
        ERDGPassFlags::Copy,
        [&Parameters, SrcBuffer, DstBuffer, SrcOffset, DstOffset, NumBytes](FRDGAsyncTask, FRHICommandList& RHICmdList)
        {
            RHICmdList.CopyBufferRegion(DstBuffer->GetRHI(), DstOffset, SrcBuffer->GetRHI(), SrcOffset, NumBytes);
        });
}
```

The event name carries the resource name and size (`:396`) — that is what makes the pass identifiable in RenderGraphInsights and in a GPU dump.

## Helper catalog

| Helper | Declaration | Implementation |
|---|---|---|
| `FComputeShaderUtils::AddPass` | `RenderGraphUtils.h:538-686` (flag validation and `ClearUnusedGraphResources` at `549-564`; callback form `580-600`) | — |
| `FComputeShaderUtils::Dispatch` | `:479-504` | `PrepareDispatch` `:451-460` |
| `FComputeShaderUtils::DispatchIndirect` | `:506-536` | `AddIndirectArgsSetupCsPass1D` (`RenderGraphUtils.cpp:880`) |
| `FPixelShaderUtils::DrawFullscreenTriangle` / `Quad` | `PixelShaderUtils.h:63-67` | graphics PSO and bind `105-107`, `146-148` |
| `FPixelShaderUtils::AddFullscreenPass` | `:153-177` | — |
| `FPixelShaderUtils::AddRasterizeToRectsPass` | `:218-252` | PSO/bind `275-288` |
| `AddCopyTexturePass` | `RenderGraphUtils.h:711` (region `696-729`) | `RenderGraphUtils.cpp:268-294` |
| `AddDrawTexturePass` | `RenderGraphUtils.h:750-755` (`FRDGDrawTextureInfo` `731-748`) | `RenderGraphUtils.cpp:296-383` |
| `AddClearUAVPass` (buffer) | `RenderGraphUtils.h:757-760` | `RenderGraphUtils.cpp:415-435` |
| `AddClearUAVPass` (texture) | `:762-783` | `RenderGraphUtils.cpp:453-483`, `FClearUAVRectsPS` `:554` |
| `AddClearRenderTargetPass` | `:785-792`, `FRDGTextureClearInfo` overload `:794-814` | `RenderGraphUtils.cpp:650` |
| `AddClearStencilPass` | — | `RenderGraphUtils.cpp:804` |
| `AddResummarizeHTilePass` | — | `RenderGraphUtils.cpp:811` |
| `AddCopyBufferPass` | `:838-841` | `RenderGraphUtils.cpp:386-413` |
| `AddEnqueueCopyPass` | `:843-847` | `RenderGraphUtils.cpp:829-861` |
| `FRDGUploadData` | `:849-858` | — |
| `CreateUploadBuffer` | `:1086-1128` | `RenderGraphUtils.cpp:957` |
| `CreateStructuredUploadBuffer` | `:1134-1144` | — |
| `CreateByteAddressUploadBuffer` | `:1150-1160` | — |
| `CreateStructuredBuffer` | — | `RenderGraphUtils.cpp:905` |
| `CreateByteAddressBuffer` | — | `RenderGraphUtils.cpp:932` |
| `CreateVertexBuffer` | — | `RenderGraphUtils.cpp:974` |
| `CreateTextureMSAA` | `:215` | — |
| `RegisterExternalTextureWithFallback` | `:199` | — |
| `AllocatePooledBuffer` | — | `RenderGraphUtils.cpp:999`, `AllocatePooledBufferCurrentLLMTag` `:1032` |
| `AllocatePooledTexture` | — | `RenderGraphUtils.cpp:1021` |
| `AddReadbackTexturePass` | `RenderGraphUtils.h:1241` | — |
| `AddReadbackBufferPass` | `:1253` | — |
| `AddBarrierPass` | `:1441`, `:1446` | gated by `r.RDG.BarrierPass` (`RenderGraphUtils.cpp:15`) |
| `AddPass` / `AddPassIfDebug` | `:1218`, `:1224` | — |
| `AddDispatchToRHIThreadPass` | `:1231` | — |
| `FGenerateMips::Execute` | `GenerateMips.h:37-51`, compute overloads `53-65` | raster `GenerateMips.cpp:129-198`, compute `200-259`, entrypoint `339-349` |
| `ClearUnusedGraphResources` | `RenderGraphUtils.h:171-208`, `220-240` | `RenderGraphUtils.cpp:22-100`, `103` |
| `FRDGExternalAccessQueue::Submit` | — | `RenderGraphUtils.cpp:990` |

Names that do **not** exist in RenderCore (searched `Public/` and `Private/`): `AddClearUAVBufferPass`, `AddBlitTexture`, `AddDrawMesh`, `AddRectangle`, `AddGenerateMipmaps`, `AddConvert*`, `AddFillBuffer`, `AddUploadBuffer`, `AddComputePass`, `AddParallel*`. The exact symbols are the ones in the table above.

## Readback and debug helpers

Readback passes are `ERDGPassFlags::Readback = Copy | NeverCull` (`RenderGraphDefinitions.h:159`), so they are never culled (`RenderGraphBuilder.cpp:1387-1423`). `AddPassIfDebug` (`RenderGraphUtils.h:1224`) adds a pass only under `RDG_ENABLE_DEBUG`. `VisualizePassOutputs` (`RenderGraphBuilder.cpp:4903`) and `ClobberPassOutputs` (`:4994`) are the debug execution paths; `r.RDG.ClobberResources` (`RenderGraphPrivate.cpp:60`, `ECVF_Cheat` `:67`) forces pass outputs to a recognisable value — `GetClobberColor()` `FLinearColor(1000,1000,1000,1000)`, `GetClobberDepth()` `0.123456789f`, `GetClobberStencil()` `123` (`RenderGraphPrivate.cpp:168-204`).

## Why helpers matter

Three concrete reasons, all observable in source:

1. Access is declared in the parameter struct, so barriers are correct without manual state management (`ShaderParameterMacros.h:1868-1885`).
2. `ClearUnusedGraphResources` removes references the shader does not read (`RenderGraphUtils.cpp:22-100`), which shortens tracked lifetimes and allows overlap (`RenderGraph.h:23-27`).
3. Event names include parameters, so a profiling trace is diagnosable without re-running with a filter.
