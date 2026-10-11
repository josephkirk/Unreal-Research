# 15 — Testing

**No RDG-specific automated test suite exists in the directories searched** — `Engine/Source/Runtime/RenderCore`, `Engine/Plugins/Tests`, `Engine/Source/Developer`. The engine-wide search timed out, so this is a scoped result, not an engine-wide one. `RenderCore` contains no automation tests: a search for `AUTOMATION_TEST` there returns no matches, and no file in the module is named for tests. Graph correctness is enforced at runtime by validation (`RenderGraphValidation.h:40-150`, gated by `RDG_ENABLE_DEBUG` at `:24`), and that layer checks API inputs and lifecycle — `ValidateCreate*` (`:47-63`), `ValidateRegisterExternal*` (`:65-76`), `ValidateUploadBuffer` (`:78-82`), `ValidateCommitBuffer` (`:84`), `ValidateExtract*` (`:87-88`), `ValidateAddPass` (`:98-100`), `ValidateExecutePassBegin`/`End` (`:103-104`), `ValidateExecuteBegin`/`End` (`:107-108`) — not expected graph decisions or produced contents.

## Adjacent test surfaces

| Surface | What it covers | RDG involvement |
|---|---|---|
| `Engine/Plugins/Tests/RHITests/Source/RHITests/Private/RHITests.spec.cpp` | `FAutomationRHITest`, spec path `Rendering.RHI`, flags `EngineFilter \| ApplicationContextMask \| NonNullRHI` (`:18`); clear, formats, update, copy, readback, create buffer, draw, reserved resources, graphics UAV, allocators, bindless, root constants (`:22-269`) | Three test files include `RenderGraphUtils.h` (`RHIBindlessTests.cpp:6`, `RHIReadbackTests.cpp:7`, `RHIRootConstantsTests.cpp:13`), but none construct an `FRDGBuilder` — they call RHI directly through `RunOnRenderThreadSynchronous` |
| `Engine/Source/Developer/FunctionalTesting/Private/ScreenshotFunctionalTestBase.cpp` | Screenshot functional tests | Includes `RenderGraphBuilder.h` (`:21`); rendering goes through RDG, so graph regressions surface as image mismatches |
| `Engine/Source/Runtime/RenderCore/Private/RenderCommandStateStream.{h,cpp}` | State-stream records for command lists | The only RenderCore file matching a `*test*` search pattern; not a test suite |

The gap is structural: RDG behaviour depends on platform capabilities (`RHIHasTiledGPU`, `GSupportsEfficientAsyncCompute`, `GRHIMergeableAccessMask`), so a deterministic unit test of merge or culling decisions requires a mock RHI that the engine does not provide. `[INFERENCE]`

## What is verifiable without a test suite

Each of these is a runtime observation, not an assertion, and each isolates one graph decision:

| Decision | Verification | Anchor |
|---|---|---|
| Pass culling | `STAT_RDG_PassCullCount` non-zero for a pass expected to run; compare with `r.RDG.CullPasses 0` | `RenderGraphPrivate.h:166`, `RenderGraphBuilder.cpp:1403-1405`, `RenderGraphPrivate.cpp:233` |
| State merging | `STAT_RDG_ResourceTransitions` vs `STAT_RDG_TransitionBatches`; `r.RDG.TransitionLog 1` | `RenderGraphPrivate.h:178`, `:180`, `:77` |
| Render-pass merging | `STAT_RDG_RenderPassMergeCount` | `RenderGraphPrivate.h:167`, `RenderGraphBuilder.cpp:1492` |
| Lifetime correctness | `r.RDG.Debug.ExtendResourceLifetimes 1` removes reuse as a variable | `RenderGraphPrivate.cpp:43` |
| Transient path | `r.RDG.Debug.DisableTransientResources 1` forces pooled allocation | `RenderGraphPrivate.cpp:53` |
| Execution order | `r.RDG.ImmediateMode 1` restores creation order | `RenderGraphPrivate.cpp:18`, `RenderGraph.h:29-32` |
| Declared-but-unused references | `MarkResourceAsUsed` warnings | `RenderGraphResources.h:147-151`, `RenderGraphUtils.cpp:22-100` |
| Pass outputs | `r.RDG.ClobberResources 1` produces recognisable values | `RenderGraphPrivate.cpp:60`, `:168-204` |
| Dependency graph | `RDG_DUMP_GRAPH_PRODUCERS` / `RESOURCES` / `TRACKS` output | `RenderGraphPrivate.h:12-14` |
| Filtered reproduction | `r.RDG.Debug.GraphFilter` / `PassFilter` / `ResourceFilter` | `RenderGraphPrivate.cpp:85`, `:135`, `:152` |

## If a test is added

The shape that matches existing conventions (`RHITests.spec.cpp:18-219`): a spec under `Rendering.RHI` with `NonNullRHI`, a helper class per area, and `RunOnRenderThreadSynchronous` around the graph. A graph test must assert on graph decisions, not on RHI calls — for example, that a declared-but-unconsumed pass is culled (`RenderGraphBuilder.cpp:1387-1423`), that a UAV write prevents a merge (`RenderGraphResources.cpp:96`, `:115`), or that an extracted resource is filled after execution (`RenderGraphBuilder.cpp:2202-2212`). Validation does not check those outcomes — it asserts API inputs and lifecycle (`RenderGraphValidation.h:47-108`) — so a regression test adds value with `RDG_ENABLE_DEBUG` enabled as well; it only overlaps validation when the assertion is the same input or lifecycle check validation already performs.
