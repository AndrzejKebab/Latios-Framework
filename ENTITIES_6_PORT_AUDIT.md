# Porting Latios Framework 0.16 to Entities 6.x

Audit of what blocks `Latios.Core` + `Latios.Transforms` from building against
**Entities 6.5.0 / Unity 6000.7**, and what each blocker needs.

Target project context: Entities 6.5.0, Entities Graphics 6.5.0, Unity Physics
6.5.0, Collections 6.5.0, Burst 2.0.0, Netcode 6.7.0, editor 6000.7.0a3, URP 17.7,
CoreCLR scripting backend.

Framework state: 0.16.0-alpha.4, which targets *Entities 1.4.4 with
`ENTITY_STORE_V1`*, minimum editor 6000.3.8f1.

---

## 1. Scope of the coupling

| Layer | Files | Notes |
|---|---|---|
| `EntitiesExposed/` | 8 files, 695 lines | The entire deep-internals surface |
| `Latios.Core` consumers | 25 of 96 files | Use `Unity.Entities.Exposed` |
| `Latios.Transforms` consumers | 17 of 87 files | Use `Unity.Entities.Exposed` |
| Rest of repo | 698 of 740 `.cs` | Public API only |

`EntitiesExposed/EntitiesExposed.asmref` targets `Unity.Entities`, and
`Hybrid/EntitiesHybridExposed.asmref` targets `Unity.Entities.Hybrid`. This
compiles Latios source *into* Unity's assemblies to reach private members. It is
the reason the framework is version-locked, and it is the whole of the port.

The good news: the coupling is narrow and concentrated. Port those 8 files and the
other 690 follow.

---

## 2. Hard blockers

### 2.1 Entity Store V1 removed

Unity's 6.6 notes state: *"The legacy Entity Store V1 implementation has been
removed."* The framework's README declares it targets `ENTITY_STORE_V1`.

Affected — all in `EntitiesExposed/ArchetypeChunkExposed.cs`:

- `chunk.m_EntityComponentStore->GetComponentDataWithTypeRW/RO(...)`
- `chunk.m_EntityComponentStore->AssertEntityHasComponent(...)`
- `chunk.m_Chunk.MetaChunkEntity`
- `math.asuint(chunk.m_Chunk)` — `GetChunkIndexAsUint`
- `archetype.Archetype->` — `BloomFilterMask`, `HasChunkHeader`,
  `HasSystemInstanceComponents`, `CleanupResidueArchetype`, `NumChunkComponents`,
  `NumBufferComponents`, `Types[i].TypeIndex`, `Chunks[i]`, `EntityComponentStore`

Every one of these needs re-derivation against the V2 store layout. This is
pointer-level work; it cannot be inferred, only written against the real 6.5.0
source.

### 2.2 EntityId widened to 8 bytes

Unity: *"we've completed the EntityId 8-byte migration (up from 4 bytes)"*, and
explicitly warns against *"sorting by EntityId to sort by creation order"* and
*"casting EntityId to and from an int."*

Affected:

- `Core/Internal/RadixSort.cs:116` — `RankSortInt3<T>` sorts on three 32-bit keys
- `Core/Containers/CommandBuffers/EntityOperationCommandBuffer.cs:319` — radix-sorts
  entities via `RankSortInt3`
- `Core/Internal/AddComponentsCommandBufferUntyped.cs:454` — same, on
  `WrappedEntityLocationInChunk`
- `GetChunkIndexAsUint` (above) assumes a 32-bit chunk index

Upstream already anticipated this. `EntityOperationCommandBuffer.cs:191` carries:

> *"This method depends on ENTITY_STORE_V1 for deterministic Entity ordering and
> will be removed when Entity Store V2 becomes the only supported mode in Unity
> versions > 6.3 LTS."*

So `GetEntitiesSortedByEntity` is slated for deletion, not porting. The other two
call sites need a 64-bit-safe sort.

### 2.3 Managed components deprecated

Unity is deprecating class `IComponentData`, managed `ISharedComponentData`, and
the `AddComponentObject` / `GetComponentObject` / `SetComponentObject` family in
favour of `UnityObjectRef<T>`.

Affected:

| Site | API |
|---|---|
| `EntitiesExposed/EntityManagerExposed.cs` | `MoveComponentObjectDuringStructuralChange`, `SetSharedComponentDataBoxedDefaultMustBeNull`, `GetSharedComponentDataBoxed` |
| `Core/Framework/BlackboardEntity.cs:187,207` | `SetSharedComponentManaged`, `GetSharedComponentManaged` |
| `Core/Systems/_Essentials/MergeBlackboardsSystem.cs:107` | `MoveManagedComponent` |
| `Core/Internal/ManagedStructStorage.cs` | the `IManagedComponent` storage backend |
| `Core/Editor/UnityScenesMods/LatiosEditorResolveSceneReferenceSystem.cs:202,207` | `GetComponentObject<SubScene>`, `AddComponentObject` |
| `Core/Systems/Scenes/SceneManagerSystem.cs:113` | `GetComponentObject<SubScene>` |

These still *work* in 6.5 (deprecated, not removed), so they are warnings rather
than errors — but `IManagedComponent` is a public framework feature whose backing
storage is on a removal path. Worth deciding early whether to keep it.

### 2.4 Entities is a core package — the version cannot be pinned

