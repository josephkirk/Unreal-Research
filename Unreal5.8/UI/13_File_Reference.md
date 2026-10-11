# 13 — File Reference

Graph facts are computed on a scoped corpus (`graphify-out/graph.json`, 2,688,593 nodes / 5,061,968 links, `built_at_commit bbf056d8568cdd48992ca50f5b36a2b6896930bd`, scan root `UE_5.8`). Corpus regex:

```
Engine/Source/Runtime/(Slate|SlateCore|SlateRHIRenderer|UMG)/
| Engine/Source/Developer/ToolMenus/
| Engine/Plugins/Runtime/ModelViewViewModel/Source/
| Engine/Source/Editor/(PropertyEditor|AssetDefinition)/
```

Excludes `ThirdParty` and `Intermediate`. **Corpus: 35,119 nodes.** Only source files are nodes — `.uplugin`, `.Build.cs`, `.uasset`, and Widget Blueprint assets are not in the graph.

## Corpus by module

| Module | Nodes |
|---|---|
| `Runtime/Slate` | 10,963 |
| `Runtime/SlateCore` | 7,112 |
| `Editor/PropertyEditor` | 5,941 |
| `Runtime/UMG` | 5,193 |
| `Plugins/Runtime/ModelViewViewModel/Source` | 4,063 |
| `Runtime/SlateRHIRenderer` | 820 |
| `Developer/ToolMenus` | 811 |
| `Editor/AssetDefinition` | 216 |

Relation counts: references 34,650; contains 17,524; defines 14,174; calls 9,762; imports 6,096; method 3,939; inherits 1,154; implements 1.

## Most connected files

| File | Nodes | Degree |
|---|---|---|
| `Runtime/Slate/Private/Framework/Application/SlateApplication.cpp` | 353 | 1415 |
| `Editor/PropertyEditor/Private/PropertyHandleImpl.h` | 303 | 941 |
| `Runtime/Slate/Private/Framework/Docking/TabManager.cpp` | 184 | 687 |
| `Editor/PropertyEditor/Private/PropertyNode.h` | 236 | 649 |
| `Runtime/UMG/Private/UserWidget.cpp` | 185 | 572 |
| `Runtime/Slate/Private/Widgets/Text/SlateEditableTextLayout.cpp` | 174 | 565 |
| `Runtime/SlateCore/Public/Styling/SlateTypes.h` | 358 | 540 |
| `Editor/PropertyEditor/Private/SDetailsViewBase.h` | 217 | 539 |
| `Runtime/SlateCore/Public/Widgets/SWidget.cpp` | — | 534 |
| `Runtime/SlateCore/Public/Widgets/SWindow.h` | 273 | — |
| `Runtime/Slate/Public/Framework/Docking/TabManager.h` | 225 | — |

## Who consumes the UI layer

Incoming edges from outside the corpus (top consumers):

| Consumer | Incoming edges |
|---|---|
| `Plugins/FX/Niagara` | 1581 |
| `Editor/DetailCustomizations` | 1514 |
| `Editor/UnrealEd` | 1425 |
| `Editor/Persona` | 1242 |
| `Plugins/VirtualProduction/Avalanche` | 1241 |
| `Editor/Sequencer` | 1082 |
| `Plugins/VirtualProduction/DMX` | 1025 |
| `Plugins/Animation/ControlRig` | 908 |
| `Plugins/Runtime/nDisplay` | 782 |
| `Developer/TraceInsights` | 678 |
| `Editor/Kismet` | 665 |
| `Editor/GraphEditor` | 637 |
| `Plugins/Experimental/UAF` | 633 |
| `Editor/UMGEditor` | 617 |
| `Plugins/Runtime/RigVM` | 560 |

Unscoped BFS through this corpus routes through `Core`/`UnrealEd` hubs and is not meaningful; hub degrees above are the reliable signal.

## Hot symbols

