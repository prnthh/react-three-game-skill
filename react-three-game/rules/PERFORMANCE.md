# Performance

Prefab JSON describes structure. Live Three.js objects and shared managers handle frame-frequency work.

Eligible `Mesh` components join the scene instancing service by default; set `instanced: false` to keep a mesh on its native render path. Repeat positions on a non-animated `Model` use the same service. Ordinary model nodes keep their native render path.

## Work placement

| Work | Preferred surface |
|---|---|
| Serializable edit | `prefab.update()`, `setMaterial()`, structural prefab methods |
| Animation and controls | `prefab.getObject()` or `useNodeObject()` |
| Many passive entities | One manager with one `useFrame()` callback |
| Shared asset | Asset runtime cache |
| Repeated mesh | Implicit `Mesh` instancing or repeat axes on one non-animated `Model` |
| World residency | Chunk manager with stable ids and bounded admission |

## Frame-loop shape

```tsx
const position = new Vector3();
const target = new Vector3();

useFrame((_, delta) => {
  if (scene.mode !== PrefabEditorMode.Play) return;

  for (let index = 0; index < entries.length; index += 1) {
    const entry = entries[index];
    const object = prefab.getObject(entry.id);
    if (!object) continue;
    object.getWorldPosition(position);
    position.lerp(target, Math.min(1, delta * entry.speed));
    object.position.copy(position);
  }
});
```

Stable arrays, indexed loops, reused vectors, typed-array mutation, pooled effects, and batched matrix uploads keep the hot path predictable.

## Repeated model syntax

```json
{
  "id": "tree-line",
  "components": {
    "model": {
      "type": "Model",
      "properties": {
        "filename": "/models/tree.glb",
        "repeat": true,
        "repeatAxes": [
          { "axis": "x", "count": 100, "offset": 4 },
          { "axis": "z", "count": 20, "offset": 4 }
        ]
      }
    }
  }
}
```

## Runtime interaction

| Feature | Efficient shape |
|---|---|
| Pointer events | `emitClickEvent` on interactive object components |
| Ray queries | Event-driven or throttled queries with a spatial broad phase |
| Instance transforms | One matrix update pass and one `needsUpdate` assignment |
| Streaming | Desired/resident id diff with hysteresis and bounded batches |
| Shadows | One fitted primary shadow caster for large outdoor scenes |
| UI statistics | Sampled state updates at a readable interval |

## Measurement table

| Measure | Context |
|---|---|
| Mount and streaming cost | Production build after warm-up |
| Authored mutation cost | Equal prefab and edit workload |
| CPU frame time | Equal camera path and entity count |
| GPU frame time | Equal materials, lights, shadows, DPR, backend |
| Draw calls and memory | Equal visible result |

Several samples and median values provide a useful optimization comparison.
