# 14 — File Reference

Line counts are `wc -l` over `UE_5.8/Engine/Source/Runtime/RenderCore`. Graph degree is from `graphify-out/graph.json`.

## Public (`Engine/Source/Runtime/RenderCore/Public/`)

| File | Lines | Responsibility | Graph degree |
|---|---|---|---|
| `RenderGraph.h` | 65 | Umbrella include (`:59-65`) and the usage contract (`:5-57`) | — |
| `RenderGraphFwd.h` | 59 | Forward declarations and `Ref` typedefs (`:13-59`) | — |
| `RenderGraphDefinitions.h` | 801 | Compile gates (`:17-63`), `FRDGAsyncTask` (`:107`), `ERDGBuilderFlags` (`:111-126`), `ERDGPassFlags` (`:129-161`), `ERDGBufferFlags` (`:164-183`), `ERDGTextureFlags` (`:186-208`), `ERDGSetupTaskWaitPoint` (`:210-219`), `ERDGUnorderedAccessViewFlags` (`:222-229`), `ERDGViewableResourceType` (`:232`), `ERDGViewType` (`:240`), extraction flags (`:264`), initial-data flags (`:275`), pooled alignment (`:291-301`), `TRDGHandle` (`:344-435`), `TRDGHandleRegistry` (`:447-560`), `TRDGHandleBitArray` (`:563-579`), `TRDGHandleUniqueFilter` (`:586-620`), `FRDGTextureDesc` (`:628-742`), handle typedefs (`:761-783`), callback typedefs (`:793-801`) | 623 |
| `RenderGraphBuilder.h` | 1200 | `FRDGBuilder` (`:50`): registration, allocation, execution, extraction, registries (`:533-537`), prologue/epilogue (`:464-465`) | 793 |
| `RenderGraphBuilder.inl` | — | Inline implementations: `CreateTexture` `:41`, `CreateBuffer` `:65`, `:84`, views `:102-126`, allocators `:140-192`, `CreateUniformBuffer` `:234`, `AddPassInternal` `:249`, `AddPass` `:275`, `:295`, `:311`, `AddDispatchPass` `:326`, `SetPassWorkload` `:365`, upload/commit `:370-427`, extraction `:441-495`, `AddDispatchHint` `:501`, setup tasks `:514-666`, final access `:747`, `:753` | — |
| `RenderGraphResources.h` | 1515 | `FRDGSubresourceState` (`:72-129`), `FRDGResource` (`:134`), `FRDGUniformBuffer` (`:195`), `FRDGViewableResource` (`:293`), `FAccessModeState` (`:417-437`), `FRDGView` (`:500`), `FRDGTexture` (`:569`), views (`:696`, `:719`, `:837`, `:903`, `:1417`, `:1443`), `FRDGBufferDesc` (`:939`), `FRDGPooledBuffer` (`:1194`), `FRDGBuffer` (`:1307`), `ExtractRenderTargetsInfo` (`:1512-1513`) | 1483 |
| `RenderGraphResources.inl` | — | `ExtractRenderTargetsInfo` implementation `:208-210`, slot conversion from `:213` | — |
| `RenderGraphPass.h` | 841 | `FRDGBarrierBatchBeginId` region `:54-56`, `FRDGBarrierBatchEndId` (`:88-106`), `FRDGBarrierBatchBegin` (`:108`), `FRDGBarrierBatchEnd` (`:188-213`), `FRDGPass` (`:217`), predicates (`:224-323`), fields (`:350-553`), `FRDGDispatchPass` (`:715`), `FRDGSentinelPass` (`:821`), forward decls (`:837-841`) | 341 |
| `RenderGraphParameter.h` | 346 | `FRDGParameter` (`:9-209`), `FRDGParameterStruct` (`:211-255`), static UBs (`:278-279`), `GetParameterInternal` (`:286-301`), helper (`:341-346`) | 228 |
| `RenderGraphParameters.inl` | — | `Enumerate` `:6-17`, texture/buffer traversal `:20-45`, graph UB visit `:48-58` | — |
| `RenderGraphBlackboard.h` | 179 | Design comment (`:22-55`), `FRDGBlackboard` (`:56-180`), registration macro (`:15-20`) | — |
| `RenderGraphEvent.h` | 656 | `FRDGEventName` (`:43-69`), `FRDGScope` (`:173-280`), scope macros (`:438-486`), fallbacks (`:495-516`), deprecated GPU stats (`:520-521`), task/transition allocator domains (`:581-588`) | 245 |
| `RenderGraphEvent.inl` | — | Inline scope implementation | — |
| `RenderGraphTrace.h` | 63 | `RDGChannel` (`:23`), `FRDGTrace` (`:25-45`), `RenderTracing` functions (`:51-60`) | — |
| `RenderGraphUtils.h` | 1448 | Helper pass declarations, `ClearUnusedGraphResources` (`:171-208`, `:220-240`), `FComputeShaderUtils` (`:451-686`), readback (`:1241`, `:1253`), barrier pass (`:1441`, `:1446`) | 631 |
| `RenderGraphValidation.h` | 181 | `FRDGUserValidation` (`:40-150`), `FRDGBarrierValidation` (`:153-179`), gated by `RDG_ENABLE_DEBUG` (`:24`) | 53 |
| `RenderGraphAllocator.h` | 263 | `FRDGAllocator` (`:17-104`), object retention and `ReleaseAll()` (`:118-139`), `FRDGAllocatorScope` (`:141-153`) | 29 |
| `RenderGraphTextureSubresource.h` | 271 | `FRDGTextureSubresource` (`:11`), layout (`:67`), range (`:126`), checks (`:112`, `:232-233`) | 35 |

