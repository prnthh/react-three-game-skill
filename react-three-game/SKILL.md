---
name: react-three-game
description: "Build games with React Three Game using JSON prefab documents, R3F component composition, the core component API, the scene and prefab runtime APIs, the optional editor entrypoint, and optional plugins such as Crashcat physics."
---

# react-three-game

React Three Game mounts serializable prefab documents as native React Three Fiber scenes. Components define the vocabulary; prefab JSON composes that vocabulary into entities.

## Core model

| Scope | Meaning | API |
|---|---|---|
| Scene | The outer R3F scene shared by composed prefabs | `useScene()` → `{ root, mode }` |
| Prefab | One serializable document with local node ids and shared materials | `usePrefab()` → `PrefabApi` |
| Node | One entity in the active prefab | `useNode()`, `useNodeObject()` |
| Component definition | A registered renderer/editor definition selected by `type` | `registerComponent(component)` |

Child prefabs inherit the parent scene and receive their own prefab document scope.

## Package entrypoints

| Import | Contents |
|---|---|
| `react-three-game/core` | Prefab JSON types and helpers, component definitions, defaults, and `registerComponent()` |
| `react-three-game/viewer` | `GameCanvas`, `PrefabRoot`, scene/prefab/node hooks, events, assets, runtime types |
| `react-three-game` | Alias of the viewer entrypoint |
| `react-three-game/editor` | `PrefabEditor`, editor ref/context, fields, prefab-store hooks, import/export tools |
| `react-three-game/plugins` | Optional plugin exports |
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
              "rotation": [-0.25, 0, 0]
            }
          },
          "camera": {
            "type": "Camera",
            "properties": {}
          }
        }
      },
      {
        "id": "crate",
        "components": {
          "transform": {
            "type": "Transform",
            "properties": {
              "position": [0, 0.5, 0]
            }
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
| Object | `Mesh`, `Model`, `Sprite`, `Text`, lights, `Camera` | Owns the node's Three.js object attachment |
| Attachment | `Geometry`, `BufferGeometry`, `Material` | Attaches to the enclosing object |
| Wrapper | `Sound`, `Data`, `PrefabRef`, custom behavior | Wraps the composed object and children |

Component definitions use an R3F-style target such as `attach: 'object'`, `attach: 'geometry'`, or `attach: 'material'` when only one component may occupy that role on a node.

Keep every component's `properties` sparse. Registered components define each property name, type, and default; only author values that differ. Keep the `properties` object itself, using `{}` when all defaults apply.

Materials live in `Prefab.materials`. A `Material` component selects one with `materialId`; matching definitions can share the same Three.js material object across nested prefabs. `attach` supports R3F attachment paths such as `material`.

Keep material JSON sparse. An absent `materials` field selects the built-in white material. Store values that differ from runtime defaults: `standard`, white, roughness `1`, metalness `0`, opacity `1`, and the usual depth, texture, and side settings are implicit. Reserve `materialType` for `basic` or `sprite`, and add `name` when a human-facing or exported material name matters.

## Mounting

Runtime:

```tsx
import type { Prefab } from 'react-three-game/core';
import { GameCanvas, PrefabRoot } from 'react-three-game/viewer';
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

`PrefabRoot.data` and `PrefabEditor.prefab` are document inputs. A new object value loads the new document. Editor mutations update the current document and `ref.save()` returns it. `PrefabEditor.mode` defaults to `PrefabEditorMode.Edit` and responds to new prop values.

## Runtime scopes

```tsx
import { useFrame } from '@react-three/fiber';
import {
  PrefabEditorMode,
  usePrefab,
  useScene,
} from 'react-three-game/viewer';

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
| `usePrefab()` | `root`, `basePath`, node/material CRUD, live objects, loaded assets |
| `useNode()` | `nodeId`, `editMode`, `isSelected`, interaction handlers, current-node getters |
| `useNodeObject<T>()` | Live ref for the current node's `Object3D` |
| `useRegisterNodeComponent(type, value)` | Publish a typed capability from the current node |
| `useSceneComponents(type)` | Query matching mounted capabilities across the scene |
| `useAssetRuntime()` | Shared model, texture, and sound cache |

Core `PrefabApi` syntax:

```ts
prefab.get(id);
prefab.getObject(id);
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

```tsx
import { useFrame } from '@react-three/fiber';
import {
  registerComponent,
  type Component,
  type ComponentViewProps,
} from 'react-three-game/core';
import {
  useNode,
  useNodeObject,
} from 'react-three-game/viewer';

type RotatorProperties = {
  speed?: number;
  axis?: 'x' | 'y' | 'z';
};

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
  View: RotatorView,
  properties: {
    speed: { default: 1, step: 0.1 },
    axis: {
      type: 'select',
      default: 'y',
      options: [
        { value: 'x', label: 'X' },
        { value: 'y', label: 'Y' },
        { value: 'z', label: 'Z' },
      ],
    },
  },
};

registerComponent(Rotator);
```

Omitting `type` means `number`. Numeric definitions may also provide `min`, `max`, and `step`; all other property types must remain explicit.
Select definitions require `options: { value, label }[]`; author the option `value` in prefab JSON.
The editor derives its ordinary inspector from this schema. Use a custom `Editor` for components with specialized inspector needs.

Register component definitions from JavaScript application or plugin setup before rendering prefab data that uses them. Keep scene JSON focused on component instances and their properties. Under the single-active-game assumption, definitions persist across prefab changes and viewer route remounts. `GameCanvas` installs missing engine built-ins through the same registry while preserving application definitions.

JSON usage:

```json
{
  "rotator": {
    "type": "Rotator",
    "properties": { "speed": 1.5 }
  }
}
```

## Component capabilities and events

```tsx
import { useMemo, useRef } from 'react';
import type { ComponentViewProps } from 'react-three-game/core';
import {
  createNodeComponentType,
  useRegisterNodeComponent,
  useSceneComponents,
} from 'react-three-game/viewer';

type Door = { open(): void };
const DOOR = createNodeComponentType<Door>('Door');

function DoorView({ children }: ComponentViewProps) {
  const open = useRef(false);
  const door = useMemo<Door>(() => ({ open: () => { open.current = true; } }), []);
  useRegisterNodeComponent(DOOR, door);
  return <>{children}</>;
}

function DoorSystem() {
  const doors = useSceneComponents(DOOR);
  return null;
}
```

| Communication | Syntax |
|---|---|
| Broadcast | `gameEvents.emit(name, payload)` / `useGameEvent(name, handler, deps)` |
| Mounted typed capability | `useRegisterNodeComponent(type, value)` / `useSceneComponents(type)` |
| Node pointer stream | `onPointerEvent={(type, event, node) => ...}` |
| Named click | `Mesh.properties.emitClickEvent` + `clickEventName` |

This mounted-capability lifecycle is separate from component-definition registration: capabilities enter and leave the scene index with their rendered nodes.

`AnimatedModel` publishes `ANIMATED_MODEL_COMPONENT`. Its capability exposes the model object, animation clips and state names, current animation state, and `setAnimationState()`, `stop()`, and `update()`; game-specific transitions remain in runtime systems.

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

Each nested prefab keeps its own document scope and local node ids while sharing the outer scene mode, asset and prefab-definition caches, material and geometry pools, component-capability index, and mesh-instancing service. Camera nodes become the default camera in Play mode and render as wireframes in Edit mode.

## Built-ins

| Category | Components |
|---|---|
| Core | `Transform`, `Mesh`, `Geometry`, `BufferGeometry`, `Material` |
| Assets | `Model`, `AnimatedModel`, `Sprite`, `Text`, `Sound`, `PrefabRef` |
| Scene | `Camera`, `Environment`, `AmbientLight`, `HemisphereLight`, `DirectionalLight`, `PointLight`, `SpotLight` |
| Data | `Data` |

## Focused references

| Task | Reference |
|---|---|
| Runtime managers, component capabilities, events | [`rules/RUNTIME_INTEGRATIONS.md`](rules/RUNTIME_INTEGRATIONS.md) |
| Crashcat physics | [`rules/ADVANCED_PHYSICS.md`](rules/ADVANCED_PHYSICS.md) |
| Authored lighting and shadows | [`rules/LIGHTING.md`](rules/LIGHTING.md) |
| Frame loops, instancing, streaming | [`rules/PERFORMANCE.md`](rules/PERFORMANCE.md) |

Validate consuming projects with a production build and exercise component behavior in Edit and Play modes.
