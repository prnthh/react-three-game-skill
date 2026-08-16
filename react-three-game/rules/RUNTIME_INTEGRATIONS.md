# Runtime integrations

Runtime systems live as R3F children of `PrefabRoot` or `PrefabEditor`. They share the outer scene and the current prefab document.

## Scope table

| Need | API |
|---|---|
| Edit/Play mode | `useScene().mode` |
| Current document nodes | `usePrefab()` |
| Current component node | `useNode()` / `useNodeObject()` |
| Broadcast communication | `gameEvents` / `useGameEvent()` |
| Direct component API | `prefab.registerHandle()` / `prefab.getHandle()` |
| Shared loaded assets | `useAssetRuntime()` |

## Component registration pattern

```tsx
import { useEffect } from 'react';
import {
  useNode,
  type Component,
  type ComponentViewProps,
} from 'react-three-game';
import {
  FieldRenderer,
  type ComponentEditorProps,
  type FieldDefinition,
} from 'react-three-game/editor';
import { proximityRuntime } from './proximityRuntime';

type ProximityProperties = {
  radius?: number;
  eventName?: string;
};

const fields = [
  { name: 'radius', type: 'number', label: 'Radius', min: 0, step: 0.25 },
  { name: 'eventName', type: 'string', label: 'Event' },
] satisfies FieldDefinition<ProximityProperties>[];

function ProximityEditor({ properties, update }: ComponentEditorProps<ProximityProperties>) {
  return <FieldRenderer fields={fields} values={properties} onChange={update} />;
}

function ProximityView({ properties, children }: ComponentViewProps<ProximityProperties>) {
  const { nodeId } = useNode();

  useEffect(() => {
    proximityRuntime.register(nodeId, {
      radius: properties.radius ?? 1,
      eventName: properties.eventName ?? '',
    });
    return () => proximityRuntime.unregister(nodeId);
  }, [nodeId, properties.eventName, properties.radius]);

  return <>{children}</>;
}

export const Proximity: Component<ProximityProperties> = {
  name: 'Proximity',
  Editor: ProximityEditor,
  View: ProximityView,
  defaultProperties: { radius: 1, eventName: '' },
};
```

Prefab syntax:

```json
{
  "id": "door-zone",
  "components": {
    "transform": {
      "type": "Transform",
      "properties": { "position": [0, 0, -3] }
    },
    "proximity": {
      "type": "Proximity",
      "properties": { "radius": 2, "eventName": "door:near" }
    }
  }
}
```

Manager syntax:

```ts
type ProximityEntry = {
  id: string;
  radius: number;
  eventName: string;
  active: boolean;
};

const entries: ProximityEntry[] = [];

export const proximityRuntime = {
  entries,
  register(id: string, values: Pick<ProximityEntry, 'radius' | 'eventName'>) {
    const entry = entries.find(current => current.id === id);
    if (entry) Object.assign(entry, values);
    else entries.push({ id, ...values, active: false });
  },
  unregister(id: string) {
    const index = entries.findIndex(entry => entry.id === id);
    if (index >= 0) entries.splice(index, 1);
  },
};
```

## Shared manager pattern

```tsx
import { useFrame } from '@react-three/fiber';
import {
  gameEvents,
  PrefabEditorMode,
  usePrefab,
  useScene,
} from 'react-three-game';
import { Vector3 } from 'three';
import { proximityRuntime } from './proximityRuntime';

const playerPosition = new Vector3();
const zonePosition = new Vector3();

export function ProximitySystem({ playerId = 'player' }: { playerId?: string }) {
  const scene = useScene();
  const prefab = usePrefab();

  useFrame(() => {
    if (scene.mode !== PrefabEditorMode.Play) return;
    const player = prefab.getObject(playerId);
    if (!player) return;
    player.getWorldPosition(playerPosition);

    const zones = proximityRuntime.entries;
    for (let index = 0; index < zones.length; index += 1) {
      const zone = zones[index];
      const object = prefab.getObject(zone.id);
      if (!object) continue;
      object.getWorldPosition(zonePosition);

      const active = playerPosition.distanceToSquared(zonePosition) <= zone.radius * zone.radius;
      if (active === zone.active) continue;
      zone.active = active;
      if (active && zone.eventName) {
        gameEvents.emit(zone.eventName, {
          sourceNodeId: playerId,
          targetNodeId: zone.id,
        });
      }
    }
  });

  return null;
}
```

Mounting syntax:

```tsx
registerComponent(Proximity);

<PrefabEditor prefab={prefab}>
  <ProximitySystem playerId="player" />
</PrefabEditor>
```

## Runtime data placement

| Data | Location |
|---|---|
| Inspector properties and relationships | Component `properties` |
| Serializable node changes | `PrefabApi` mutations |
| Animation and simulation state | Refs, live `Object3D`s, runtime managers |
| Direct node capability | Registered handle |
| Scene-wide notification | Named game event |

Stable arrays, indexed loops, preallocated math objects, and one manager callback form the preferred shape for large entity populations.
