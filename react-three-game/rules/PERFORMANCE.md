# Performance

Measure before adding a new optimization system. Compare the same scene, camera path, resolution, and lighting in a production build.

- Keep animation and physics on live objects, not document mutations.
- Reuse scratch vectors and avoid React state updates every frame.
- Eligible leaf meshes instance automatically; `instanced: false` opts out.
- Keep ordinary animated/interactive objects on their native path when batching does not fit.
- Share assets and immutable materials through the scene runtime.
- Stream URL-backed chunks with `PrefabInstance`; keep old terrain active until replacements activate.
- Use `static` only for immutable chunks. Remount to move or edit them.
- Start with one shadow caster. Use cached shadows for static scenes and CSM for moving outdoor views.

`onStatus` reports preparation phases; `onActivate` means the chunk is active. Pending downloads reaching zero does not mean rendering is prepared.

Low draw counts do not guarantee fast startup or low CPU cost. Measure startup, chunk transitions, steady frames, and memory after unloading separately. Preparation does not move React mounting or all asset decoding off the main thread.
