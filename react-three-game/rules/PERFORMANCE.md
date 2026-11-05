# Performance

Compare the same scene, camera path, resolution, and lighting in a production build.
Measure startup, content changes, steady frames, and memory after unloading.

- Animate live Three objects in `useFrame`; keep scratch vectors and input in refs.
- Compatible leaf meshes with shared geometry/materials instance automatically.
  Use `instanced:false` for meshes needing ordinary per-object rendering.
- Matching built-in material settings share resources across names. Reuse a name
  when edits should affect all users. Custom shaders use `useSharedMaterialResource`.
- Use `scene.analyze()` for authoring hints and a profiler for draw calls and FPS.
- Start with one shadow caster. Cache shadows for static scenes; use cascades for
  moving outdoor views.

Use `PrefabRoot` for inline documents and `PrefabInstance` for URL-backed
placements. Configure root `ChunkSettings` and static placements for distance-based
streaming. Applications own world layout and travel policy. R3G shares assets and
batches compatible meshes; Three.js handles frustum culling.

Profile mounting separately: React mounting, geometry work, and asset decoding
can cost CPU time even when the final scene has few draw calls.