Entities now ships *with* the editor and is versioned *as* the editor. There is no
way to upgrade or downgrade it independently. The only lever on the Entities
version is the Unity version itself.

Two consequences, and the second is the important one:

**No escape hatch.** You cannot pin Entities 1.4.8 under Unity 6000.7 to get the
framework building. Running Latios 0.16 as shipped would mean moving the whole
project back to Unity 6000.3 — giving up Netcode 6.7, Collections 6.5, Burst 2.0,
and the dedicated-server 3.0 stack.

**A port is a recurring tax, not a one-time cost.** `EntitiesExposed` is pinned to
private fields and internal layout. Those carry no compatibility guarantee and can
shift on any editor update — and the version cannot be held back while the port
catches up. Every Unity upgrade becomes a re-verification of pointer-level code,
where the failure mode is memory corruption rather than a build error. This project
is on **6000.7.0a3, an alpha**, so that internal layout is actively churning.

Also unresolved: `.asmref` injection requires the target assembly to be compiled
*from source*. Worth confirming that `com.unity.entities@6.5.0` ships `.cs` sources
and an `.asmdef` rather than a prebuilt `Unity.Entities.dll` — if it's the latter,
the approach cannot work at all.

---

## 3. Things that got *easier*

Entities 1.4 added public API that replaces some of the hacks outright. These
`EntitiesExposed` members can likely be **deleted** rather than ported:

| Exposed hack | Public replacement |
|---|---|
| `ArchetypeChunkExposed.GetBufferAccessor<T>(ref DynamicComponentTypeHandle)` — reinterprets `UnsafeUntypedBufferAccessor` to steal its pointer; source comment reads *"Todo: Super dangerous"* | `ArchetypeChunk.GetUntypedBufferAccessorReinterpret<T>` |
| Implicit-access-mode buffer workarounds | `ArchetypeChunk.GetBufferAccessorRO` / `GetBufferAccessorRW` |
| `WorldExposed` system-type-index plumbing | `WorldUnmanaged.GetSystemTypeIndex(SystemHandle)` |
| Optional-component ref patterns | `ComponentLookup.TryGetRefRO` / `TryGetRefRW` |

Also note Unity's `IAspect` is now obsolete. This is *not* a new problem for
Latios — the framework already ships its own `Core/Framework/IAspect.cs` and the
0.15.x line contains the migration ("unfinished IAspect refactoring", commit
`14565bf`). Latios is moving off Unity's aspects under its own steam.

---

## 4. Recommendation

**Don't port. Take the extract, and wait for upstream.**

Given §2.4, a private port of `EntitiesExposed` would need re-verifying against
private Entities internals on every editor update, with no ability to hold the
version back, on an alpha editor. That is a standing maintenance cost on the most
dangerous code in the project, paid indefinitely, to get features that upstream
will eventually ship for free — Entity Store V2 support is on their roadmap, and
they have already begun marking the V1-dependent paths obsolete.

[`LatiosContainers/`](LatiosContainers/) takes the part that has none of this
exposure: the containers and allocators that never touch `Unity.Entities`. For a
voxel project that is most of the practical value anyway (see §5).

### If you port anyway

**Do `Latios.Core`. Defer `Latios.Transforms`.**

QVVS Transforms replaces the transform system wholesale. Against Netcode 6.7 with
server-authoritative ghost serialization of transforms, that is the highest-risk
combination in this plan, and Latios's own networking module is still unwritten.
The framework ships `LATIOS_TRANSFORMS_UNITY` precisely so other modules can run on
Unity Transforms instead — use it.

For a voxel project that already has its own animation framework, Core is where
essentially all the value is anyway.

---

## 5. Usable today, without any porting

44 of Core's 96 files reference no `EntitiesExposed` type. Of those, these 11 touch
**no Entities API at all** — pure Collections/Burst/Mathematics, so they carry only
ordinary API-drift risk (compile errors, not memory corruption):

```
Core/Containers/Allocators/GapAllocator.cs
Core/Containers/Allocators/ThreadStackAllocator.cs
Core/Containers/Collections/AtomicDoubleBuffer.cs
Core/Containers/Collections/UnsafeParallelBlockList.cs
Core/Containers/Collections/UnsafeParallelBlockListTyped.cs
Core/Internal/RadixSort.cs          (see 2.2 — 32-bit key assumption)
Core/Internal/ProjectFlags.cs
Core/Authoring/SubsceneLoadOptions.cs
Core/Editor/PropertyInspectorUtilities.cs
Core/Utilities/StringBuilderExtensions.cs
Core/Utilities/UnityEngineObjectExtensions.cs
```

`UnsafeParallelBlockList` (parallel per-thread append), `ThreadStackAllocator`, and
`GapAllocator` (free-list suballocation) are a strong fit for a chunked voxel
meshing pipeline — worth lifting into a standalone assembly regardless of whether
the full port ever happens.

---

## 6. Order of work

1. **Verify §2.4.** Source or DLL. Everything depends on this.
2. Get `com.unity.entities@6.5.0` sources readable.
3. Delete the `EntitiesExposed` members superseded by §3's public API.
4. Port `ArchetypeChunkExposed` + `EntityManagerExposed` against the V2 store.
5. Fix the 64-bit entity sorts (§2.2); drop `GetEntitiesSortedByEntity`.
6. Decide the fate of `IManagedComponent` (§2.3).
7. Build `Latios.Core` with `LATIOS_TRANSFORMS_UNITY`; iterate on errors.