| Symbol | Calls | Location |
|---|---|---|
| `ConstructComponent()` | 15 | `Slate/Public/Widgets/Input/SVectorInputBox.h:477` |
| `FRichTextLayoutMarshaller::AppendRunsForText()` | 13 | `SlateCore/.../RichTextLayoutMarshaller.cpp:135` |
| `FToolBarBuilder::AddComboButton()` | 12 | `Slate/Public/Framework/MultiBox/MultiBoxBuilder.cpp:753` |
| `FHittestGrid::FindFocusableWidget()` | 12 | `SlateCore/Private/Input/HittestGrid.cpp:366` |
| `MakeAssetPickerWithMenu()` | 11 | `PropertyEditor/Private/PropertyCustomizationHelpers.cpp:681` |
| `FSlateTextHighlightRunRenderer::OnPaint()` | 11 | `SlateCore/Private/.../SlateTextHighlightRunRenderer.cpp:12` |

## Directory inventory

`Runtime/SlateCore/Public/`: `Widgets` (+ `Images`, `Accessibility`), `Application`, `Debugging`, `FastUpdate`, `Fonts`, `Input`, `Layout`, `Navigation`, `Rendering`, `Styling`, `Textures`, `Types`/`Attributes`, `Sound`, `Brushes`, `Containers`, `Animation`, `Trace`. Core widget base files: `SWidget.h`, `SCompoundWidget.h`, `SLeafWidget.h`, `SPanel.h`, `SOverlay.h`, `SBoxPanel.h`, `SWindow.h`, `SNullWidget.h`, `SUserWidget.h`, `IToolTip.h`, `DeclarativeSyntaxSupport.h`, `InvalidateWidgetReason.h`, `WidgetPixelSnapping.h`, `SlateControlledConstruction.h`, `SWidgetUtils.h`.

`Runtime/Slate/Public/`: `Framework/Application`, `Framework/Docking`, `Framework/MultiBox`, `Widgets` (112 `.h` files, see [03](03_Slate_Widget_Catalog_and_Theming.md)).

`Runtime/UMG/Public/`: `Blueprint`, `Components`, `Animation`, `Slate` (`SObjectWidget.h`, `SRetainerWidget.h`, `SPostBufferUpdate.h`, `SWorldWidgetScreenLayer.h`, `SObjectTableRow.h`, `SMeshWidget`, `SlateVectorArtData`), `Extensions` (+ `UIComponents`), `FieldNotification`, `Binding/States`, `Editor/WidgetCompilerLog.h`, `UMGStyle.h`, `IUMGModule.h`.

`Editor/UMGEditor/Public/`: `WidgetBlueprint.h` (`218`), `WidgetBlueprintEditor.h` (`42`), `WidgetBlueprintEditorUtils.h` (`28`), `WidgetBlueprintOperationUtils.h` (`21`), `WidgetBlueprintToolMenuContext.h` (`12`), `WidgetReference.h` (`FWidgetHandle`, `13`), `UMGEditorModule.h` (`IUMGEditorModule`, `25`, implementing seven extensibility interfaces `27-33`), `IHasWidgetContextMenuExtensibility.h` (`10`), `UIComponentUtils.h` (`15`), `BlueprintModes/WidgetBlueprintApplicationMode.h` (`22`, `FWorkflowAllowedTabSet TabFactories` `43`), `Customizations/IBlueprintWidgetCustomizationExtender.h` (`14`), `Designer/SDesignerView.h` (`48`), `Templates/`, `IUMGDesigner.h`, `Preview/PreviewMode.h`.

`Editor/UMGEditor/Private/`: `Palette/SPaletteView.h` + `SPaletteViewModel.h`, `Library/SLibraryView.h`, `Navigation/SWidgetDesignerNavigation.h`, `Details/SWidgetDetailsView.h`, `Hierarchy/SHierarchyView.h`, `Preview/SWidgetPreview.h`, `DragDrop/SelectedWidgetDragDropOp.h`, `ToolPalette/WidgetEditorModeUILayer.h`, `TabFactory/` (10 tab summoners), `Graph/`, `Extensions/` (12 slot extensions), `Nodes/`, `Customizations/`, `Animation/UMGDetailKeyframeHandler.cpp`.

`Plugins/Runtime/ModelViewViewModel/Source/`: `ModelViewViewModel` (Runtime), `ModelViewViewModelBlueprint` (UncookedOnly), `ModelViewViewModelEditor` (Editor), `ModelViewViewModelDebugger`, `ModelViewViewModelDebuggerEditor`.
