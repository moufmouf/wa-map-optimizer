# Follow-up: Phaser 4 `TilemapGPULayer` support in the WorkAdventure front

> Context for the implementer: this document was written alongside a refactor of
> [`wa-map-optimizer`](https://github.com/workadventure/wa-map-optimizer). It describes what the new
> optimizer guarantees about optimized maps, and the runtime changes the WorkAdventure front needs
> to take advantage of Phaser 4's `TilemapGPULayer`.

## What the new wa-map-optimizer guarantees

Phaser 4's `TilemapGPULayer` can only render a tile layer whose tiles all come from a **single
tileset**. The optimizer now guarantees this by construction:

1. **Every tile layer of an optimized map references tiles from exactly one tileset.** The
   optimizer clusters layers and renders one tileset per cluster (a tileset is typically shared by
   several layers). Where necessary, a tile is duplicated into two tilesets so that each layer
   stays self-contained.
2. **Tileset textures have rectangular power-of-2 dimensions** (e.g. 2048×1024), capped at a
   configurable maximum (default 4096×4096).
3. **Animation frames always live in the same tileset as their base tile**, so animated tiles work
   on GPU layers too.
4. **Exception — oversized layers:** if a *single layer* uses more tiles than fit in the maximum
   texture size, the optimizer emits a warning and renders that one layer with several tilesets
   rather than splitting the layer. Such a layer is NOT GPU-eligible and must fall back to the
   classic CPU tilemap layer. All other layers of the map keep the single-tileset guarantee.

There is no explicit marker on the map: **GPU eligibility must be detected per layer** by scanning
the layer's tile GIDs (after stripping the flip bits, i.e. `gid & 0x1FFFFFFF`) and checking they
all fall within a single tileset's `[firstgid, firstgid + tilecount)` range. This detection also
makes the front work correctly with non-optimized or hand-made maps: any layer that happens to use
a single tileset is GPU-eligible, whatever produced the map.

## Required runtime change: dynamically modified layers lose GPU eligibility

The scripting API (`WA.room.setTiles`, and anything else that mutates layer tile data at runtime)
can place **any tile of any tileset** on **any layer** — typically "named tiles" (tiles carrying a
`name` property), which the optimizer keeps in the output even when unused, but possibly in a
tileset different from the one the target layer uses.

Decision (from David): **a dynamically modified layer loses its GPU-layer eligibility** instead of
constraining the optimizer to duplicate every named tile into every tileset.

Suggested implementation:

- Render every GPU-eligible layer with `TilemapGPULayer` by default.
- On the first scripting mutation touching a layer (e.g. first `setTiles` call targeting it),
  **demote that layer** to a classic CPU tilemap layer, then apply the mutation. The demotion is
  per-layer; other layers keep their GPU path.
- Demoted layers must support tiles from any tileset of the map (this is what the classic layer
  already does today).
- An alternative/optimization if lazy demotion is awkward mid-frame: eagerly demote layers that the
  map declares script-mutable, if such a declaration exists or is introduced. Lazy demotion is the
  preferred default since most layers are never touched by scripts.

## Notes for the implementation

- **Named tiles** are guaranteed to exist in at least one tileset of the optimized map (the
  optimizer packs named tiles that no layer uses into one of the output tilesets). Looking up a
  tile by its `name` property must search all tilesets, not just the one used by the target layer.
- **Flip bits**: optimized maps still use Tiled's flip bits (bits 29–31 of the GID). GPU-eligibility
  detection and any GID → tileset resolution must mask them off first.
- **Oversized-layer fallback**: don't assume every layer of an optimized map is GPU-eligible (see
  exception above); the per-layer detection handles this naturally.
- **String-encoded layer data** (base64/compressed) is not optimized by wa-map-optimizer; such maps
  should not be assumed to have the single-tileset guarantee either. Detection by scanning decoded
  GIDs still works.

## Suggested test scenarios

1. Optimized map, static layers → all rendered as `TilemapGPULayer`, visual result identical to the
   CPU renderer (including flipped/rotated tiles and animated tiles).
2. `WA.room.setTiles` placing a named tile (from another tileset) on a GPU layer → layer demotes,
   tile displays correctly, other layers stay GPU-rendered.
3. Map with an oversized layer (optimizer warned, layer spans several tilesets) → that layer
   renders via the CPU path, the rest via GPU.
4. Non-optimized legacy map → per-layer detection: GPU where possible, CPU elsewhere; no crash.
