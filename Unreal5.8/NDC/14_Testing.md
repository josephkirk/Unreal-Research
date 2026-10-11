# 14 — Testing and Verification

**NDC has no automated tests in UE 5.8.** The Niagara plugin has no test module — its modules are `Niagara`, `NiagaraAnimNotifies`, `NiagaraBlueprintNodes`, `NiagaraCore`, `NiagaraEditor`, `NiagaraEditorWidgets`, `NiagaraShader`, `NiagaraVertexFactories` — and no `DataChannel` test file exists in the plugin. Verification is runtime observation through the debug utilities, the SimCache visualizer, asserts and console variables.

## Debug surface

`NiagaraDataChannelDebugUtilities.cpp` is gated by `#if WITH_NIAGARA_DEBUGGER` (`:38`) for `BeginFrame`, `EndFrame`, `Tick` and `FindDataChannelHandler` (`:39-61`), and the console command `fx.Niagara.DataChannels.DumpWriteLog` → `DumpAllWritesToLog` is also inside `WITH_NIAGARA_DEBUGGER` (`:29`). `LogWrite` (`:63-147`) keeps a rolling `FrameData` window sized by `FrameDataToCapture` (`:66-85`), records `DebugSource`, the three visibility flags and the `TickGroup` (`:86-102`), and prints:

```
Frame %llu, TG %ls, NDC write by %ls (BP[%ls]/CPU[%ls]/GPU[%ls]): %i entries to data channel %ls %ls%ls
```

(`:112-146`). `Get()` is a lazy singleton (`:178-181`), `TearDown` `:183-186`, `ToJson(FNiagaraDataChannelGameData*)` `:188-288`, `TickGroupToString` `:290-294`.

Compile-time debug flags: `DEBUG_NDC_ACCESS` = `!UE_BUILD_SHIPPING && !UE_BUILD_TEST` (`NiagaraDataChannelAccessor.h:174`) enables `ValidateLayout()` stale-layout asserts (`:303-309`); `DEBUG_NDI_DATACHANNEL` (`NiagaraDataInterfaceDataChannelCommon.h:34`); `DEBUG_NDC_MAP_ACCESS` is hard-coded 0 (`NiagaraDataChannel_Map.h:156`). Missing-parameter collection is only in non-shipping builds (`...Common.cpp:52-54`), and the access-context type warning is non-shipping only (`NiagaraDataChannelHandler.cpp:215-229`).

## Asserts and error paths worth testing

| Check | Anchor |
|---|---|
| Game thread only | `NiagaraDataChannelAccessorImpl.h:35,81`; `NiagaraDataChannelData.cpp:395`; `NiagaraDataChannel.h:146-151`; `NiagaraDataChannelHandler.h:153-164`; `NiagaraDataChannelManager.cpp:79-106,156-178` |
| Buffer size matches type | `NiagaraDataChannelVariableBuffer.h:52` |
| Every prepared access is ended | `NiagaraDataChannelData.cpp:704` |
| Unsupported property in binding | `checkf` `NiagaraDataInterfaceDataChannelCommon.cpp:93`, hash forced to 0 `:94` |
| Unsupported property in game data write | `checkf` `NiagaraDataChannelGameData.cpp:242` |
| Layout identity between channel and game data | `check` `NiagaraDataChannelGameData.cpp:252` |
| Spawn bucket table integrity | `check` `NiagaraDataInterfaceDataChannelRead.cpp:900` |
| `SpawnDirect` requires one instance | `check` `...Read.cpp:1927` |
| Read DI proxy released before destruction | `check` `...Read.cpp:327-331` |
| Context and entry struct present | `check` `NiagaraDataChannelAccessContext.h:223`, `NiagaraDataChannel_Map.h:113` |
| Register binding packing | `check` `NiagaraDataInterfaceDataChannelCommon.h:71-72` |
| Writer/reader not re-entered | `ensure` `NiagaraDataChannelAccessorImpl.h:18,64` |
| Write DI instance count consistency | `ensure` + warning `NiagaraDataInterfaceDataChannelWrite.cpp:455-468` |
| Island pool exhaustion | `ensure` `NiagaraDataChannel_Islands.cpp:258,378-381`, `UE_LOGF` error `:295` |
| Missing required accessor variables | `ensureMsgf`/`ensureAlwaysMsgf` `NiagaraDataChannelAccessor.cpp:385-392` |
| Legacy `FindData` misuse | error log + `nullptr` `NiagaraDataChannelHandler.cpp:231-237` |
| Blueprint node out of sync | `K2Node_DataChannelBase.cpp:85-89,93-97` |

## Console variables

