# 13 — File Reference

All paths relative to `G:/Epic/UE_5.8/Engine/Plugins/FX/Niagara/`. Line counts from `wc -l`; `Intermediate/.../UHT/*.generated.h` excluded. `Engine/Source/Runtime/Engine/Private/DataChannel.cpp` is the network `UControlChannel` and is **not** part of this corpus.

## Runtime — public

| File | Lines | Content |
|---|---|---|
| `Source/Niagara/Public/NiagaraDataChannel.h` | 168 | abstract asset base, stats group, layout, handler factory |
| `Source/Niagara/Public/NiagaraDataChannelAsset.h` | 37 | `UNiagaraDataChannelAsset` |
| `Source/Niagara/Public/NiagaraDataChannelPublic.h` | 19 | public umbrella |
| `Source/Niagara/Public/NiagaraDataChannelCommon.h` | 67 | shared-ptr aliases, update context, dynamic delegate |
| `Source/Niagara/Public/NiagaraDataChannelVariable.h` | 34 | variable definition |
| `Source/Niagara/Public/NiagaraDataChannelVariableBuffer.h` | 100 | AoS byte buffers, LWC special case |
| `Source/Niagara/Public/NiagaraDataChannelLayoutInfo.h` | 51 | game data layout, compiled data |
| `Source/Niagara/Public/NiagaraDataChannelGameData.h` | 79 | game data, write/read templates |
| `Source/Niagara/Public/NiagaraDataChannelPublishRequest.h` | 51 | publish request |
| `Source/Niagara/Public/NiagaraDataChannelData.h` | 197 | data object + render-thread proxy |
| `Source/Niagara/Public/NiagaraDataChannelHandler.h` | 163 | handler base, subscriptions |
| `Source/Niagara/Public/NiagaraDataChannelAccessContext.h` | 276 | context hierarchy |
| `Source/Niagara/Public/NiagaraDataChannelAccessContextImpl.h` | 28 | spawned-system iteration |
| `Source/Niagara/Public/NiagaraDataChannelReference.h` | 36 | asset + context reference |
| `Source/Niagara/Public/NiagaraDataChannelAccessor.h` | 535 | reader/writer objects, accessor bases, macros |
| `Source/Niagara/Public/NiagaraDataChannelAccessorImpl.h` | 98 | `BeginWrite_Internal`, `BeginRead_Internal` |
| `Source/Niagara/Public/NiagaraDataChannelFunctionLibrary.h` | 276 | Blueprint function library |
| `Source/Niagara/Public/NiagaraDataChannelDebugUtilities.h` | 48 | debug singleton |
| `Source/Niagara/Public/NiagaraDataChannel_Global.h` | 52 | global channel + handler |
| `Source/Niagara/Public/NiagaraDataChannel_Islands.h` | 219 | island channel, island struct, handler |
| `Source/Niagara/Public/NiagaraDataChannel_Map.h` | 283 | map base, key writer, entry, handler |
| `Source/Niagara/Public/NiagaraDataChannel_GameplayBurst.h` | 175 | gameplay burst channel, context, entry, handler |

## Runtime — private

| File | Lines |
|---|---|
| `Source/Niagara/Private/NiagaraDataChannel.cpp` | 173 |
| `Source/Niagara/Private/NiagaraDataChannelAsset.cpp` | 37 |
| `Source/Niagara/Private/NiagaraDataChannelCommon.cpp` | 5 |
| `Source/Niagara/Private/NiagaraDataChannelVariable.cpp` | 52 |
| `Source/Niagara/Private/NiagaraDataChannelLayoutInfo.cpp` | 56 |
| `Source/Niagara/Private/NiagaraDataChannelGameData.cpp` | 411 |
| `Source/Niagara/Private/NiagaraDataChannelData.cpp` | 919 |
| `Source/Niagara/Private/NiagaraDataChannelHandler.cpp` | 259 |
| `Source/Niagara/Private/NiagaraDataChannelManager.h` | 58 (manager header is private) |
| `Source/Niagara/Private/NiagaraDataChannelManager.cpp` | 232 |
| `Source/Niagara/Private/NiagaraDataChannelAccessContext.cpp` | 25 |
| `Source/Niagara/Private/NiagaraDataChannelReference.cpp` | 102 |
| `Source/Niagara/Private/NiagaraDataChannelAccessor.cpp` | 445 |
| `Source/Niagara/Private/NiagaraDataChannelFunctionLibrary.cpp` | 411 |
| `Source/Niagara/Private/NiagaraDataChannelDebugUtilities.cpp` | 295 |
| `Source/Niagara/Private/NiagaraDataChannel_Global.cpp` | 89 |
| `Source/Niagara/Private/NiagaraDataChannel_Islands.cpp` | 518 |
| `Source/Niagara/Private/NiagaraDataChannel_Map.cpp` | 418 |
| `Source/Niagara/Private/NiagaraDataChannel_GameplayBurst.cpp` | 286 |

