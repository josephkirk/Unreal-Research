# 12 — Tech Artist Guide

## Creating and editing the asset

New assets come from `UNiagaraDataChannelAssetFactoryNew` (`Source/NiagaraEditor/Private/NiagaraDataChannelAssetFactoryNew.cpp`): `SupportedClass`, `bEditAfterNew`, `bCreateNew` (`:11-17`), `RF_Transactional` creation (`:19-26`), and `CanCreateNew()` returns `INiagaraModule::DataChannelsEnabled()` (`:28-30`) — with `fx.Niagara.DataChannels.Enabled` off, the factory is unavailable.

Asset definition (`AssetDefinition_NiagaraDataChannel.cpp`): colour style key `"NiagaraEditor.AssetColors.DataChannelDefinitions"` (`:15-18`), class (`:20-23`), category `EAssetCategoryPaths::FX` with an "Advanced" section (`:25-29`), and a context-menu extension "Mark Dependent Compilable Assets Dirty" registered at `EDelayedRegisterRunPhase::EndOfEngineInit` (`:41`) on `ExtendToolMenu_AssetContextMenu(UNiagaraDataChannelAsset::StaticClass())` (`:45`) in section `GetAssetActions` (`:47`), calling `FNiagaraEditorUtilities::MarkDependentCompilableAssetsDirty` (`:38`). Use it after editing a variable — `PostEditChangeProperty` regenerates `VersionGuid` with `FGuid::NewGuid()` (`NiagaraDataChannel.cpp:136`); the hash-derived deterministic guid only applies on first load (`PostLoad` `:53-65`).

Detail customization (`NiagaraDataChannelDetails.cpp`): single-object only (`:37-40`), one category `FName("Data Channel")` (`:48-49`) because the class uses `CLASS_CollapseCategories` (comment `:54-56`), the property `DataChannel` is renamed to "Data Channel Type" (`:51-52`), the type property is always added (`:59-60`), and a hyperlink row "Which data channel type should I choose?" points to `https://dev.epicgames.com/community/learning/tutorials/RJbm/unreal-engine-niagara-data-channels-intro` (`:61-81`, `LaunchURL` `:78`). A duplicate customization exists in `NiagaraEditorWidgets/Private/DetailCustomizations/`.

Editing variables triggers `PreEditChange` which removes the channel from every world manager (`NiagaraDataChannel.cpp:90-99`), and `PostEditChangeProperty` enforces unique names on `ArrayAdd`/`Duplicate` (`:101-151`) and rebuilds the layout (`LayoutInfo = nullptr` then `GetLayoutInfo()`, `:145-146`).

## Variable types

`FNiagaraDataChannelVariable::Serialize` re-fixes the type through `SetType(ToDataChannelType(GetType()))` on load because only SWC types are serialized (`NiagaraDataChannelVariable.cpp:7-20`).

| Rule | Anchor |
|---|---|
| Allowed types are editor-checked | `IsAllowedType` `:23-26` |
| Rejected | DataInterface types, `GetClass()`, ParameterMap, GenericNumeric, **Half**, Matrix4 (`:23-26`) |
| LWC mapping | `Vec3→Vector`, `Float→Double`, `Quat→Quat`, `Vec2→Vector2D`, `Vec4→Vector4` (`:29-52`) |
| Unique names | `EnforceUniqueNames` on `ChannelVariables` (`NiagaraDataChannel.h:109`) |
| Layout validity | requires `ChannelVariables.Num() > 0` and both CPU and GPU compiled layouts matching the variable count (`NiagaraDataChannel.cpp:169-172`) |

Half is rejected at the variable level and has no shader accessors (`Shaders/Private/DataChannel/NiagaraDataInterfaceDataChannelCommon.ush:99-134,207-238`), so an fp16 pipeline is not available.

## Wizard choices

`CreateNDCWizardGenerator` (`NiagaraDataChannelWizard.cpp:1255-1291`) offers "Read From Data Channel..." (`:1264-1268`), "Write To Data Channel..." (`:1270-1274`) and "Spawn From Data Channel..." only for `EmitterUpdateScript` (`:1276-1283`).

