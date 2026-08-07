# Latios Containers (extract)

Entities-independent containers and allocators lifted out of `Latios.Core`, so they
can be used on **Entities 6.x**, where the full framework cannot build.

This is **not** an official Latios Framework package. See
[`../ENTITIES_6_PORT_AUDIT.md`](../ENTITIES_6_PORT_AUDIT.md) for why the rest of
Core doesn't come along.

## Why this exists

`Latios.Core` reaches into Unity's `Unity.Entities` assembly via `.asmref` source
injection to touch private members. That surface is tied to Entity Store V1, which
Unity has removed in the 6.x line. Since Entities is now a **core package** versioned
with the editor, you cannot pin an older Entities to get the framework back — the
only lever is the editor version itself.

The types in here never touch `Unity.Entities` at all. They depend only on
Collections, Burst, and Mathematics, so they carry ordinary API-drift risk
(compile errors you can fix) rather than the memory-corruption risk of ported
pointer code.

## Contents

| Type | Namespace | Use |
|---|---|---|
| `UnsafeParallelBlockList` / `UnsafeIndexedBlockList` | `Latios.Unsafe` | Per-thread lock-free append, then enumerate. Untyped. |
| `UnsafeParallelBlockList<T>` / `UnsafeIndexedBlockList<T>` | `Latios.Unsafe` | Typed variants. |
| `ThreadStackAllocator` | `Latios.Unsafe` | Per-thread scratch stack allocator for job-local temporaries. |
| `GapAllocator` | `Latios.Unsafe` | Free-list / gap suballocation within a larger buffer. |
| `AtomicDoubleBuffer<T>` | `Latios.Unsafe` | Atomic double-buffered value. |
| `RadixSort` | `Latios` | LSD radix sort over `int` and `int3` keys. |

### Relevance to a voxel project

`UnsafeParallelBlockList` is the natural fit for greedy/binary meshing: each worker
thread appends vertices or quads to its own block chain with no contention, and you
walk the whole set afterwards to build the mesh. `ThreadStackAllocator` covers the
per-chunk scratch buffers that pass through those jobs, and `GapAllocator` suits
suballocating chunk meshes inside one large GPU buffer rather than one buffer per
chunk.

## Installation

Copy the `LatiosContainers` folder into your project's `Packages/` directory, or
drop `Runtime/` anywhere under `Assets/`.

The asmdef references `Unity.Collections`, `Unity.Burst`, and `Unity.Mathematics`.
`JobHandle` and `JobsUtility` come from the engine core module, so no `Unity.Jobs`
reference is needed. Add one if your Unity version says otherwise.

## Modifications from upstream

Source is otherwise verbatim from Latios Framework `0.16.0-alpha.4` (commit
`07af21f` on this fork). One change:

- `RadixSort`, `IRadixSortableInt`, and `IRadixSortableInt3` were `internal` in
  `Core/Internal/RadixSort.cs`; promoted to `public` so they are usable from
  outside this assembly.

## Caveat on RadixSort

`RadixSort` sorts on 32-bit keys. That is fine for ordinary integer data — voxel
coordinates, chunk indices, material IDs.

Do **not** use it to sort `Entity` values. Unity has widened `EntityId` from 4 to 8
bytes and explicitly warns that sorting by `EntityId` no longer yields creation
order. Upstream has already marked its own entity-sorting path obsolete for this
reason.

## Untested

None of this has been compiled against Entities 6.5.0 / Unity 6000.7 — it was
extracted by static analysis, without an editor. Expect Collections API drift to
need small fixes on first import.

## License

Unity Companion License, same as the framework. See `LICENSE.md`.
Upstream: https://github.com/Dreaming381/Latios-Framework