## Data interfaces

| File | Lines |
|---|---|
| `Source/Niagara/Internal/DataInterface/NiagaraDataInterfaceDataChannelCommon.h` | 475 |
| `Source/Niagara/Internal/DataInterface/NiagaraDataInterfaceDataChannelRead.h` | 377 |
| `Source/Niagara/Internal/DataInterface/NiagaraDataInterfaceDataChannelWrite.h` | 298 |
| `Source/Niagara/Private/DataInterface/NiagaraDataInterfaceDataChannelCommon.cpp` | 721 |
| `Source/Niagara/Private/DataInterface/NiagaraDataInterfaceDataChannelRead.cpp` | 2348 |
| `Source/Niagara/Private/DataInterface/NiagaraDataInterfaceDataChannelWrite.cpp` | 1872 |

## Shaders

| File | Lines |
|---|---|
| `Shaders/Private/DataChannel/NiagaraDataInterfaceDataChannelCommon.ush` | 241 |
| `Shaders/Private/DataChannel/NiagaraDataInterfaceDataChannelTemplateCommon.ush` | 37 |
| `Shaders/Private/DataChannel/NiagaraDataInterfaceDataChannelTemplateReadCommon.ush` | 267 |
| `Shaders/Private/DataChannel/NiagaraDataInterfaceDataChannelTemplateWriteCommon.ush` | 164 |
| `Shaders/Private/DataChannel/NiagaraDataInterfaceDataChannelTemplate_Append.ush` | 24 |
| `Shaders/Private/DataChannel/NiagaraDataInterfaceDataChannelTemplate_Consume.ush` | 31 |
| `Shaders/Private/DataChannel/NiagaraDataInterfaceDataChannelTemplate_Read.ush` | 26 |
| `Shaders/Private/DataChannel/NiagaraDataInterfaceDataChannelTemplate_Write.ush` | 14 |

## Blueprint nodes

| File | Lines |
|---|---|
| `Source/NiagaraBlueprintNodes/Internal/K2Node_DataChannelBase.h` | 64 |
| `Source/NiagaraBlueprintNodes/Internal/K2Node_ReadDataChannel.h` | 30 |
| `Source/NiagaraBlueprintNodes/Internal/K2Node_WriteDataChannel.h` | 29 |
| `Source/NiagaraBlueprintNodes/Internal/K2Node_DataChannel_WithContext.h` | 229 |
| `Source/NiagaraBlueprintNodes/Private/K2Node_DataChannelBase.cpp` | 266 |
| `Source/NiagaraBlueprintNodes/Private/K2Node_ReadDataChannel.cpp` | 256 |
| `Source/NiagaraBlueprintNodes/Private/K2Node_WriteDataChannel.cpp` | 206 |
| `Source/NiagaraBlueprintNodes/Private/K2Node_DataChannel_WithContext.cpp` | 560 |

## Editor