| Wizard | Model identifier | Module name | Anchor |
|---|---|---|---|
| Read | `ReadNDCWizardModel` | `Read {0}` | `:497-661`, header `:528`, `:533`, `:550`, usage `System \| Emitter \| Particle` `:557` |
| Write | `WriteNDCWizardModel` | `Write {0}` | `:663-851`, header `:693`, `:698`, `:715` |
| Spawn | `SpawnNDCWizardModel` | `Spawn From {0}` | `:853-1253`, pages `AssetPage`/`ConditionPage`/`VariablesPage` `:885-889`, `:876`, `:895`, creates emitter-spawn + emitter-update + particle-spawn modules `:898-913` |

Read module data (`NiagaraDataChannelWizard.h:22-51`): `DataChannel` `:29`, `bReadCurrentFrame` `:35`, `bUpdateSourceDataEveryTick` `:44`, `bAutoTransformPositionData` `:50`. Write module data (`:129-174`): `WriteMode` default `AppendNewElement` (`:140`, modes `:118-123`), `bVisibleToGame` displayed as "Visible to Blueprint" (`:143-144`), `bVisibleToCPU` `:147-148`, `bVisibleToGPU` `:151-152`, `AllocationMode` `Static` `:156`, `AllocationCount = 1` `:160`, `bUpdateDestinationDataEveryTick` `:167`, `bAutoTransformPositionData` `:173`. Spawn mode default `ConditionalSpawn` (`:88`, modes `:53-66`), `bModifySpawnCountByScalability` `:115`.

## SimCache inspection

`FNiagaraDataChannelSimCacheVisualizer` clamps `SimCacheViewModel->GetFrameIndex()` into `DataChannelData->FrameData` (`NiagaraDataChannelSimCacheVisualizer.cpp:14-18`), builds one column per `FrameData.VariableData` (`:157-173`) plus an `_ElementIndex` column (`:146-155`), and shows "This frame writes {0} elements to the data channel … [X] Visible to Blueprint / CPU / GPU" (`:192-205`). `GenerateWidgetForColumn` (`:252-301`) re-types Position to `Vector` through `GetLWCType` (`:269-276`), reads `SetData(Buffer.Data.GetData() + Buffer.Size * InstanceIndex)` (`:277`) and draws a colour swatch for `FLinearColor` (`:280-297`); unsupported types print "Unsupported Type" (`ChannelVariableToString` `:20-82`, `:81`).

## Blueprint node hygiene

Nodes store `DataChannel` as an asset reference (`K2Node_DataChannelBase.h:62`) and `DataChannelVersion` (`:47-50`); `PostLoad` reconstructs the node when the stored version differs from the asset and a valid blueprint exists (`K2Node_DataChannelBase.cpp:38-48`). Pins are matched by `PersistentGuid` (`:207-210`), which is the variable's `Version` (`K2Node_ReadDataChannel.cpp:44`, `K2Node_WriteDataChannel.cpp:40`). Renaming does **not** change that identity — `PostEditChangeProperty` deliberately leaves the variable guid alone on rename (`NiagaraDataChannel.cpp:137`); `Version` changes when the type is changed through the variable details customization (`NiagaraTypeCustomizations.cpp:3612`) or when the variable is duplicated (`NiagaraDataChannel.cpp:131`). `IgnoredVariables` (`K2Node_DataChannelBase.h:40`) lets an artist drop pins without deleting the variable. `ExpandNode` reports "Node does not have a valid data channel" (`:85-89`) or the out-of-sync message (`:93-97`).

## Checklist

1. Pick the channel type before adding variables — it determines the access context and the handler (`NiagaraDataChannel.h:70,76`).
2. Keep variable names unique (`NiagaraDataChannel.h:109`) and avoid rejected types (`NiagaraDataChannelVariable.cpp:23-26`).
3. Decide visibility per write (`bVisibleToGame`/`bVisibleToCPU`/`bVisibleToGPU`, `NiagaraDataChannelPublishRequest.h:23-29`); a GPU-only consumer cannot read a buffer that was never published to GPU (`NiagaraDataChannelData.cpp:157-164`).
4. Set `bKeepPreviousFrameData` only if a consumer reads the previous frame (`NiagaraDataChannel.h:67,113`; `NiagaraDataChannelGameData.cpp:52`).
5. Turn on `bEnforceTickGroupReadWriteOrder` when a reader must see the same frame's writes (`NiagaraDataChannel.h:93,117`), and remember the read DI then runs at `FinalWriteTickGroup + 1` (`NiagaraDataInterfaceDataChannelRead.cpp:1041-1053`).
6. Verify with the SimCache visualizer and `fx.Niagara.DataChannels.LogWritesToOutputLog` before shipping (`NiagaraDataChannelDebugUtilities.cpp:23-24`).