## Private (`Engine/Source/Runtime/RenderCore/Private/`)

| File | Lines | Responsibility | Graph degree |
|---|---|---|---|
| `RenderGraphBuilder.cpp` | 5259 | Compile (`:1327`), execute (`:1766`), collect (`:3590`, `:3632`), allocate (`:3129`, `:3180`, `:3217`), views (`:3315`), barriers (`:3783`, `:3872`, `:3930`), transitions (`:4045-4586`), scopes (`:3346-3408`), prologue/epilogue (`:3424`, `:3457`), execute (`:3511`, `:3530`), MGPU copy (`:5162`) | 680 |
| `RenderGraphResources.cpp` | 222 | Constructors (`:11-67`), `IsMergeAllowed` (`:69-133`), `IsTransitionRequired` (`:135-152`), `SkipUAVBarrier` (`:6-9`) | — |
| `RenderGraphPass.cpp` | 465 | Pass execution, static uniform buffers (`:10-34`) | — |
| `RenderGraphPrivate.h` | 261 | Dump flags (`:12-14`), async compute gates (`:16-18`), parallel gate (`:20-22`), RHI access scope (`:39-55`), debug predicates (`:63-67`), constants (`:69-82`), globals (`:84-115`), CSV (`:117-123`), `RDG_STATS` (`:125`), stat globals (`:128-142`), trace counters (`:145-162`), `STATGROUP_RDG` counters (`:164-180`), cycle stats (`:182-188`), `SkipUAVBarrier` (`:208-217`), `IsImmediateMode` (`:219-222`), `IsRenderPassMergeEnabled` (`:224-227`), `IsAsyncComputeSupported` (`:229-234`), extended-lifetime enumeration (`:245-262`) | — |
| `RenderGraphPrivate.cpp` | 702 | CVars (`:18-426`), clobber helpers (`:168-204`) | — |
| `RenderGraphResourcePool.h` | 142 | `FRDGBufferPool` (`:14-64`), `GRenderGraphResourcePool` (`:67`), lifetime states (`:69-74`), `FRDGTransientRenderTarget` (`:76-111`), `FRDGTransientResourceAllocator` (`:113-141`), `GRDGTransientResourceAllocator` (`:142`) | 198 |
| `RenderGraphResourcePool.cpp` | 415 | Pool page alignment `64 * 1024` (`:34`), PowerOfTwo fallback (`:40`) | — |
| `RenderGraphTrace.cpp` | 712 | `RDGChannel` definition (`:13`), messages (`:15-121`), `r.Trace.RDG.UseFullPassNames` (`:544`), event names (`:516-540`) | — |
| `RenderGraphUtils.cpp` | 1057 | `r.RDG.BarrierPass` (`:15`), `ClearUnusedGraphResourcesImpl` (`:22-100`, `:103`), copy texture (`:268-294`), draw texture (`:296-383`), copy buffer (`:386-413`), clear UAV (`:415-435`, `:453-483`), clear render target (`:650`), enqueue copy (`:829-861`), external access queue submit (`:990`) | 421 |
| `RenderGraphValidation.cpp` | 1208 | Concrete checks (`:241`, `:522`, `:583`, `:623`) | 205 |
| `RenderGraphAllocator.cpp` | 112 | Out-of-scope assert (`:51-54`), `ReleaseAll` (`:71-95`), scope implementation (`:11`, `:98-107`) | — |
| `RenderGraphBlackboard.cpp` | 29 | Blackboard implementation | — |
| `RenderGraphEvent.cpp` | 366 | Event and scope implementation | 38 |

## RenderGraphInsights plugin

`Engine/Plugins/RenderGraphInsights/Source/RenderGraphInsights/Private/`:

| File | Responsibility | Graph degree |
|---|---|---|
| `RenderGraphProvider.h` | `FPacket` (`:39`), `FPassIntervalPacket` (`:53`), `FScopePacket` (`:63`), `FResourcePacket` (`:74`), `FTexturePacket` (`:101`), `FBufferPacket` (`:113`), `FPassPacket` (`:125`), `FGraphPacket` (`:153`), `FRenderGraphProvider` (`:203`) | 310 |
| `RenderGraphTrack.h` | `FBoundingBox` (`:23`), `FVisibleItem` (`:37`), `FVisiblePass` (`:97`), `FVisibleGraph` (`:142`), `FRenderGraphTrack` (`:203`) | 442 |
| `RenderGraphTrackDrawHelper.h` | `EDrawLayer` (`:49`), `FSplinePrimitive` (`:63`), `FRenderGraphTrackDrawState` (`:73`), builder (`:109`), helper (`:161`), `DrawTrackHeader` (`:177-178`), `DrawSpline` (`:180`) | 258 |
| `RenderGraphTrack.cpp` | `FRenderGraphTimingEvent` (`:143`), visible event types (`:192-221`), `FPacketFilter` (`:248`) | 192 |
| `RenderGraphAnalyzer.h` | `FRenderGraphAnalyzer` (`:16`) | — |
| `RenderGraphTimingViewSession.h` | `FRenderGraphTimingViewSession` (`:19`) | — |
| `RenderGraphTimingViewExtender.h` | `FRenderGraphTimingViewExtender` (`:18`) | — |
| `RenderGraphInsightsModule.h` | `FRenderGraphInsightsModule` (`:17`) | — |
| `RenderGraphTraceModule.h` | `FRenderGraphTraceModule` (`:12`) | — |
| `RenderGraphInsights.uplugin` | Plugin manifest | — |

Adjacent RDG surface outside RenderCore: `NiagaraRenderGraphUtils.h` (30 nodes), `Engine/Source/Runtime/Renderer/Private/` consumers, `Engine/Source/Runtime/Engine/Private/` consumers.
