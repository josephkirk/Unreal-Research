# 10 — Tick Order and Frame Integration

NDC has its own place in the Niagara frame loop, and read/write ordering is opt-in. The header comment at `NiagaraDataChannel.h:20-25` states the intended order: game-level data is authored in LWC AoS, converted to SoA for Niagara, and writes are consumed at a defined tick group so a reader in a later tick group sees the same frame's data.

## Frame hooks

| Phase | Anchor | What happens |
|---|---|---|
| Manager construction | `NiagaraWorldManager.cpp:422` | `DataChannelManager(new FNiagaraDataChannelManager(this))` |
| `OnPostWorldInit()` | `:493` | `DataChannelManager->Init()`; `AddReferencedObjects` keeps channels alive `:612` |
| `BeginFrame(DeltaSeconds)` | `:1339`, inside `TickStart()` (`:1319`) | called **before** the `DataManagers` loop; `check(IsInGameThread())`, gated by `INiagaraModule::DataChannelsEnabled()`, `CSV_SCOPED_TIMING_STAT(Particles, CoreSystems_NiagaraDataChannelManager)`, prunes dead channels (`NiagaraDataChannelManager.cpp:79-106`) |
| `ExecutePreTickWork(ETickingGroup, float)` | `NiagaraWorldManager.cpp:1673-1680` | `DataChannelManager->Tick(DeltaSeconds, TickGroup)` at `:1680`, before `DataManagers` |
| `EndFrame(DeltaSeconds)` | `:1391`, inside `TickEnd()` (`:1359`) | after the `HandleCSVStats` call (`:1389`); consumes remaining requests at `TG_LastDemotable` (`NiagaraDataChannelData.cpp:147`) |
| Cleanup paths | `:845,943,958,1004` | `DataChannelManager->Cleanup()` from `CleanupTransientObjectsAndManagers()` (`:837`), `LegacyOnWorldBeginTearDown()` (`:936`), `LegacyOnWorldCleanup()` (`:948`), `LegacyOnPostWorldCleanup()` (`:997`); handler cleanup prunes weak data and `MarkAsGarbage()` (`NiagaraDataChannelHandler.cpp:51`) |
| `OnComputeDispatchInterfaceDestroyed_Internal` | `:828-835` (call at `:830`; wrapper `OnComputeDispatchInterfaceDestroyed` `:1224-1234`) | proxy teardown (`NiagaraDataChannelManager.cpp:226-233`, handler `:203-212`) |

Per tick group, `Tick` only records `CurrentTG` (`NiagaraDataChannelHandler.cpp:73-76`, exposed as `GetCurrentTickGroup()` `:89`); the base `BeginFrame` sets `CurrentTG = TG_PrePhysics` (`:57`) and prunes `WeakDataArray` (`:59-64`); the base `EndFrame` is empty (`:68-71`). Concrete handlers do the work: Global `Tick` → `ConsumePublishRequests(this, TickGroup)` (`NiagaraDataChannel_Global.cpp:71-79`, `:77`), Map `Tick` `NiagaraDataChannel_Map.cpp:135-143`, Islands `BeginFrame` `:66-123` releases entries.

## Ordering rules

`ShouldEnforceTickGroupReadWriteOrder()` (`NiagaraDataChannel.h:93`, `bEnforceTickGroupReadWriteOrder = false` default `:117`) and `GetFinalWriteTickGroup()` (`:96`, `FinalWriteTickGroup = TG_EndPhysics` `:121`) drive the read DI: `HasTickGroupPrereqs` uses `GNDCReadForceTG` (CVar `fx.Niagara.DataChannels.ForceReadTickGroup` `:50-56`, default `-1`) else the channel setting (`NiagaraDataInterfaceDataChannelRead.cpp:1028-1039`), and `CalculateTickGroup` returns `GetFinalWriteTickGroup() + 1` when enforced, otherwise `NiagaraFirstTickGroup` (`:1041-1053`). The read DI also defaults to previous-frame reads (`bReadCurrentFrame = false` `NiagaraDataInterfaceDataChannelRead.h:103-104`), overridden by `fx.Niagara.DataChannels.ForceReadPrevFrame` (`...Read.cpp:58-64`).

Writes land through publish requests: the write DI publishes at post tick (`NiagaraDataInterfaceDataChannelWrite.h:4-17`, `...Write.cpp:473-507`), and `ConsumePublishRequests` warns when `PublishSourceTG > FinalWriteTG` (`NiagaraDataChannelData.cpp:224-238`) under `fx.Niagara.DataChannels.WarnOnLateWrites`, default `!UE_BUILD_SHIPPING` (`:22`). The Blueprint single-element functions document the same rule: an element is not immediately visible, the earliest read point is the next tick group (`NiagaraDataChannelFunctionLibrary.h:123`).

`KeepPreviousFrameData()` (`NiagaraDataChannel.h:67`, default `true` `:113`) is what makes previous-frame reads possible: `BeginFrame` sets `PrevNumElements = bKeepPrevious ? NumElements : 0` (`NiagaraDataChannelGameData.cpp:43-52`, `:45`), the proxy swaps `PrevFrameData`/`CurrFrameData` (`NiagaraDataChannelData.cpp:644-647`) and `PrepareForReadAccess` takes the previous-frame path when reading the previous frame (`:835-838`).

## Rollback and pending CPU updates

`fx.Niagara.Rollback.DataChannels.RetainGPUDataIfPendingCpuUpdate` (`NiagaraDataChannelData.cpp:31-42`, `ECVF_RenderThreadSafe`) makes the proxy return early instead of swapping frames when a CPU update is pending (`:624-640`), so a rollback does not discard GPU data that the game thread has not yet consumed. `bHasPendingCpuUpdate` (`NiagaraDataChannelData.h:34`) and `PendingCPUBuffers` (`:40`) are the state that gates it.

## What a tech artist must know

- If a reader sees data one frame late, that is the default: `bReadCurrentFrame` is false (`NiagaraDataInterfaceDataChannelRead.h:103-104`) and the tradeoff is documented at `:92-102`.
- Enabling tick-order enforcement moves the reader DI to `FinalWriteTickGroup + 1` (`...Read.cpp:1041-1053`); with the default `TG_EndPhysics` (`NiagaraDataChannel.h:121`) that is later than `NiagaraFirstTickGroup`.
- The whole manager cycle is skipped when `fx.Niagara.DataChannels.Enabled` is false (`NiagaraDataChannelManager.cpp:79-106`), and handlers are cleaned up on the disabled branch (`:155-163`).
- Handlers are never created on dedicated servers (`InitDataChannel` skips `NM_DedicatedServer`, `NiagaraDataChannelManager.cpp:180-205`, `:183`).
