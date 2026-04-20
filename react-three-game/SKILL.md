---
name: react-three-game
description: react-three-game, a JSON-first scene mounting and authoring library built on React Three Fiber, WebGPU, and Rapier Physics.
---

# react-three-game

Reference for prefabs, scene mounting, custom components, and direct Three.js or Rapier access in `react-three-game`.

## Scope

- Build JSON prefabs and scene graphs.
- Render prefabs with `PrefabRoot`.
- Edit scenes with `PrefabEditor`.
- Add custom components with `registerComponent()`.
- Mutate authored scenes through the prefab store or the editor/root refs.

## Schema

JSON prefab with a single root node.

```ts
interface Prefab {
  id?: string;
  name?: string;
  root: GameObject;
}

interface GameObject {
  id: string;
  name?: string;
  disabled?: boolean;
  locked?: boolean;
  children?: GameObject[];
  components?: Record<string, ComponentData | undefined>;
}

interface ComponentData {
  type: string;
  properties: Record<string, any>;
}
```

Conventions:

- Component keys are lowercase in JSON.
- Component `type` names are TitleCase.
- Transforms are local to the parent.
- Rotations use radians.
- Asset paths are relative to `/public`.
- Use `crypto.randomUUID()` for new node ids.

Example:

```json
{
  "root": {
    "id": "scene",
    "children": [
      {
        "id": "crate",
        "components": {
          "transform": { "type": "Transform", "properties": { "position": [0, 1, 0] } },
          "geometry": { "type": "Geometry", "properties": { "geometryType": "box", "args": [1, 1, 1] } },
          "material": { "type": "Material", "properties": { "color": "#c97316" } }
        }
      }
    ]
  }
}
```

## Rendering model

`PrefabRoot` composition rules:

- `Transform` is applied by the renderer as the node's outer transform.
- `Geometry` or `BufferGeometry` + `Material` are special-cased into the node's primary mesh content.
- `Model` is also special-cased as primary content when non-instanced.
- `Physics` is a renderer-owned outer wrapper.
- Every other component `View` composes by wrapping the current subtree.

Implications:

- Components like `Environment` wrap the subtree and use `children` to generate the envmap.

## PrefabRoot

Pure prefab rendering inside an R3F scene.

```tsx
import { Physics } from '@react-three/rapier';
import { GameCanvas, PrefabRoot } from 'react-three-game';

<GameCanvas>
  <Physics>
    <PrefabRoot data={prefabData} />
  </Physics>
</GameCanvas>
```

Props:

- `data?: Prefab`
- `store?: PrefabStoreApi`
- `editMode?: boolean`
- `selectedId?: string | null`
- `onSelect?: (id: string | null) => void`
- `onClick?: (event, entity) => void`
- `onObjectRefChange?: (id, object) => void`
- `basePath?: string`

Use `data` for static prefab input. Use `store` when state is owned externally.

Custom authored mesh example:

```json
{
  "id": "triangle",
  "components": {
    "bufferGeometry": {
      "type": "BufferGeometry",
      "properties": {
        "positions": [0, 0, 0, 1, 0, 0, 0, 1, 0],
        "indices": [0, 1, 2],
        "uvs": [0, 0, 1, 0, 0, 1]
      }
    },
    "material": {
      "type": "Material",
      "properties": {
        "texture": "textures/proto32/grid.png",
        "repeat": true,
        "repeatCount": [2, 2],
        "offset": [0, 0],
        "animateOffset": true,
        "offsetSpeed": [0.25, 0]
      }
    }
  }
}
```

## PrefabEditor

Managed authoring UI with history, selection, inspector editing, and play/edit mode.

```tsx
import { PrefabEditor } from 'react-three-game';

<PrefabEditor
  initialPrefab={prefabData}
  onChange={setPrefabData}
/>
```

Notes:

- Use `onChange` to receive prefab updates.
- `mode` is `edit` or `play`.
- Physics is enabled in play mode and paused in edit mode.
- `children` render inside the editor canvas.

Editor ref:

```ts
editorRef.current?.root;
editorRef.current?.store;
editorRef.current?.getObject('player');
editorRef.current?.getRigidBody('player');
editorRef.current?.addNode(node, { parentId: 'root' });
editorRef.current?.save();
editorRef.current?.load(prefab, { resetHistory: true });
editorRef.current?.exportGLB();
editorRef.current?.exportGLBData();
editorRef.current?.screenshot();
```

## Direct runtime access

Use the surface that matches where you are operating.

```tsx
const root = editorRef.current?.root;
const store = editorRef.current?.store;
const object = editorRef.current?.getObject('player');
const rigidBody = editorRef.current?.getRigidBody('player');

store?.getState().updateNode('player', (node) => ({
  ...node,
  components: {
    ...node.components,
    transform: {
      type: 'Transform',
      properties: {
        ...node.components?.transform?.properties,
        position: [5, 0, 0],
      },
    },
  },
}));

editorRef.current?.addNode({
  id: crypto.randomUUID(),
  name: 'Cube',
  components: {
    transform: { type: 'Transform', properties: { position: [0, 1, 0] } },
    geometry: { type: 'Geometry', properties: { geometryType: 'box' } },
  },
});

rigidBody?.applyImpulse({ x: 0, y: 5, z: 0 }, true);
root?.getObject('player')?.position.set(0, 2, 0);
```

Guidance:

- Use the prefab store when you want authored data changes that stay serializable.
- Use `getObject()` or `getRigidBody()` when you want raw Three.js or Rapier methods.
- Use `addNode()` for spawning authored nodes into the current prefab.

## Runtime access

