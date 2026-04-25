# Advanced Runtime Bindings

`react-three-game` is a renderer and editor. Runtime systems such as collision, navigation, gameplay state, and authoring-time debug overlays should be integrated from userland.

## Core model

Use a two-layer setup:

- Authored scene data lives in prefab JSON through components like `Transform`, `Geometry`, `Model`, `Sound`, and `Data`.
- Runtime systems live beside `PrefabRoot` or `PrefabEditor` and read the mounted scene through refs, hooks, and user-authored metadata.

That keeps the core library focused on prefab mounting and editing while letting docs apps or downstream apps bring their own runtime.

## Runtime binding surfaces

Use the matching surface for the integration point:

| Situation | Surface | Purpose |
|----------|---------|---------|
| External system mounted beside the editor | `useScene()`, `scene.root`, `scene.getObject(id)`, `scene.getHandle(id, kind)` | Read mounted objects and runtime handles |
| External system needs edit-time re-sync | `useScene()`, `scene.revision` | Rebuild derived state only when authored data changes |
| Custom component needs its own node object | `useCurrentNodeObject()` | Read and mutate the current mounted `Object3D` |
| Custom component needs a node-local imperative handle | `useCurrentNodeHandle(kind)` | Read a live handle registered for the current node |
| Custom component wants to expose a handle | `useAssetRuntime().registerNodeHandle(nodeId, kind, handle)` | Publish runtime state back into the tree |
| Custom component targets another authored node | `useAssetRuntime().getObject(id)` or `getHandle(id, kind)` | Cross-node runtime coordination |

## Authored physics pattern

Author physics and collider config in dedicated custom components, not in `Data` blobs mirrored into `object.userData`.

Use `Data` only for generic app metadata that is not worth a first-class component.

Example:

```json
{
  "id": "wall",
  "components": {
    "transform": {
      "type": "Transform",
      "properties": {
        "position": [0, 1, 0],
        "rotation": [0, 0.4, 0]
      }
    },
    "geometry": {
      "type": "Geometry",
      "properties": {
        "geometryType": "box",
        "args": [4, 2, 0.4]
      }
    },
    "crashcatPhysics": {
      "type": "CrashcatPhysics",
      "properties": {
        "shape": "autoBox",
        "motionType": "static"
      }
    }
  }
}
```

Guidance:

- Put authored runtime config on the same node as the rendered object when possible.
- Prefer dedicated authored components like `CrashcatPhysics` for gameplay-facing systems.
- Prefer stable ids and authored component data over name-based lookups.
- Do not teach or rely on writing physics config into `object.userData` from component `View`s.

## External runtime mounted beside `PrefabEditor`

This is the normal pattern for docs-side systems like collision or controller runtimes.

```tsx
import { useEffect, useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import { useScene } from 'react-three-game';

function RuntimeBinding() {
  const dirtyRef = useRef(true);
  const scene = useScene();

  useEffect(() => {
    dirtyRef.current = true;
  }, [scene.revision]);

  useFrame(() => {
    const root = scene.root;
    if (!root || !dirtyRef.current) return;

    dirtyRef.current = false;

    root.traverse((object) => {
      const nodeId = object.userData?.prefabNodeId;
      if (typeof nodeId !== 'string') return;

      const node = scene.get(nodeId);
      const crashcatPhysics = node?.components?.crashcatPhysics;
      void crashcatPhysics;
    });
  });

  return null;
}
```

Guidance:

- Rebuild derived runtime state from `scene.revision`, not every edit-mode frame.
- Keep steady-state play mode free of edit-time polling when possible.
- Treat the mounted Three scene as the canonical runtime source of truth for transforms.
- Treat authored prefab components as the canonical source of truth for physics and gameplay config.

## Node-local handle pattern

When a custom component owns imperative runtime state, register a handle so other code can access it without searching outside the mounted node tree.

