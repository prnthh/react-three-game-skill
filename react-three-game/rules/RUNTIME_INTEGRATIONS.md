# Runtime communication

Use component properties for editable configuration and refs/live objects for simulation. Mount shared systems as children of `PrefabRoot` or `PrefabEditor`.

For a known node in the same prefab, use `useGameObject(id)`: `.transform` reads the live Three object and `.getComponent(type)` reads its runtime handle. Omit the id for the current node. Both reads return null when unavailable. For notifications, use `useGameEvents().emit(name, payload)` and `useGameEvent(name, handler, deps)`.

When a system needs to query many nodes by capability:

```tsx
import { createNodeComponentType, useRegisterNodeComponent,
  useSceneComponents } from 'react-three-game/viewer';

const HEALTH = createNodeComponentType<{ damage(amount: number): void }>('Health');

// Inside the node's component view; keep health's identity stable:
useRegisterNodeComponent(HEALTH, health);

// Inside a scene system:
const actors = useSceneComponents(HEALTH);
// Each entry exposes .value, with the registered API.
```

Capabilities enter and leave the query when nodes mount and unmount. Use this only when direct node access or events are insufficient.

`AnimatedModel` publishes `ANIMATED_MODEL_COMPONENT` for animation controls. Game-specific state transitions belong in gameplay code.

Runtime systems should check `useScene().mode` before simulating in the editor. Node behaviors can check `useNode().editMode`.