| File | Lines |
|---|---|
| `Source/NiagaraEditor/Public/NiagaraDataChannelAssetFactoryNew.h` | 20 |
| `Source/NiagaraEditor/Private/NiagaraDataChannelAssetFactoryNew.cpp` | 31 |
| `Source/NiagaraEditor/Private/AssetDefinitions/AssetDefinition_NiagaraDataChannel.h` | 21 |
| `Source/NiagaraEditor/Private/AssetDefinitions/AssetDefinition_NiagaraDataChannel.cpp` | 66 |
| `Source/NiagaraEditor/Private/Customizations/NiagaraDataChannelDetails.h` | 20 |
| `Source/NiagaraEditor/Private/Customizations/NiagaraDataChannelDetails.cpp` | 136 |
| `Source/NiagaraEditor/Private/Customizations/SimCache/FNiagaraDataChannelSimCacheVisualizer.h` | 16 |
| `Source/NiagaraEditor/Private/Customizations/SimCache/FNiagaraDataChannelSimCacheVisualizer.cpp` | 312 |
| `Source/NiagaraEditor/Private/Widgets/DataChannel/NiagaraDataChannelWizard.h` | 182 |
| `Source/NiagaraEditor/Private/Widgets/DataChannel/NiagaraDataChannelWizard.cpp` | 1293 |
| `Source/NiagaraEditorWidgets/Private/DetailCustomizations/NiagaraDataChannelDetails.h` | 47 |
| `Source/NiagaraEditorWidgets/Private/DetailCustomizations/NiagaraDataChannelDetails.cpp` | 338 |

## Graphify facts

Scan root `UE_5.8`, graphify 0.9.80, graph at `G:/Projects/SkyHunters/graphify-out/graph.json` (2,688,593 nodes, 5,061,968 links; `GRAPHIFY_MAX_GRAPH_BYTES=6GB`).

NDC corpus excluding the network `DataChannel.cpp`: **1,927 nodes**.

| Module | Nodes |
|---|---|
| `Niagara:Niagara` | 1,228 |
| `Plugin:FX/Niagara` | 260 |
| `Niagara:NiagaraBlueprintNodes` | 163 |
| `Niagara:NiagaraEditor` | 130 |
| `Niagara:NiagaraEditorWidgets` | 34 |
| `Runtime:Runtime` | 45 |
| `Plugin:Experimental/PCGInterops` | 32 |
| `Runtime:ThirdParty` | 14 |
| `Plugin:Media/PixelStreaming` | 12 |
| `Plugin:FX/NiagaraInsights` | 7 |
| `Plugin:Media/PixelStreaming2` | 2 |

Top-degree NDC nodes: `NiagaraDataInterfaceDataChannelRead.cpp` 59, `FNiagaraDataChannelData` 55, `UNiagaraDataChannelHandler` 51, `UNiagaraDataInterfaceDataChannelWrite` 50, `UNiagaraDataInterfaceDataChannelRead` 48, `NiagaraDataInterfaceDataChannelWrite.cpp` 47, `UNiagaraDataChannelLibrary` 45, `NiagaraDataChannelAccessor.cpp` 44, `K2Node_DataChannel_WithContext.cpp` 40, `FNiagaraDataChannelDataProxy` 38, `NiagaraDataChannelCommon.h` 36, `FSelectVariablesPageBase` (wizard) 35, `UK2Node_DataChannelBase` 34.

Cross-module edges from NDC nodes: `Runtime:Runtime` 447, `Plugin:Experimental/PlainProps` 90, `Niagara:Niagara` 32, `Runtime:Editor` 29, `Runtime:ThirdParty` 18, `Plugin:Runtime/OpenCV` 18, `Niagara:NiagaraBlueprintNodes` 17, `Plugin:VirtualProduction/VirtualCameraCore` 14, `Plugin:Runtime/GeometryProcessing` 11, `Niagara:NiagaraEditor` 9, `Plugin:Media/TimecodeSynchronizer` 6, `Plugin:Experimental/PCGInterops` 3.

Shortest path `FNiagaraDataChannelDataProxy` → `FRDGBuilder` (3 hops): proxy → `NiagaraDataChannelCommon.h` → `NiagaraDataChannelData.h` → `FRDGBuilder`; alternate proxy → `NiagaraGpuComputeDispatchInterface.h` → `FRDGBuilder`. Shader `.ush` files are not graphify nodes (0 shader nodes), so shader facts come from direct reads only. Niagara plugin totals: 44,024 nodes (Niagara 20,514; NiagaraEditor 20,308; NiagaraEditorWidgets 2,004; NiagaraShader 609; NiagaraVertexFactories 231; NiagaraBlueprintNodes 191; NiagaraAnimNotifies 87; NiagaraCore 80).

Mass ↔ Niagara: 2 edges, both `references` → `FGraphEventArray` from `Engine/Source/Runtime/MassEntity/Private/MassProcessor.cpp` (`UMassProcessor::DispatchProcessorTasks()`, `UMassCompositeProcessor::DispatchProcessorTasks()`).