```tsx
import { useEffect, useRef } from 'react';
import { useAssetRuntime, useCurrentNode, type Component } from 'react-three-game';

function RotatorView({ children }: { children?: React.ReactNode }) {
  const { nodeId } = useCurrentNode();
  const { registerNodeHandle } = useAssetRuntime();
  const speedRef = useRef(1);

  useEffect(() => {
    const handle = {
      get speed() {
        return speedRef.current;
      },
      setSpeed(next: number) {
        speedRef.current = next;
      },
    };

    registerNodeHandle(nodeId, 'rotator', handle);
    return () => registerNodeHandle(nodeId, 'rotator', null);
  }, [nodeId, registerNodeHandle]);

  return <>{children}</>;
}

const Rotator: Component = {
  name: 'Rotator',
  View: RotatorView,
};
```

Another component or an editor child can then read that handle through `getHandle(id, kind)` or `useCurrentNodeHandle(kind)`.

## Controller pattern

Put authored visuals and tuning on prefab nodes, and keep transient controller state in refs.

- Read input into refs.
- Read the active camera or facing direction with `useThree()`.
- Use `useCurrentNodeObject()` or `scene.getObject(id)` for the mounted transform target.
- Keep per-frame controller state in refs, not prefab properties.
- Emit gameplay events through `gameEvents` when needed.

Minimal pattern:

```tsx
import { useEffect, useRef } from 'react';
import { useFrame, useThree } from '@react-three/fiber';
import { useCurrentNode, useCurrentNodeObject } from 'react-three-game';

function PlayerView({ children }: { children?: React.ReactNode }) {
  const objectRef = useCurrentNodeObject();
  const { editMode } = useCurrentNode();
  const inputRef = useRef({ forward: false, backward: false, left: false, right: false });
  const { camera } = useThree();

  useEffect(() => {
    const onKeyDown = (event: KeyboardEvent) => {
      if (event.code === 'KeyW') inputRef.current.forward = true;
      if (event.code === 'KeyS') inputRef.current.backward = true;
    };
    const onKeyUp = (event: KeyboardEvent) => {
      if (event.code === 'KeyW') inputRef.current.forward = false;
      if (event.code === 'KeyS') inputRef.current.backward = false;
    };

    window.addEventListener('keydown', onKeyDown);
    window.addEventListener('keyup', onKeyUp);
    return () => {
      window.removeEventListener('keydown', onKeyDown);
      window.removeEventListener('keyup', onKeyUp);
    };
  }, []);

  useFrame((_, delta) => {
    if (editMode) return;

    const object = objectRef.current;
    if (!object) return;

    object.position.z -= delta * Number(inputRef.current.forward);
    object.lookAt(camera.position.x, object.position.y, camera.position.z);
  });

  return <>{children}</>;
}
```

## Edit-mode re-sync

If a runtime derives colliders, bounds, or cached spatial data from authored nodes:

- subscribe to scene changes from the editor
- mark the external runtime dirty
- rebuild once on the next frame or task tick
- avoid full edit-mode rebuild work when nothing changed

This keeps edit mode responsive without adding non-stop sync cost to play mode.

## Bounds and rotation guidance

When building runtime boxes from authored meshes:

- compute box extents in object-local space
- transform only the local center into world space
- use the mounted object quaternion for orientation

Do not derive a world-space axis-aligned bounding box and then also rotate it, because that double-bakes rotation into the result.

## Event-driven runtime flow

Use `gameEvents` for gameplay events, not browser globals.

- Geometry clicks can emit named events.
- Sound can listen to named events.
- Custom controller code can emit gameplay events like footsteps, triggers, or interactions.

Typical split:

- `Data` stores authored runtime config.
- `getObject()` and `useCurrentNodeObject()` provide scene-native access.
- `getHandle()` and `useCurrentNodeHandle()` provide runtime-owned imperative access.
- `scene.revision` provides edit-time re-sync.
