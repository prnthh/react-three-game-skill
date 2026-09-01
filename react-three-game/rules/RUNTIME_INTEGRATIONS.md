# Runtime integrations

Runtime systems mount as R3F children of `PrefabRoot` or `PrefabEditor`. Components publish typed capabilities when their nodes mount; systems query the shared scene capability index.

| Need | API |
|---|---|
| Edit/Play mode | `useScene().mode` |
| Current document and live objects | `usePrefab()` |
| Current component node | `useNode()` / `useNodeObject()` |
| Typed mounted capabilities | `useRegisterNodeComponent()` / `useSceneComponents()` |
| Broadcast notification | `gameEvents` / `useGameEvent()` |
| Shared loaded assets | `useAssetRuntime()` |

## Component and system pattern

Keep inspector configuration in component properties and runtime state in the published capability. Memoize the capability when its identity should remain stable.

```tsx
import { useFrame } from '@react-three/fiber';
import { useMemo } from 'react';
import type { Object3D } from 'three';
import {
  registerComponent,
  type Component,
  type ComponentViewProps,
} from 'react-three-game/core';
import {
  PrefabEditorMode,
  createNodeComponentType,
  useNodeObject,
  useRegisterNodeComponent,
  useScene,
  useSceneComponents,
  type LiveRef,
} from 'react-three-game/viewer';

type SpinnerProperties = { speed?: number };
type Spinner = { speed: number; object: LiveRef<Object3D> };

export const SPINNER = createNodeComponentType<Spinner>('Spinner');

function SpinnerView({ properties, children }: ComponentViewProps<SpinnerProperties>) {
  const object = useNodeObject();
  const spinner = useMemo(() => ({
    object,
    speed: properties.speed ?? 1,
  }), [object, properties.speed]);

  useRegisterNodeComponent(SPINNER, spinner);
  return <>{children}</>;
}

export const SpinnerComponent: Component<SpinnerProperties> = {
  name: 'Spinner',
  View: SpinnerView,
  properties: {
    speed: { default: 1, step: 0.1 },
  },
};

export function SpinnerSystem() {
  const scene = useScene();
  const spinners = useSceneComponents(SPINNER);

  useFrame((_, delta) => {
    if (scene.mode !== PrefabEditorMode.Play) return;
    for (let index = 0; index < spinners.length; index += 1) {
      const spinner = spinners[index].value;
      const object = spinner.object.current;
      if (object) object.rotation.y += spinner.speed * delta;
    }
  });

  return null;
}
```

```tsx
import { PrefabEditor } from 'react-three-game/editor';

registerComponent(SpinnerComponent);

<PrefabEditor prefab={prefab}>
  <SpinnerSystem />
</PrefabEditor>
```

Use the scene capability index as the source for mount, replacement, and unmount notification. Register component definitions once from JavaScript setup and let them persist across scene changes.

## Data placement

| Data | Location |
|---|---|
| Inspector configuration and relationships | Component `properties` |
| Serializable scene changes | `PrefabApi` mutations |
| Per-node runtime API or state | Typed component capability |
| Animation and simulation state | Refs and live `Object3D`s |
| Scene-wide notification | Named game event |

For large populations, keep one system frame callback, use indexed loops, preallocate scratch math objects, and reuse values throughout the frame loop.
