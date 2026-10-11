# 03 — Slate Widget Catalog and Theming

## Catalog

112 `.h` files under `Runtime/Slate/Public/Widgets/` (counted by `find`; one `.inl` and one stray `.cpp` in `Layout/` are not headers). Categories and the widgets that matter for tool work:

| Category | Headers | Notable |
|---|---|---|
| Input | 33 | `SButton`, `SCheckBox`, `SSlider`, `SSpinBox`, `SNumericEntryBox`, `SEditableText`, `SEditableTextBox`, `SComboBox`, `SSearchBox`, `SSegmentedControl`, `SComboButton`, `SMenuAnchor`, `SInputKeySelector`, `SVectorInputBox`, `SRotatorInputBox` |
| Layout | 33 | `SBox`, `SBoxPanel` (H/V), `SGridPanel`, `SUniformGridPanel`, `SWrapBox`, `SScrollBox`, `SSplitter`, `SSafeZone`, `SScaleBox`, `SDPIScaler`, `SConstraintCanvas`, `SBackgroundBlur`, `SBorder`, `SExpandableArea`, `SWidgetSwitcher`, `SPopup`, `SMenuOwner`, `Anchors` |
| Text | 9 | `STextBlock`, `SRichTextBlock`, `SMultiLineEditableText`, `SInlineEditableTextBlock`, `SlateTextBlockLayout`, `SlateEditableTextLayout`, `ISlateEditableTextWidget` |
| Views | 9 | `STableViewBase`, `SListView`, `STreeView`, `STileView`, `STableRow`, `IItemsSource`, `ITableRow`, `SHeaderRow` |
| Notifications | 8 | `SNotificationList`, `GlobalNotification`, `INotificationWidget`, `SErrorText`, `SErrorHint` |
| Root/misc | 7 | `SCanvas`, `SInvalidationPanel`, `SToolTip`, `SDeferredToolTip`, `SViewport`, `SVirtualWindow`, `SWeakWidget` |
| Colors | 6 | `SColorWheel`, `SColorSpectrum`, `SColorGradingWheel`, `SComplexGradient`, `SSimpleGradient`, `SColorBlock` |
| LayerManager | 2 | `SLayerManager`, `STooltipPresenter` |
| Images | 2 | `SThrobber`, `SSpinningImage` |
| Docking | 1 | `SDockTab` |
| Accessibility | 1 | `SlateAccessibleWidgets` |
| Navigation | 1 | `SBreadcrumbTrail` |

`SCompoundWidget` lives in SlateCore (`Public/Widgets/SCompoundWidget.h:21`), not in the widget library — that is what a plugin subclasses.

Containers default to `SelfHitTestInvisible` (`SBoxPanel.h:278`, `432`, `543`; `SOverlay.h:80`): visible, painted, but clicks pass through to children.

## Style system

| Object | Anchor | Role |
|---|---|---|
| `ISlateStyle` | `Styling/ISlateStyle.h:17` | Interface: `GetStyleSetName` (`28`), `GetResources` (`34`), `GetContentRootDir` (`35`), `GetEntriesUsingBrush` (`44`), `GetWidgetStyle<T>` template (`51-62`) |
| `FSlateStyleSet` | `Styling/SlateStyle.h:27` | Concrete set; `GetWidgetStyleInternal` at `206` |
| `FSlateStyleRegistry` | `Styling/SlateStyleRegistry.h:13` | `RegisterSlateStyle` (`22`), `UnRegisterSlateStyle` (`29`, `36`), `FindSlateStyle` (`44`) |
| `FCoreStyle` | `Styling/CoreStyle.h:18` | `Create()` builds the default set; `FSlateApplication::InitializeCoreStyle` (`SlateApplication.h:301`) |
| `FWidgetStyle` | `Styling/WidgetStyle.h:14` | Per-widget style context passed to `OnPaint` (`SWidget.h:1771`) |
| `FSlateBrush` | `Styling/SlateBrush.h:235` | `USTRUCT(BlueprintType)` — drawable resource reference |
| `FSlateColor` | `Styling/SlateColor.h:41` | `USTRUCT(BlueprintType)` — literal or style-set lookup (`38`) |
| `SlateTypes.h` | `Styling/SlateTypes.h` | Widget style structs (`FButtonStyle`, `FTextBlockStyle`, …). 358 graph nodes, degree 540 — the most connected header in the UI corpus |

Style resolution order at paint time: the widget's own `TAttribute`/`FSlateColor` → the style set entry → `WidgetStyleType::GetDefault()` (`ISlateStyle.h:58`). A missing entry warns; it does not fail.

## What an artist can change without C++

- Style set entries (brushes, colors, widget styles) are data: `FSlateBrush` and `FSlateColor` are `BlueprintType` structs, so UMG widget properties expose them directly.
- UMG widget style properties (e.g. `UButton` style, `UTextBlock` font/color) are editable in the designer and are stored on the `UWidgetBlueprintGeneratedClass`.
- Fonts: `FSlateFontInfo` (`Fonts/SlateFontInfo.h:146`) is `BlueprintType` with a native make hook; font assets (`UFont`) resolve through `FSlateFontCache` (`Fonts/FontCache.h:756`).

What needs C++: registering a new style set (`FSlateStyleRegistry::RegisterSlateStyle`), custom widget classes, and any `OnPaint` behavior. See [06](06_Artist_vs_Programmer_Boundary.md).

## Theme and locale behavior

- Flow direction is per-widget (`SWidget.h:1356`), resolved from `EFlowDirectionPreference::Culture` via `FLayoutLocalization::GetLocalizedLayoutDirection()` (`1361`); `GSlateFlowDirection` is the global fallback (`1242`, `1368`). RTL affects mirroring of transforms (`1242-1265`) and layout order.
- DPI scaling is applied through `SDPIScaler` and `LayoutScaleMultiplier` in `ComputeDesiredSize` (`SWidget.h:774`); `Canvas`-based HUD text scales via `FDisplayDebugManager::GetTextScale()` (`Runtime/Engine/Private/UserInterface/Canvas.cpp:2345`).
- Accessibility: `#if WITH_ACCESSIBILITY` gates (`SWidget.h:1070`, `SlateApplicationBase.h:20-21`); `SlateAccessibleWidgets.h` is the widget-side surface.
