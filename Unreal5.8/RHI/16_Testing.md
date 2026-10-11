# Testing — UE 5.8 RHI

## Scope

The RHI test harness, what it covers, and how to run it against a specific backend.

## Entry points

Two independent paths into the same test bodies:

1. **Automation spec** `FAutomationRHITest`, registered as `Rendering.RHI` with `EAutomationTestFlags::NonNullRHI` (`Engine/Plugins/Tests/RHITests/Source/RHITests/Private/RHITests.spec.cpp:18`). `NonNullRHI` means these tests require a real RHI — they do not run under NullDrv.
2. **Command line** `-rhiunittest` runs all, `-rhiunittest=Filter` runs matching (`RHITestsModule.cpp:167-173`). The module is `RHITests` (`IMPLEMENT_MODULE` `L194`), and `LaunchEngineLoop.cpp:4380-4383` calls `IRHITestModule::RunAllTests()` in any non-Shipping build if the module is loaded.

The interface is `IRHITestModule` (`RHI/Public/Tests/RHIUnitTests.h:6-11`). Shader sources are mapped to `/Plugin/RHITests` at module startup (`RHITestsModule.cpp:161-162`), which is why every `IMPLEMENT_GLOBAL_SHADER` in the suite uses that prefix.

Log category: `LogRHIUnitTestCommandlet` (`RHITestsCommon.cpp:5`).

## Coverage

| Suite | File | What it proves |
|---|---|---|
| Bindless / descriptor ranges | `RHIBindlessTests.cpp` | descriptor ranges of textures (`L219`), UAVs (`L368`), mixed SRV+UAV in one range (`L508`), texture + buffer SRV mix (`L596`), bindless uniform buffer handle and contents (`L733`) |
| Draw and instancing | `RHIDrawTests.cpp` | instanced draw path (`TestDrawInstanced.usf`, `L23`, `L39`) |
| Clear / fill | `RHIClearTextureTests.cpp` | clear vs compute-fill equivalence across mips/slices; `L457`/`L475`/`L545` are the `TestFillTexture.usf` VS/PS/CS registrations |
| Graphics UAVs | `RHIGraphicsUAVTests.cpp` | UAV writes from vertex and pixel shaders (`L38`, `L54`); gated on `GRHIGlobals.SupportsPixelShaderUAVs` (`L112`) and `GRHIGlobals.SupportsVertexShaderUAVs` (`L164`) |
| Readback | `RHIReadbackTests.cpp` | buffer and texture readback correctness (`L39`, `L64`, `L66`) |
| Root constants | `RHIRootConstantsTests.cpp` | root constants in VS/PS/CS (`L50`, `L62`, `L74`) |
| Shader bundles | `RHIShaderBundleTests.cpp` | CS/MS/PS/VSPS bundles (`L81`, `L113`, `L145`, `L174`, `L208`) |
| Buffers | `Public/RHIBufferTests.h` | UAV clear on vertex/structured buffers (`L236`, `L310`, `L346`, `L387`), buffer creation (`L492`), parallel creation gated on `GRHISupportsMultithreadedResources` (`L504`); `RHIBufferTests.cpp:5` and `L68` provide `VerifyBufferContents`, reused by the other suites |
| Textures | `Public/RHITextureTests.h` | UAV clear across mips/slices (`L167`, `L239`), per-format render-target and depth capability checks (`L304`, `L324`, `L344`), texture updates (`L858`), copy (`L876`), multiple lock (`L1501`); `RHITextureTests.cpp:5` provides `VerifyTextureContents` |
| Allocators | `RHIAllocatorTests.cpp` | `Test_LockBuffer16ByteAlignment` (`L5`) — locked address must be 16-byte aligned when alignment `0` is requested (`L10`, check `L37`) |
| Reserved resources | `RHIReservedResourceTests.cpp` | reserved/sparse creation, gated on `GRHIGlobals.ReservedResources.Supported` (`L80`) and `SupportsVolumeTextures` (`L11`) |

Most test bodies are `static bool` members in the plugin's `Public/` headers; the `Private/*.cpp` files hold the readback helpers and the non-templated suites. That is why the coverage table mixes header and source paths.

Shaders live in `Engine/Plugins/Tests/RHITests/Shaders/Private/`: `TestBindlessUniformBuffer.usf`, `TestDescriptorRange.usf`, `TestDrawInstanced.usf`, `TestFillBuffer.usf`, `TestFillTexture.usf`, `TestGraphicsUAV.usf`, `TestRootConstants.usf`, `TestShaderBundle.usf`.

## What the tests exercise that unit tests usually miss

- **Access-state contracts.** A test that dispatches, then reads back, only passes if the resource was transitioned to the right access — the validating RHI asserts it (`RHIValidationContext.h:934-968`).
- **Capability gates.** Tests that need vertex-shader UAVs or descriptor ranges check the platform capability before running, so a failure distinguishes "backend lacks the feature" from "backend implements it wrong".
- **Alignment.** `Test_LockBuffer16ByteAlignment` (`RHIAllocatorTests.cpp:5`) asserts a locked buffer address is 16-byte aligned when the requested alignment is `0` (`L10`, check `L37`); the command-list allocator itself uses `FAlignedAllocator` (`RHIAllocators.h:20`) with 64 KB blocks (`L14`).
- **Indirect args.** Draw tests cover the indirect path that `MeshDrawCommands.cpp:930-936` falls back from when `PrimitiveIdStreamIndex < 0`.

## Running

```
UnrealEditor -rhiunittest                       # all
UnrealEditor -rhiunittest=Bindless              # filter
UnrealEditor -rhiunittest=ShaderBundle          # filter
```

Against a specific backend: `-rhi=D3D12`, `-rhi=Vulkan`, `-rhi=Metal`. NullDrv cannot run them because the spec requires a non-null RHI (`RHITests.spec.cpp:18`).

For CI, the automation framework runs `Rendering.RHI` as part of the engine filter; the command-line path is for targeted single-backend runs.

## Practical notes

- Tests run on the render thread against `FRHICommandListImmediate`; they do not go through RDG, so RDG-specific behavior (merge eligibility, `ERDGPassFlags::NeverMerge`) is not covered here.
- A test that passes on D3D12 and fails on Vulkan usually indicates a capability assumption, not a bug — check `DataDrivenShaderPlatformInfo.h` bits first.
- Shipping builds skip the module lookup entirely (`LaunchEngineLoop.cpp:4379`), so RHI unit tests are not part of a Shipping run.
