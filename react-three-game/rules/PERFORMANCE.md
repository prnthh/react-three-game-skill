# Performance

Measure before adding a new optimization system. Compare the same scene, camera path, resolution, and lighting in a production build.

- Keep animation and physics on live objects, not document mutations.
- Reuse scratch vectors and avoid React state updates every frame.
- Compatible leaf meshes with shared geometry/materials instance automatically; `instanced: false` opts out. For repeated boxes, share unit geometry and use transform scale for dimensions.
- `api.analyzeScene()` and batch advisories flag common authoring problems. They do not measure runtime draw calls or FPS. Named groups organize content but do not themselves reduce draw calls.
- Keep ordinary animated/interactive objects on their native path when batching does not fit.
- Built-in material definitions with matching settings share automatically across IDs. Reuse IDs for linked editing, not merely for batching. Custom shaders still use `useSharedMaterialResource`; imported models retain their asset materials.
- Stream URL-backed chunks with `PrefabInstance`; keep old terrain active until replacements activate.
- Use `static` only for immutable chunks. Remount to move or edit them.
- Start with one shadow caster. Use cached shadows for static scenes and CSM for moving outdoor views.

`onStatus` reports preparation phases; `onActivate` means the chunk is active. Pending downloads reaching zero does not mean rendering is prepared.

Low draw counts do not guarantee fast startup or low CPU cost. Measure startup, chunk transitions, steady frames, and memory after unloading separately. Preparation does not move React mounting or all asset decoding off the main thread.
