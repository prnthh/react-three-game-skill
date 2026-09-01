# Crashcat physics

Crashcat is an optional plugin entrypoint for authored rigid bodies, sensors, contact events, and runtime physics queries.

## Setup

```tsx
import { registerComponent } from 'react-three-game/core';
import { PrefabEditor } from 'react-three-game/editor';
import {
  CrashcatPhysicsComponent,
  CrashcatRuntime,
  useCrashcat,
} from 'react-three-game/plugins/crashcat';

registerComponent(CrashcatPhysicsComponent);

<PrefabEditor prefab={prefab}>
  <CrashcatRuntime debug={false}>
    <PlayerController />
  </CrashcatRuntime>
</PrefabEditor>
```

One `CrashcatRuntime` owns the physics world for the shared scene.

## Authored rigid body

```json
{
  "id": "physics-scene",
  "materials": {
    "crate": {
      "color": "#c97316",
      "roughness": 0.8
    }
  },
  "root": {
    "id": "root",
    "children": [
      {
        "id": "crate",
        "components": {
          "transform": {
            "type": "Transform",
            "properties": { "position": [0, 3, 0] }
          },
          "mesh": {
            "type": "Mesh",
            "properties": {}
          },
          "geometry": {
            "type": "Geometry",
            "properties": {}
          },
          "material": {
            "type": "Material",
            "properties": { "materialId": "crate" }
          },
          "physics": {
            "type": "CrashcatPhysics",
            "properties": {
              "type": "dynamic",
              "friction": 0.6,
              "restitution": 0.1
            }
          }
        }
      }
    ]
  }
}
```

## Property table

| Property | Values |
|---|---|
| `type` | `fixed`, `dynamic`, `kinematicPosition`, `kinematicVelocity` |
| `colliders` | `cuboid`, `ball`, `capsule`, `cylinder`, `hull`, `trimesh` |
| `sensor` | Trigger-style contact body |
| `linearVelocity`, `angularVelocity` | Initial velocity tuples |
| `friction`, `restitution` | Surface response |
| `collisionEnterEventName`, `collisionExitEventName` | Named collision events |
| `sensorEnterEventName`, `sensorExitEventName` | Named sensor events |

## Controller pattern

```tsx
import { useFrame } from '@react-three/fiber';
import {
  PrefabEditorMode,
  usePrefab,
  useScene,
} from 'react-three-game/viewer';
import { useCrashcat } from 'react-three-game/plugins/crashcat';

function PlayerController({ playerId = 'player' }: { playerId?: string }) {
  const scene = useScene();
  const prefab = usePrefab();
  const crashcat = useCrashcat();

  useFrame((_, delta) => {
    if (scene.mode !== PrefabEditorMode.Play || !crashcat) return;
    const player = prefab.getObject(playerId);
    const body = crashcat.getBody(playerId);
    if (!player || !body) return;

    // Query crashcat.world, update the body, and synchronize the live object.
  }, -2);

  return null;
}
```

Transient controller state fits refs and preallocated vectors. Authored controller tuning fits component properties.

## Contact payload

```ts
type ContactEventPayload = {
  sourceEntityId: string;
  sourceNodeId: string;
  targetEntityId: string | null;
  targetNodeId: string | null;
  collisionNormal?: [number, number, number];
};
```

Named physics events connect naturally to `useGameEvent()` and authored `Sound.properties.eventName`.