| CVar | Default | Anchor |
|---|---|---|
| `fx.Niagara.DataChannels.Enabled` | 1, forced false in the cook commandlet | `NiagaraModule.cpp:158-175,330-334` |
| `fx.Niagara.DataChannels.WarnOnLateWrites` | `!UE_BUILD_SHIPPING` | `NiagaraDataChannelData.cpp:22` |
| `fx.Niagara.DataChannels.AutoUploadGPUSpawnData` | true | `:25` |
| `fx.Niagara.DataChannels.EnableAccumulatedWrites` | true | `:28` |
| `fx.Niagara.Rollback.DataChannels.RetainGPUDataIfPendingCpuUpdate` | `ECVF_RenderThreadSafe` | `:31-42` |
| `fx.Niagara.DataChannels.AllowLazyHandlerInit` | 1 | `NiagaraDataChannelManager.cpp:16-22` |
| `fx.Niagara.DataChannels.MissingNDCAccessorMode` | 0 silent, 1 warning, 2 ensure once, 3 ensure always; default 1 when `DEBUG_NDC_ACCESS` is on, 0 when it is off | `NiagaraDataChannelAccessor.cpp:20-32` |
| `fx.Niagara.DataChannels.LogWritesToOutputLog` | 0 off, 1 summary, 2 also data | `NiagaraDataChannelDebugUtilities.cpp:23-24` |
| `fx.Niagara.DataChannels.FrameDataToCapture` | rolling window size | `:26-27` |
| `fx.Niagara.DataChannels.DumpWriteLog` (command) | dumps captured writes | `:29-34` |
| `fx.Niagara.DataChannels.ResetLayoutInfo` (command) | clears the shared layout cache | `NiagaraDataInterfaceDataChannelCommon.cpp:13-22` |
| `fx.Niagara.DataChannels.DumpHandlerTick` | 0 | `NiagaraDataChannel_Global.cpp:11-17` |
| `fx.Niagara.DataChannels.DebugDumpWriterDI` | 0 | `NiagaraDataInterfaceDataChannelWrite.cpp:51-56` |
| `fx.Niagara.DataChannels.WriteDIZeroCPUBuffersMode` | 1 (zero only on `Write`); 0 none, 2 always | `:58-64` |
| `fx.Niagara.DataChannels.ForceReadTickGroup` (`GNDCReadForceTG`) | `-1` = off; forces read DIs into the given tick group when `>= 0` | `NiagaraDataInterfaceDataChannelRead.cpp:50-56` |
| `fx.Niagara.DataChannels.ForceReadPrevFrame` (`GNDCReadForcePrevFrame`) | `false`; forces previous-frame reads | `:58-64`, used at `:915,1385,1595,1680,1821,1952,2029` |

## Profiling hooks

Stats group `STATGROUP_NiagaraDataChannels` (`NiagaraDataChannel.h:42`); cycle counters `STAT_DataChannelManager_BeginFrame/EndFrame/Tick` (`NiagaraDataChannelManager.cpp:11-13`), `STAT_NDIDataChannelRead_Tick` `:738` and `_PostTick` `:762`, `STAT_NDIDataChannelWrite_Write/Append/Tick/PostTick/PreStageTick/PostStageTick` (`...Write.cpp:43-48`), `STAT_DataChannelHandler_Global_Tick` (`NiagaraDataChannel_Global.cpp:19`); CSV stat `CSV_SCOPED_TIMING_STAT(Particles, CoreSystems_NiagaraDataChannelManager)` (`NiagaraDataChannelManager.cpp:79-106`); Insights hook `FNiagaraInsights::NotifyDataChannelPublish` (`NiagaraDataChannelData.cpp:385`).

## Manual verification recipe

1. Create an asset, add variables, confirm `IsValid()` requires at least one variable and matching CPU and GPU layouts (`NiagaraDataChannel.cpp:169-172`).
2. Write from Blueprint or C++ with `DebugSource` set, then `fx.Niagara.DataChannels.LogWritesToOutputLog 2` and `fx.Niagara.DataChannels.DumpWriteLog` (`NiagaraDataChannelDebugUtilities.cpp:23-34`).
3. Read the same frame in the SimCache visualizer: element index column, per-variable columns, visibility header (`FNiagaraDataChannelSimCacheVisualizer.cpp:146-205`); unsupported types show "Unsupported Type" (`:81`).
4. For Islands, enable `DebugDrawSettings` bounds (`NiagaraDataChannel_Islands.h:179`, `ShowBounds()` `:105`, `DrawDebugBox` red `...Islands.cpp:512-518`).
5. For accessor drift, set `fx.Niagara.DataChannels.MissingNDCAccessorMode 3` in a non-shipping build (`NiagaraDataChannelAccessor.cpp:20-32`) and rename a variable.
6. Confirm the frame order by checking that `ConsumePublishRequests` runs at `TG_LastDemotable` in `EndFrame` (`NiagaraDataChannelData.cpp:147`) and that the read DI lands at `FinalWriteTickGroup + 1` when ordering is enforced (`...Read.cpp:1041-1053`).
