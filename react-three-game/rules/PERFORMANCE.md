# Performance

Compare the same scene, camera path, resolution, and lighting in a production build.
Measure startup, chunk transitions, steady frames, and memory after unloading.

- Animate live Three objects in `useFrame`; keep scratch vectors and input in refs.
- Compatible leaf meshes with shared geometry/materials instance automatically.
  Use `instanced:false` for meshes needing ordinary per-object rendering.
- Matching built-in material settings share resources across names. Reuse a name
  when edits should affect all users. Custom shaders use `useSharedMaterialResource`.
- Use `scene.analyze()` for authoring hints and a profiler for draw calls and FPS.
- Start with one shadow caster. Cache shadows for static scenes; use cascades for
  moving outdoor views.

Begin with inline prefabs and ordinary placement. When a world needs streaming,
mount URL-backed chunks with `PrefabInstance` and keep old terrain active until
replacements report `active`. `onStatus` reports preparation progress;
`onActivate` signals gameplay readiness after mounting and shader compilation.
Use `static` for immutable chunks and remount them to change placement or content.

Profile preparation separately: React mounting, geometry work, and asset decoding
can cost CPU time even when the final scene has few draw calls.
