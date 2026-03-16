# Technical Analysis: Skelerealms vs. Bethesda's Creation Engine

## What Is Skelerealms?

Skelerealms is a GDScript addon for Godot 4.2+ that supplies the *infrastructure* layer for open-world RPGs in the style of The Elder Scrolls and Fallout. It does not ship with gameplay, UI, dialogue, or combat – those are intentionally left to individual projects. Its value proposition is solving the three hardest technical problems in open-world games:

1. **Cross-scene persistence** – objects remain where the player left them across scene loads.
2. **Cross-scene NPC navigation** – characters can path-find to a destination that spans multiple loaded and unloaded scenes.
3. **Living-world NPC AI** – characters have goals, schedules, factions, and perception that keep running even when not visible to the player.

---

## Architecture Overview

```
SKEntityManager          (singleton – owns all living entities)
└── SKEntity             (persistent data node – one per character/item/object)
    ├── NPCComponent     (AI brain: GOAP planner, schedule, perception)
    ├── InventoryComponent
    ├── EquipmentComponent
    ├── NavigatorComponent
    ├── PuppetSpawnerComponent  → spawns/despawns the visible CharacterBody3D
    └── … (custom components)
```

**Entities are always alive** in `SKEntityManager`, whether or not they are visible on screen. The visible `CharacterBody3D` (called a *puppet*) is a disposable view – it is spawned when the entity enters `actor_fade_distance` and freed when it exits. This is the core insight that makes cross-scene persistence and simulation possible.

---

## Comparison with Bethesda's Creation Engine

| Feature | Skelerealms | Creation Engine |
|---|---|---|
| **Persistent world object** | `SKEntity` (GDScript Node) | Actor/Object/Form (compiled C++) |
| **Scene/cell management** | `WorldLoader` (threaded ResourceLoader) | Cell streaming (background thread) |
| **Cross-scene NPC simulation** | Three-tier: FULL → GRANULAR → NONE | Radiant AI with off-cell stub simulation |
| **Off-scene pathfinding** | KD-tree + A* across nav networks | Simplified stub movement; limited cross-cell pathing |
| **NPC scheduling** | `Schedule` / `ScheduleEvent` with conditions | AI Packages (fixed-priority stack) |
| **NPC reasoning** | GOAP (Goal-Oriented Action Planning) | Behaviour-tree-like package system |
| **Faction system** | `Coven` with rank and opinion tables | Faction with disposition and rank |
| **NPC–NPC relationships** | `Relationship` with typed levels | Relationship rank (Actor value) |
| **Crime** | `CrimeMaster` per coven | Crime system per faction/hold |
| **Perception / stealth** | FSM (`PerceptionEyes`, `PerceptionEars`) | Threat/detection values on each actor |
| **Spell / effect system** | Script-based `Spell` + `StatusEffect` | Magic Effect FormIDs in ESM |
| **Inventory** | `InventoryComponent` + `ItemComponent` | Container/inventory on every Actor |
| **Persistence format** | JSON merge-save (`user://saves/*.dat`) | Binary `.ess` snapshot |
| **Scripting language** | GDScript (interpreted, JIT in Godot 4) | Papyrus (compiled bytecode VM) |
| **Engine coupling** | Godot 4.2+ addon – replaceable components | Monolithic engine – non-replaceable subsystems |
| **Dialogue / quests** | Not included (by design) | Full dialogue/quest engine built in |
| **LOD / chunking** | Not included | Cell streaming + object LOD |
| **Networking** | Planned (v0.6) | Not supported in base engine |

### Key Improvements Over Creation Engine

