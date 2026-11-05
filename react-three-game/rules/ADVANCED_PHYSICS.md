# Crashcat physics

Crashcat is an optional adapter that advances its own physics world from R3F frames. Keep application simulation timing in the host.

Register the physics component and mount one runtime inside the prefab. The runtime
advances physics in Play mode; component schemas supply inspector controls.

```tsx
import { GameCanvas, PrefabRoot, registerComponent } from 'react-three-game/viewer';
import { CrashcatPhysicsComponent, CrashcatRuntime } from 'react-three-game/plugins/crashcat';

registerComponent(CrashcatPhysicsComponent);

// level is the app's prefab document.
<GameCanvas>
  <PrefabRoot data={level}>
    <CrashcatRuntime />
  </PrefabRoot>
</GameCanvas>
```

For an editor, mount `CrashcatRuntime` as a child of `PrefabEditor`.

Add physics alongside a node's visual components:

```json
{
  "physics": {
    "type": "CrashcatPhysics",
    "properties": { "type": "dynamic", "colliders": "ball", "friction": 0.6 }
  }
}
```

| Property | Values |
| --- | --- |
| `type` | `fixed`, `dynamic`, `kinematicPosition`, `kinematicVelocity` |
| `colliders` | `cuboid`, `ball`, `capsule`, `cylinder`, `hull`, `trimesh` |
| `sensor` | Enables trigger contacts |
| `linearVelocity`, `angularVelocity` | Initial velocity tuples |
| `friction`, `restitution` | Surface response |
| `collisionEnterEventName`, `collisionExitEventName` | Collision event names |
| `sensorEnterEventName`, `sensorExitEventName` | Sensor event names |

Call `useGameObject()` (or pass a local node ID) in the component body. Inside the frame callback, read its `.getComponent(RIGID_BODY_COMPONENT)` handle; import `RIGID_BODY_COMPONENT` from the Crashcat plugin. It returns null until the body mounts; instance scoping is automatic. Drive active bodies through the Crashcat handle; use document edits for authored settings.

Input/controller frame callbacks use priority `-2`, before the physics step at `-1`. Check Play mode and body availability. Reuse scratch vectors and keep input state in refs.

Contact events include source/target entity IDs and node IDs, plus an optional collision normal. Listen with `useGameEvent`; a Sound component can listen through its `eventName` property.
