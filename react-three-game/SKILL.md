---
name: react-three-game
description: "Build games with React Three Game using JSON prefab documents, R3F component composition, the scene and prefab runtime APIs, the optional editor entrypoint, and optional plugins such as Crashcat physics."
---

# react-three-game

React Three Game mounts serializable prefab documents as native React Three Fiber scenes. Components define the vocabulary; prefab JSON composes that vocabulary into entities.

## Core model

| Scope | Meaning | API |
|---|---|---|
| Scene | The outer R3F scene shared by composed prefabs | `useScene()` → `{ root, mode }` |
| Prefab | One serializable document with local node ids and shared materials | `usePrefab()` → `PrefabApi` |
| Node | One entity in the active prefab | `useNode()`, `useNodeObject()`, `useNodeHandle()` |
| Component | A registered renderer/editor capability selected by `type` | `registerComponent(component)` |

Child prefabs inherit the parent scene and receive their own prefab document scope.

## Package entrypoints

| Import | Contents |
|---|---|
| `react-three-game` | `GameCanvas`, `PrefabRoot`, scene/prefab/node hooks, component registration, events, types, loaders |
| `react-three-game/editor` | `PrefabEditor`, editor ref/context, fields, prefab-store hooks, import/export tools |
| `react-three-game/plugins/crashcat` | Crashcat runtime, authored physics component, ragdoll helpers |

## Prefab syntax

```ts
interface Prefab {
  id?: string;
  name?: string;
  materials?: Record<string, PrefabMaterial>;
  root: GameObject;
}

interface GameObject {
  id: string;
  name?: string;
  disabled?: boolean;
  hidden?: boolean;
  locked?: boolean;
  components?: Record<string, ComponentData | undefined>;
  children?: GameObject[];
}

interface ComponentData<P extends object = Record<string, unknown>> {
  type: string;
  properties: P;
}
```

Use stable ids, local transforms, radians, and asset paths resolved from `basePath`.

```json
{
  "id": "starter",
  "name": "Starter Scene",
  "materials": {
    "crate": {
      "name": "Crate",
      "materialType": "standard",
      "color": "#d97706",
      "roughness": 0.7,
      "metalness": 0.1
    }
  },
  "root": {
    "id": "root",
    "children": [
      {
        "id": "camera",
        "components": {
          "transform": {
            "type": "Transform",
            "properties": {
              "position": [0, 3, 8],
              "rotation": [-0.25, 0, 0],
              "scale": [1, 1, 1]
            }
          },
          "camera": {
            "type": "Camera",
            "properties": { "projection": "perspective", "fov": 50 }
          }
        }
      },
      {
        "id": "crate",
        "components": {
          "transform": {
            "type": "Transform",
            "properties": {
              "position": [0, 0.5, 0],
              "rotation": [0, 0, 0],
              "scale": [1, 1, 1]
            }
          },
          "mesh": {
            "type": "Mesh",
            "properties": { "castShadow": true, "receiveShadow": true }
          },
          "geometry": {
            "type": "Geometry",
            "properties": { "geometryType": "box", "args": [1, 1, 1] }
          },
          "material": {
            "type": "Material",
            "properties": { "materialId": "crate" }
          }
        }
      }
    ]
  }
}
```

## Component composition

Components on a node compose like R3F children:

```tsx
<mesh>
  <boxGeometry attach="geometry" />
  <meshStandardMaterial attach="material" />
  {children}
</mesh>
```

| Built-in role | Components | Composition |
|---|---|---|
| Transform | `Transform` | Applies the node's outer position, rotation, and scale |
| Object | `Mesh`, `Model`, `Sprite`, `Text`, lights, `Camera` | Owns the node's Three.js object slot |
| Attachment | `Geometry`, `BufferGeometry`, `Material` | Attaches to the enclosing object |
| Wrapper | `Sound`, `Data`, `PrefabRef`, custom behavior | Wraps the composed object and children |

Materials live in `Prefab.materials`. A `Material` component selects one with `materialId`; every reference in that prefab shares the same Three.js material object. `attach` supports R3F attachment paths such as `material`.

## Mounting

Runtime:

```tsx
import { GameCanvas, PrefabRoot, type Prefab } from 'react-three-game';
import sceneJson from './scene.json';

const prefab = sceneJson as Prefab;

export function Game() {
  return (
    <GameCanvas>
      <PrefabRoot data={prefab} basePath="/game" />
    </GameCanvas>
  );
}
```

Editor:

```tsx
import { PrefabEditor } from 'react-three-game/editor';

export function Authoring() {
  return <PrefabEditor prefab={prefab} basePath="/game" />;
}
```

`PrefabRoot.data` and `PrefabEditor.prefab` are document inputs. A new object value loads the new document. Editor mutations update the current document and `ref.save()` returns it. `mode` defaults to `PrefabEditorMode.Edit` and responds to new prop values.

## Runtime scopes

```tsx
import { useFrame } from '@react-three/fiber';
import {
  PrefabEditorMode,
  usePrefab,
  useScene,
} from 'react-three-game';

function SceneRuntime() {
  const scene = useScene();
  const prefab = usePrefab();

  useFrame((_, delta) => {
    if (scene.mode !== PrefabEditorMode.Play) return;
    const crate = prefab.getObject('crate');
    if (crate) crate.rotation.y += delta;
  });

  return null;
}
```