* **More capable AI planner** – GOAP supports dynamic re-planning as the world changes. Creation Engine's package system is a rigid priority-ordered stack that cannot reason about compound goals.
* **Composable design** – Any component can be swapped or omitted. Creation Engine's Actor class is a fixed hierarchy with hundreds of hard-coded fields.
* **Cleaner persistence** – Merge-based JSON saves preserve out-of-scope entities without requiring a monolithic snapshot of the entire game world.
* **True cross-scene off-world pathfinding** – The granular navigation system (KD-tree + A*) allows NPCs to actually traverse doors and navigate between unloaded scenes. Bethesda's NPCs are "teleported" to their schedule destination off-cell, which is why they sometimes appear to materialise out of nowhere.
* **No Papyrus overhead** – Papyrus is a slow, single-threaded scripting VM known for its CPU bottlenecks. GDScript in Godot 4 is several times faster for equivalent workloads, and performance-critical paths can be moved to GDExtension/C++ without changing the API.

### Where Creation Engine Still Leads

* **Dialogue and quests** – Creation Engine ships a full-featured dialogue tree, quest log, and cutscene system. Skelerealms intentionally provides none of these; they must be built from scratch.
* **Content pipeline and editor tooling** – The Creation Kit is a mature, battle-tested level editor. Skelerealms' editor tools are early and limited.
* **LOD / streaming** – Creation Engine's cell-streaming and object LOD are deeply integrated. Skelerealms has no built-in chunking or terrain system.
* **Community and ecosystem** – Decades of community modding and third-party tooling exist for Creation Engine. Skelerealms is in early alpha.

---

## The Persistence Model in Depth

### How it Works

```
save()
  ├── Collect dirty SKEntity blobs  (group: savegame_entity)
  ├── Collect GameInfo blobs         (group: savegame_gameinfo)
  ├── Collect other blobs            (group: savegame_other)
  ├── Merge with most-recent save    (preserves out-of-scope entities)
  └── Write timestamped .dat file    (JSON)

load_game(path)
  ├── Parse JSON
  ├── Reset entities absent from save to default state
  └── For each entity in save:
        SKEntityManager.get_entity(id)   ← cascading retrieval (cache → save → disk)
        entity.load_data(blob)
```

### Cascading Entity Retrieval

`SKEntityManager.get_entity()` follows a three-stage cascade:

1. **In-memory cache** – O(1) hash-table lookup.
2. **Save file** – If not cached, call `SaveSystem.entity_in_save(id)` to check the save file. (After the caching fix described below, this is a single parse per load session, not one per entity.)
3. **Disk** – If never seen before, load the entity's `.tscn` from `res://entities/`, instantiate it, and call `entity.generate()` for first-time setup.

### Comparison with Bethesda's `.ess` Format

| Property | Skelerealms (JSON) | Creation Engine (binary .ess) |
|---|---|---|
| **Format** | UTF-8 JSON | Custom binary (chunked records) |
| **Size** | Large (human-readable, no compression) | Compact (binary, LZ4 compressed) |
| **Partial saves** | Merge-based (only dirty state recorded) | Full snapshot of all modified records |
| **Portability** | Trivially inspectable and editable | Requires dedicated tools |
| **Performance** | O(n) parse on load (single pass with caching) | O(n) binary deserialisation (faster constant) |
| **Corruption risk** | Low (valid JSON is self-describing) | High (one bad byte can break the whole save) |

The JSON merge approach is an elegant solution for early development, but will require redesign for larger worlds. The planned v0.7 rework should consider:
- MessagePack or a Godot-native binary format for compact, fast serialization.
- A sparse/delta format that only writes records that changed since the last save.
- Compression (zstd or lz4) to reduce I/O time.

---

## Performance Assessment for a Smaller-Scale Bethesda-Like RPG

"Smaller-scale" is taken here to mean a game roughly comparable to Morrowind or a small Fallout New Vegas run: 1–3 outdoor zones + 10–20 interior cells, 50–150 persistent NPCs, and no more than ~500 persistent item/object entities.

### What Performs Well

| System | Assessment |
|---|---|
| Entity cache lookup | O(1) – very fast for any entity count |
| KD-tree nearest-node query | O(log n) per NPC per frame – efficient for ≤ 200 off-scene NPCs |
| Granular navigation A* | Negligible for small nav networks (< 500 nodes per world) |
| GOAP planning | Negligible if plan depth ≤ 6 actions; replanning is rare |
| Perception FSM | O(visible\_objects) per NPC per frame – cheap with a small view frustum |
| Puppet spawn/despawn | Threaded world loading avoids stalls |
| JSON save (small world) | < 1 ms for ≤ 200 entities – imperceptible |

