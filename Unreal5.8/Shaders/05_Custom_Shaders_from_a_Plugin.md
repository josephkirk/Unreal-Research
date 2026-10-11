# 05 — Custom Shaders from a Plugin

The complete recipe, with the exact macro or virtual at each step. No engine source edit is required at any step.

## 1. Module load phase

Shader types must be registered before the type list is frozen. `FShaderTypeRegistration`'s constructor asserts this (`RenderCore/Private/Shader.cpp:315`):

> `checkf(!AreShaderTypesInitialized(), ... "use ELoadingPhase::PostConfigInit on your module")`

```cpp
// Plugin module
virtual void StartupModule() override
{
    FString PluginShaderDir = FPaths::Combine(IPluginManager::Get().FindPlugin(TEXT("MyPlugin"))->GetBaseDir(), TEXT("Shaders"));
    AddShaderSourceDirectoryMapping(TEXT("/Plugin/MyPlugin"), PluginShaderDir);   // ShaderCore.cpp:4218
    ...
}
```

`AddShaderSourceDirectoryMapping` (`Runtime/RenderCore/Private/ShaderCore.cpp:4218-4231`) requires game thread, rejects `.` in the virtual directory, requires the virtual dir to start with `/` and not end with `/`, and returns early when `FPlatformProperties::RequiresCookedData() || !AllowShaderCompiling()` — so the mapping is editor/non-cooked only. Shared includes go through `AddShaderSourceSharedVirtualDirectory` (Niagara uses it at `NiagaraShaderModule.cpp:18`).

## 2. Register the type

```cpp
// Global shader, compute entry
class FMySimCS : public FGlobalShader
{
    DECLARE_GLOBAL_SHADER(FMySimCS)                                   // GlobalShader.h:431
    SHADER_USE_PARAMETER_STRUCT(FMySimCS, FGlobalShader)              // ShaderParameterStruct.h:62

    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)   // Shader.h:848
    {
        return RHISupportsComputeShaders(Parameters.Platform);
    }

    static void ModifyCompilationEnvironment(const FGlobalShaderPermutationParameters& Parameters,
                                             FShaderCompilerEnvironment& OutEnvironment)          // Shader.h:842
    {
        OutEnvironment.SetDefine(TEXT("MY_GROUP_SIZE"), 8);
    }

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )                      // ShaderParameterMacros.h:1411
        SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture1D<float4>, OutData)   // ShaderParameterMacros.h:1733
        SHADER_PARAMETER(int32, NumElements)
    END_SHADER_PARAMETER_STRUCT()                                       // ShaderParameterMacros.h:1414
};

IMPLEMENT_GLOBAL_SHADER(FMySimCS, TEXT("/Plugin/MyPlugin/Private/MySim.usf"), "MainCS", SF_Compute);  // GlobalShader.h:433
```

For a material-family shader use `IMPLEMENT_MATERIAL_SHADER_TYPE` (`Runtime/Engine/Public/MaterialShaderType.h:13-20`) and register the type into `FMaterialShaderTypes::AddShaderType` (`192-207`, `AddShaderType` at `200`).

## 3. Execute

Through RDG (`RenderGraphUtils.h:333` `namespace FComputeShaderUtils`, `GetGroupCount` at `339`, `kGolden2DGroupSize = 8` at `336` — the comment states 8×8 fills a GCN wave or two Nvidia warps), or directly `DispatchComputeShader` (`Runtime/RHI/Public/RHICommandList.h:2699`) / `DispatchIndirectComputeShader` (`2709`).

## 4. Plugin extension points that do not need a shader type

| Extension | Mechanism | File |
|---|---|---|
| Mesh deformer | `IMeshDeformerProvider : IModularFeature`, `ModularFeatureName` `"MeshDeformer"`, `IsSupported(EShaderPlatform)` | `RenderCore/Public/Animation/MeshDeformerProvider.h:17,22,27` |
| Render capture | `IRenderCaptureProvider : IModularFeature`, `"RenderCaptureProvider"` | `RenderCore/Public/IRenderCaptureProvider.h:10,18-22` |
| Shader audit session view | `IShaderAuditExtension : IModularFeature`, `FeatureName = "ShaderAuditExtension"`, `GetExtensionId`, `GetDisplayName`, `GetContributions`, `OnSessionLoaded`, `OnSessionClosed` | `Engine/Plugins/Developer/ShaderToolkit/Source/ShaderAuditCore/Public/IShaderAuditExtension.h:42` |
| Scene view extension | `ISceneViewExtension` + `FSceneViewExtensions::NewExtension<T>()` | `Runtime/Engine/Public/SceneViewExtension.h:117,328-334` |
| Post process pass subscription | `ISceneViewExtension::SubscribeToPostProcessingPass` | `SceneViewExtension.h:222` |

`MeshDeformerProvider.h:12-16` states the provider must be loaded before shader compilation starts (`PostConfigInit`) — same constraint as shader types. `IRenderCaptureProvider::IsAvailable()` (`29-34`) locks the modular feature list, so capture providers must also register early.

## Worked example: Niagara

Niagara is a plugin that owns a shader family, a virtual shader directory, a compile-queue delegate and a compute dispatch path:

- `AddShaderSourceDirectoryMapping(TEXT("/Plugin/FX/Niagara"), PluginShaderDir)` + `AddShaderSourceSharedVirtualDirectory(TEXT("/Plugin/FX/Niagara/Shared/"))` — `NiagaraShaderModule.cpp:13-21`
- `SetOnProcessShaderCompilationQueue` delegate, with a `checkf` enforcing a single binding — `NiagaraShaderModule.cpp:23-28`
- `IMPLEMENT_SHADER_TYPE(, FNiagaraShader, TEXT("/Plugin/FX/Niagara/Private/NiagaraEmitterInstanceShader.usf"), TEXT("SimulateMain"), SF_Compute)` — `NiagaraShader.cpp:31`
- `FNiagaraShaderType` constructor is one of the highest-degree symbols in the shader corpus (`NiagaraShaderType.h:82`, degree 15); `FNiagaraShaderMap::CreateCompileJobs()` (`NiagaraShader.cpp:734`) and `BeginCompileShaderFromSource()` (`409`) are the plugin-side compile entry points.
- Dispatch: `FNiagaraGpuComputeDispatch::DispatchStage(FRDGBuilder&, ...)` (`Engine/Plugins/FX/Niagara/Source/Niagara/Private/NiagaraGpuComputeDispatch.cpp:1475`)

That is the whole pattern: directory mapping + type registration + permutation gate + RDG dispatch. Every other plugin shader (ComputeFramework, OCIO, NNE, HairStrands, PCG, GLTF, nDisplay, media decoders) is the same shape.

## Failure modes

- Registering at a load phase later than `PostConfigInit` → assert at `Shader.cpp:315`, or a type that never appears in a shader map.
- Virtual directory not starting with `/` or containing `.` → rejected at `ShaderCore.cpp:4218-4231`.
- `ShouldCompilePermutation` returning false for a target platform → permutation missing in the cooked archive; the material binds to the fallback.
- Parameter struct field not bound → validation error only when the struct is fully bound; `SHADER_USE_PARAMETER_STRUCT_WITH_LEGACY_BASE` (`ShaderParameterStruct.h:69`) suppresses that validation for legacy bases.