| Hook | Useful members |
|---|---|
| `useScene()` | `root`, `mode` |
| `usePrefab()` | `root`, `basePath`, node/material CRUD, live objects, handles, loaded assets |
| `useNode()` | `nodeId`, `editMode`, `isSelected`, interaction handlers, current-node getters |
| `useNodeObject<T>()` | Live ref for the current node's `Object3D` |
| `useNodeHandle<T>(kind)` | Live ref for a current-node runtime handle |
| `useAssetRuntime()` | Shared model, texture, and sound cache |

Core `PrefabApi` syntax:

```ts
prefab.get(id);
prefab.getObject(id);
prefab.getHandle(id, kind);
prefab.getModel(path);
prefab.getMaterial(materialId);

prefab.add(node, parentId);
prefab.update(id, node => nextNode);
prefab.setMaterial(materialId, material);
prefab.replaceNode(id, node);
prefab.remove(id);
prefab.duplicate(id);
prefab.move(id, targetId, 'inside');
prefab.replace(nextPrefab);
```

Authored changes use prefab mutations. Frame-by-frame motion uses live Three.js objects.

## Custom components

Component fields infer their property names and value types from `FieldDefinition<P>`.

```tsx
import { useFrame } from '@react-three/fiber';
import {
  registerComponent,
  useNode,
  useNodeObject,
  type Component,
  type ComponentViewProps,
} from 'react-three-game';
import {
  FieldRenderer,
  type ComponentEditorProps,
  type FieldDefinition,
} from 'react-three-game/editor';

type RotatorProperties = {
  speed?: number;
  axis?: 'x' | 'y' | 'z';
};

const fields = [
  { name: 'speed', type: 'number', label: 'Speed', step: 0.1 },
  {
    name: 'axis',
    type: 'select',
    label: 'Axis',
    options: [
      { value: 'x', label: 'X' },
      { value: 'y', label: 'Y' },
      { value: 'z', label: 'Z' },
    ],
  },
] satisfies FieldDefinition<RotatorProperties>[];

function RotatorEditor({ properties, update }: ComponentEditorProps<RotatorProperties>) {
  return <FieldRenderer fields={fields} values={properties} onChange={update} />;
}

function RotatorView({ properties, children }: ComponentViewProps<RotatorProperties>) {
  const { editMode } = useNode();
  const objectRef = useNodeObject();

  useFrame((_, delta) => {
    const object = objectRef.current;
    if (editMode || !object) return;
    object.rotation[properties.axis ?? 'y'] += delta * (properties.speed ?? 1);
  });

  return <>{children}</>;
}

const Rotator: Component<RotatorProperties> = {
  name: 'Rotator',
  Editor: RotatorEditor,
  View: RotatorView,
  defaultProperties: { speed: 1, axis: 'y' },
};

registerComponent(Rotator);
```

JSON usage:

```json
{
  "rotator": {
    "type": "Rotator",
    "properties": { "speed": 1.5, "axis": "y" }
  }
}
```

## Handles and events

```tsx
import { useEffect, useRef } from 'react';
import {
  useNode,
  usePrefab,
  type ComponentViewProps,
} from 'react-three-game';

function DoorView({ children }: ComponentViewProps) {
  const { nodeId } = useNode();
  const prefab = usePrefab();
  const open = useRef(false);

  useEffect(() => {
    const handle = { open: () => { open.current = true; } };
    prefab.registerHandle(nodeId, 'door', handle);
    return () => prefab.registerHandle(nodeId, 'door', null);
  }, [nodeId, prefab]);

  return <>{children}</>;
}
```

| Communication | Syntax |
|---|---|
| Broadcast | `gameEvents.emit(name, payload)` / `useGameEvent(name, handler, deps)` |
| Cross-node handle | `prefab.getHandle<T>(nodeId, kind)` |
| Current-node handle | `useNodeHandle<T>(kind).current` |
| Node pointer stream | `onPointerEvent={(type, event, node) => ...}` |
| Named click | `Mesh.properties.emitClickEvent` + `clickEventName` |

## Nested prefabs and cameras

```json
{
  "id": "room-instance",
  "components": {
    "prefab": {
      "type": "PrefabRef",
      "properties": { "url": "/prefabs/room.json" }
    }
  }
}
```

Each nested prefab keeps local node ids and materials while sharing the outer scene mode. Camera nodes become the default camera in Play mode and render as wireframes in Edit mode.

## Built-ins

| Category | Components |
|---|---|
| Core | `Transform`, `Mesh`, `Geometry`, `BufferGeometry`, `Material` |
| Assets | `Model`, `Sprite`, `Text`, `Sound`, `PrefabRef` |
| Scene | `Camera`, `Environment`, `AmbientLight`, `DirectionalLight`, `PointLight`, `SpotLight` |
| Data | `Data` |

## Focused references

| Task | Reference |
|---|---|
| Runtime managers, handles, events | [`rules/RUNTIME_INTEGRATIONS.md`](rules/RUNTIME_INTEGRATIONS.md) |
| Crashcat physics | [`rules/ADVANCED_PHYSICS.md`](rules/ADVANCED_PHYSICS.md) |
| Authored lighting and shadows | [`rules/LIGHTING.md`](rules/LIGHTING.md) |
| Frame loops, instancing, streaming | [`rules/PERFORMANCE.md`](rules/PERFORMANCE.md) |

Validate consuming projects with a production build and exercise component behavior in Edit and Play modes.
