# UnrealResearch — UE 5.8 Subsystem Documentation

Research documentation sets for Unreal Engine 5.8 subsystems, grounded in the actual UE 5.8 source (`UE_5.8`). Each bundle covers **architecture** (types, responsibilities, key APIs, data flow) and **usage** (patterns, pitfalls, cost model, what a tech artist must know). Every claim is cited as `file:line`; claims not observed in source are marked `[INFERENCE]`. Graph facts come from `graphify-out/graph.json` (scan root `UE_5.8`).

## Bundles

| Bundle | Scope |
|---|---|
| [Unreal5.8/UnrealRenderPipeline/](Unreal5.8/UnrealRenderPipeline/) | Full render pipeline: frame flow, scene views, culling, mesh passes, GBuffer, shadows, Lumen, MegaLights, Nanite, Substrate, screen-space techniques, post-process chain, translucency/volumetrics/water, scaling/VRS |
| [Unreal5.8/RHI/](Unreal5.8/RHI/) | Render Hardware Interface: backends, feature levels, resources, command lists, access states/transitions, pipeline states, draw calls, multi-GPU/threading, ray tracing |
| [Unreal5.8/RDG/](Unreal5.8/RDG/) | Render Dependency Graph: graph phases, resources/allocation, declared access, pass model, execution, blackboard, parameters, validation, profiling, Insights plugin |
| [Unreal5.8/Shaders/](Unreal5.8/Shaders/) | Shader system: type/registration model, compilation and caching, material shader generation, plugin shaders, compute shaders, debugging, profiling |
| [Unreal5.8/UI/](Unreal5.8/UI/) | User interface: Slate core architecture, widget catalog/theming, UMG runtime, MVVM, input/focus/events, editor UI extension, Slate rendering and fonts |
| [Unreal5.8/GAS/](Unreal5.8/GAS/) | Gameplay Ability System: abilities, gameplay effects, tasks, tags, attribute sets, GameplayCues, replication/prediction, GameFeatures |
| [Unreal5.8/MASS/](Unreal5.8/MASS/) | MASS ECS: entities/fragments/traits, processors, queries, observers, commands/defers, simulation phases, representation/LOD, movement, crowd/ZoneGraph, SmartObjects |
| [Unreal5.8/NDC/](Unreal5.8/NDC/) | Niagara Data Channel: asset variables, channel types, storage/publish pipeline, access context, query API, GPU execution, Mass interop |
| [Unreal5.8/MCP/](Unreal5.8/MCP/) | Model Context Protocol: JSON-RPC/transport, tool model, toolset registry, Python bridge, toolset catalog, AI agent control surface, limits/safety |

Each bundle has its own `README.md` with an index, layered architecture diagrams, module dependency graphs, source roots, and graph facts.

## Conventions

- Citations are `file:line` against UE 5.8 source paths unless another path is given.
- `[INFERENCE]` marks claims not directly observed in source.
- Cross-bundle behaviour is cross-linked, not restated (e.g. RDG chapters link to RHI for access states and command lists).
- Chapters are numbered `NN_Name.md`; each bundle ends with a file reference and a testing chapter.