### Bottlenecks to Watch

1. **Off-scene NPC `_process` loop** – Every off-scene NPC runs its position-advance logic in `NPCComponent._process()` on the *main thread* every frame. For 100 NPCs this is hundreds of dictionary reads and Vector3 operations per frame. At 150+ off-scene NPCs, frame-time impact becomes measurable. Mitigation: move granular simulation to a background thread or a `SubViewport`-less `Thread`, and update positions once per simulated second rather than every frame.

2. **A* open-list sort** – `NavMaster.calculate_path()` re-sorts the entire open list with `Array.sort_custom()` on every iteration. This is O(n log n) per A* step instead of O(log n) with a min-heap. For nav graphs with > 200 nodes and concurrent path requests from several NPCs, this will spike frame time. Mitigation: replace the open list with a `PriorityQueue` implemented over a sorted Array or a binary heap.

3. **`SKEntity._process()` per-frame distance check** – Every entity calls `position.distance_squared_to(...)` and checks `GameInfo.world` every frame. For 500 entities, that is 500 per-frame property reads. Mitigation: move the in-scene check to a timer (e.g. every 500 ms) or use Godot's area-exit signals on the player's `Area3D` to trigger entity culling.

4. **Save-file I/O growth** – The merge strategy accumulates all entity state into a single file. After many saves a large world's save file can become several megabytes of JSON. `load_game()` deserialises this once, which is fine; the concern is that `_get_most_recent_savegame()` calls `FileAccess.get_modified_time()` for *every* file in `user://saves/` every time an entity is retrieved. Mitigation: cache the most-recent path for the duration of a load session (analogous to the existing `_cached_save_data` fix).

5. **Opinion calculation on game loop** – `NPCComponent.determine_opinion_of()` calls `SKEntityManager.instance.get_entity(id)`, which can trigger a save-file parse or disk load in the middle of an AI update. Mitigation: cache opinion results per entity pair and invalidate only when a relevant crime or relationship event fires.

### Realistic Verdict

For a smaller-scale game with ≤ 150 NPCs and ≤ 500 persistent entities at the current alpha stage:

- **Persistence**: Works correctly after the bugs fixed in this release. JSON merge saves are practical at this scale. The caching fix reduces load-screen time from O(n × file\_parse) to O(1 × file\_parse + n × hash lookup).
- **Performance**: Acceptable at 60 fps on mid-range hardware, but fragile. The main risk is concurrent off-scene NPC simulation crossing 50–100 actors. Developers should keep `skelerealms/granular_navigation_sim_distance` tightly tuned and avoid worlds with more than 100 off-scene actors simultaneously.
- **Stability**: The framework is in alpha. The bugs fixed in this release (`entity.gd` save/load key errors, `navigation_master.gd` reduce crash, save-file re-parse loop) were all silent crashes that would surface immediately on first save/load or first KD-tree construction. Expect further edge-case issues as integration deepens.

---

## Summary of Bugs Fixed in This Release

| File | Bug | Impact |
|---|---|---|
| `scripts/entities/entity.gd` | `save()` wrote to `data["components"]` before the key existed → `KeyError` crash on any save with dirty components | Critical – saves with component data were broken |
| `scripts/entities/entity.gd` | `load_data()` used `data[d]` instead of `data["components"][d]`; called `JSON.parse_string()` on an already-parsed Vector3 string instead of `str_to_var()`; no guard for missing `"components"` key | Critical – entity load was unreliable |
| `scripts/granular_navigation/navigation_master.gd` | Three reduce-chain lambdas used `NavPoint` as an `Array` accumulator and returned `null`, so the chained `.reduce()` always crashed at KD-tree construction time | Critical – any world with a nav network crashed on startup |
| `scripts/system/save_system.gd` | `entity_in_save()` opened and fully parsed the save file once per entity during `load_game()`, making load time O(n × file\_size) | Performance – load times grew quadratically with save file size and entity count |
