# 14 — Testing

There are **no visual or pixel-comparison tests for UI** in UE 5.8. Automation coverage is limited to data-structure and window-zone logic. Every claim below is a test that exists in the source.

## Existing coverage

| Area | Test | Anchor |
|---|---|---|
| Window zones | `Slate.Window.GetCurrentWindowZone.NotInWindow` | `Runtime/Slate/Private/Test/SWindowZoneTest.cpp:184` |
| | `...WindowedFullscreen` | `220` |
| | `...NonResizable` | `283` |
| | `...UserSizedBorders` | `351` |
| Cursor lock | `Slate.User.CursorLock.Borderless` | `Runtime/Slate/Private/Test/SlateUserCursorLockTest.cpp:490` |
| | `Slate.User.CursorLock.Bordered` | `497` |
| SlateCore data | `Slate.Debugger.ContentFilter` | `Runtime/SlateCore/Private/Test/ConsoleSlateDebuggerWindowFilterTest.cpp:74` |
| | `Slate.ObservableArray` | `.../ObservableArrayTest.cpp:13` |
| | `Slate.Attribute` | `.../SlateAttributeTest.cpp:18` |
| | `Slate.Children.ForEach` | `.../SlateChildrenTest.cpp:18` |
| | `Slate.InvalidationWidgetList.AddBuildRemove` | `.../SlateInvalidationWidgetListTest.cpp:18` |
| | `Slate.ParsingInvalidationWidgetReason` | `.../SlateInvalidationWidgetReasonLexTest.cpp:12` |
| UMG widget state | `Slate.WidgetState.OperatorBoolTest` | `Runtime/UMG/Private/Binding/States/Tests/WidgetStateBitfieldTests.cpp:29` |
| | `...OperatorBitwiseAndTest` | `56` |
| | `...OperatorBitwiseOrTest` | `96` |
| | `...OperatorBitwiseNotTest` | `127` |
| | `...HasFlagTest` | `151` |
| MVVM | `System.Plugins.MVVM.BindingExecution` | `Plugins/Runtime/ModelViewViewModel/Source/ModelViewViewModelEditor/Private/Tests/MVVBindingExecuteTest.cpp:126` |
| | `System.Plugins.MVVM.BindingHelper` | `.../MVVMBindingHelperTest.cpp:17` |
| | `System.Plugins.MVVM.GetNotifyBindingInfoFromFieldPath` | `.../MVVMFieldPathHelperTest.cpp:18` |
| | `System.Plugins.MVVM.FieldValueChanged` | `.../MVVMFieldValueChangedTest.cpp:16` |

UMG's only automation tests are the widget-state bitfield tests above. `Editor/UMGEditor`, `Editor/PropertyEditor`, `Developer/ToolMenus`, and `Runtime/SlateRHIRenderer` have none.

## What is testable without a running application

- Visibility flag semantics (`SlateCore/Public/Layout/Visibility.h:11`, encoding `103`) — pure data.
- Invalidation reason parsing (`InvalidateWidgetReason.h:13`, test at `:12`).
- Fast-path list build/remove (`Private/FastUpdate/SlateInvalidationWidgetList.h:19`, test at `SlateInvalidationWidgetListTest.cpp:18`).
- Widget state bitfields (`WidgetStateBitfieldTests.cpp`).
- MVVM field path resolution and binding execution (`MVVMFieldPathHelperTest.cpp:18`, `MVVBindingExecuteTest.cpp:126`).

## What is not

Paint output, batching, atlas behavior, layout geometry, focus transitions, and hit-test results all require a live `FSlateApplication`. The cursor-lock test already depends on it (`SlateUserCursorLockTest.cpp:340` `FSlateApplication::Get().GetCursorUser()`), so it cannot run headless. Test flags are `EditorContext`/`EngineFilter` (e.g. `ObservableArrayTest.cpp:13`), so these run in the editor automation driver, not in a game build.

## Practical guidance

1. Filter by prefix: `Slate.`, `Slate.WidgetState.`, `Slate.Window.`, `System.Plugins.MVVM.`.
2. For new UI logic, put the decision in a plain function (state → reply/visibility), test that, and keep `OnPaint` thin. `OnPaint` itself is not testable without a paint context (`SWidget.h:1771`).
3. Verify visual changes by running the actual tool window or widget blueprint, not by asserting on element lists.