Use the matching runtime surface for each operation.

### 1. Inside a component `View`

Runtime hooks.

- `useEntityRuntime()` for `editMode`, `nodeId`, and live getters.
- `useEntityObjectRef()` when you need the current `Object3D`.
- `useEntityRigidBodyRef()` when you need raw Rapier body methods.
- `useAssetRuntime().getObject(nodeId)` or `useAssetRuntime().getRigidBody(nodeId)` when the current component needs to target another authored node in the same prefab.

Node-local surface for custom components.

Guidance:

- `useEntityObjectRef()` and `useEntityRigidBodyRef()` access the current node.
- `useAssetRuntime()` looks up other authored nodes for gameplay logic like elevators, doors, linked sensors, and moving platforms.
- `useEntityRuntime()` and `useAssetRuntime()` are the runtime lookup surfaces inside component views.

### 2. Outside component views, but still operating on authored prefab nodes

Use the `PrefabEditorRef` or `PrefabRootRef` directly.

- Use `editorRef.current?.store.getState()` for authored prefab mutations.
- Use `editorRef.current?.addNode()` for lifecycle changes.
- Use `editorRef.current?.getObject(id)` and `editorRef.current?.getRigidBody(id)` for raw runtime access.
- Use `prefabRootRef.current?.getObject(id)` and `prefabRootRef.current?.getRigidBody(id)` when you are mounting with `PrefabRoot` only.

### 3. Use native Three.js or Rapier objects directly

Use `getObject(id)` or `getRigidBody(id)` for capabilities provided directly by Three.js or Rapier:

- reading world position or world rotation
- calling raw Rapier methods like `applyImpulse()`
- integrating with lower-level Three.js APIs

Guidance:

- Store mutations write authored component properties.
- `object` and `rigidBody` expose world-space transforms and engine methods.

## Custom components

Register before rendering `PrefabRoot` or `PrefabEditor`.

```tsx
import { FieldRenderer, registerComponent, type Component, type FieldDefinition } from 'react-three-game';

const fields: FieldDefinition[] = [
  { name: 'speed', type: 'number', label: 'Speed', step: 0.1 },
];

const Rotator: Component = {
  name: 'Rotator',
  Editor: ({ component, onUpdate }) => (
    <FieldRenderer fields={fields} values={component.properties} onChange={onUpdate} />
  ),
  View: ({ children }) => <group>{children}</group>,
  defaultProperties: { speed: 1 },
};

registerComponent(Rotator);
```

Field types:

- `number`
- `string`
- `boolean`
- `select`
- `vector3`
- `color`
- `node` for authored node ids with searchable name/id suggestions from the current prefab

Cross-node pattern inside a `View`:

```tsx
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import {
  FieldRenderer,
  type Component,
  type FieldDefinition,
  useAssetRuntime,
  useEntityRuntime,
} from 'react-three-game';

const fields: FieldDefinition[] = [
  { name: 'platformNodeId', type: 'node', label: 'Platform Node' },
];

function ElevatorMoverView({ properties, children }: { properties: any; children?: React.ReactNode }) {
  const { editMode } = useEntityRuntime();
  const { getRigidBody } = useAssetRuntime();
  const activeRef = useRef(true);

  useFrame(() => {
    if (editMode || !activeRef.current) return;
    const rigidBody = getRigidBody(properties.platformNodeId);
    rigidBody?.setTranslation({ x: 0, y: 4, z: 0 }, true);
  });

  return <>{children}</>;
}
```

Composition:

- The default component behavior is to wrap the current subtree.
- Custom `View` implementations compose around `children`, while `PrefabRoot` owns transform, geometry/material, model, and physics special cases.

## Built-in components

- `Transform`: local `position`, `rotation`, `scale`
- `Geometry`: `geometryType`, `args`
- `BufferGeometry`: custom `positions`, `indices`, optional `normals`, `uvs`; default triangle includes UVs
- `Material`: `color`, `texture`, `metalness`, `roughness`, `repeat`, `repeatCount`, `offset`, `animateOffset`, `offsetSpeed`, normal map options
- `Physics`: rigid body and collider settings
- `Model`: `filename`, `instanced`, repeat axes
- `AmbientLight`
- `PointLight`
- `SpotLight`
- `DirectionalLight`
- `Environment`
- `Camera`
- `Text`
- `Sound`

## Useful exports

Values:

- `GameCanvas`
- `PrefabRoot`
- `PrefabEditor`
- `PrefabEditorMode`
- `registerComponent`
- `useEditorContext`
- `createPrefabStore`
- `usePrefabStoreApi`
- `createModelNode`
- `createImageNode`
- `findComponent`
- `hasComponent`
- `useAssetRuntime`
- `useEntityRuntime`
- `useEntityObjectRef`
- `useEntityRigidBodyRef`
- `loadFiles`
- `loadModel`
- `loadSound`
- `loadTexture`
- `exportGLB`
- `computeParentWorldMatrix`
- `ground`
- `soundManager`

Types:

- `Prefab`
- `PrefabRootRef`
- `GameObject`
- `ComponentData`
- `Component`
- `ComponentViewProps`
- `PrefabRootProps`
- `PrefabEditorProps`
- `PrefabEditorRef`
- `PrefabStoreApi`
- `PrefabStoreState`
- `FieldDefinition`
- `FieldType`

## Constraints

- Preserve the JSON-first scene model.
- Keep custom components simple and let the renderer own transform, geometry/material, model, and physics behavior.
- Prefer prefab store updates for authored state and native object or rigid body refs for raw runtime control.
- When documenting or generating examples, use the current API names exactly as exported.


