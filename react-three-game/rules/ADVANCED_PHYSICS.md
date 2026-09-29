# Crashcat physics

Crashcat is an optional adapter that advances its own physics world from R3F frames. Its stepping policy is not a core game-loop or fixed world-tick contract.

Register `CrashcatPhysicsComponent` from `react-three-game/plugins/crashcat`. Mount one `CrashcatRuntime` inside `PrefabRoot` or `PrefabEditor`. Inspector controls are generated from the component schemas.

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

Use `useGameObject()` (or pass a local node ID) and read `.getComponent(RIGID_BODY_COMPONENT)` from the Crashcat plugin inside the frame callback. It returns null until the body mounts; instance scoping is automatic. Drive bodies through Crashcat; do not animate dynamic bodies by editing the document.

Input/controller frame callbacks use priority `-2`, before the physics step at `-1`. Check Play mode and body availability. Reuse scratch vectors and keep input state in refs.

Contact events include source/target entity IDs and node IDs, plus an optional collision normal. Listen with `useGameEvent`; a Sound component can listen through its `eventName` property.
