# 14 — Testing

## Coverage reality

There is **no automation test coverage for the shader system** in 5.8. Grep for `IMPLEMENT_*_AUTOMATION_TEST` returns:

| Scope | Matches |
|---|---|
| `Runtime/RenderCore` | none |
| `Runtime/Engine/Private/Materials` | none |
| `Runtime/Engine/Private/Tests` (shader-filtered) | none |
| `Developer/ShaderCompilerCommon` | none |
| `Plugins/Developer/ShaderToolkit` | none |

The only shader-adjacent test in the engine is `Developer/MaterialBaking/Private/Tests/ExportMaterialProxyTest.cpp:24-27`, `MaterialBaking.ExportMaterialProxy.ShaderDependencyConsistency` (`FExportMaterialProxyAnisotropyTest`). Its comment (`17-22`) describes the failure it catches: an overly broad `ShouldCache` lets a shader pass `ShouldCache` in `SubmitCompileJobs` even when `GetDependentShaderAndVFTypes` did not include it in the `ShaderMapId`, because the two use the same `ShouldCache` with different layouts (conservative vs compilation shader parameters). The test builds a minimal `FExportMaterialProxy` and asserts it compiles.

`Engine.Build.cs:51` lists `AutomationWorker` as a private include-path module, so automation runs — the shader modules simply have no tests. The `Plugin:Tests` module contributes 37 shader-named nodes in the graph, which is plugin test code, not engine shader test code.

## Verification surfaces instead of tests

| Surface | What it proves | Entry |
|---|---|---|
| RDG validation | Resource lifetime, transition, UAV overlap errors | `r.RDG.Validation` (`RenderGraphPrivate.cpp:25`) |
| Clobber + extended lifetimes | Wrong-pixel bugs become explicit | `r.RDG.ClobberResources` 60, `r.RDG.Debug.ExtendResourceLifetimes` 43 |
| Shader usage visualization | Which shaders are actually loaded | `r.ShaderCodeLibrary.VisualizeShaderUsage` (`ShaderCodeArchive.cpp:61-67`) |
| Stable key reader | Maps `.shk` keys to shader hashes for audit | `ShaderStableKeyDebugInfoReader.h:17-26` |
| ShaderAudit extension | Editor session view of shader cost/contributions | `IShaderAuditExtension.h:42` |
| Translation log | What a material graph produced | `r.Material.EnableTranslationLogFile` (`HLSLMaterialTranslator.cpp:90-94`) |
| Compile stats printout | Permutation compile time by family | `r.ShaderCompiler.StatsPrintoutInterval` (`ShaderStatsCollector.cpp:9-20`) |
| `ProfileGPU` | Per-pass GPU time | `GPUProfiler.cpp:2197-2205` |
| ShaderCompileWorker | Standalone compile of one permutation | `Engine/Binaries/Win64/ShaderCompileWorker.exe` (source not present in this install) |

## Recipe for a plugin shader

1. Confirm the permutation exists for the target platform: `ShouldCompilePermutation` result, then check the DDC record or the archive.
2. Confirm registration happened before `CommitAll` — the assert at `Shader.cpp:315` fires otherwise.
3. Run with `r.RDG.Validation` 1 and `r.RDG.ClobberResources` 1; a missing binding or a lifetime violation shows as an error, not a wrong pixel.
4. Add an RDG scope (`RDG_EVENT_SCOPE_STAT`, `RenderGraphEvent.h:480`) so the pass appears in `ProfileGPU`.
5. Verify on a cooked target, where the editor-only paths (`AddShaderSourceDirectoryMapping`, custom output gathering) are inactive.

`[INFERENCE]` — because there are no engine tests for the shader system, a plugin that changes permutation behaviour has no upstream regression net; verification is entirely runtime observation plus DDC/archive inspection.
