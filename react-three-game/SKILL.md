---
name: react-three-game
description: react-three-game, a JSON-first 3D game engine built on React Three Fiber, WebGPU, and Rapier Physics.
---

# react-three-game

Authored scene layer on top of React Three Fiber. Scenes are JSON. Everything builds up from there.

---

## Step 1 — The Atom: a GameObject

Every entity is a `GameObject` with components keyed lowercase, types TitleCase:

```json
{
  "id": "box1",
  "components": {
    "transform": { "type": "Transform", "properties": { "position": [0, 1, 0] } },
    "geometry": { "type": "Geometry", "properties": { "geometryType": "box", "args": [1, 1, 1] } },
    "material": { "type": "Material", "properties": { "color": "#ff0000" } }
  }
}
```

Rules:
- `id` — use `crypto.randomUUID()` for new ones
- `disabled` — visibility toggle
- `children` — nested GameObjects (transforms are parent-relative)
- Asset paths relative to `/public` (e.g. `"/textures/floor.png"`)
- Rotations in radians (`1.57` = 90°)

---

## Step 2 — A Scene: the Prefab

Wrap GameObjects in a Prefab with a root node:

```json
{
  "root": {
    "id": "scene",
    "children": [
      { "id": "box1", "components": { ... } },
      { "id": "light1", "components": { ... } }
    ]
  }
}
```

---

## Step 3 — Render It: PrefabRoot

Render a prefab inside a standard R3F canvas. Add `<Physics>` if you need physics.

```jsx
import { Physics } from '@react-three/rapier';
import { GameCanvas, PrefabRoot } from 'react-three-game';

<GameCanvas>
  <Physics>
    <PrefabRoot data={prefabData} />
  </Physics>
</GameCanvas>
```

This is the lightest option. Use it for production gameplay, embedding, or headless rendering.

---

## Step 4 — Edit It: PrefabEditor

Add authoring UI, transform gizmos, inspector, play/pause, undo/redo:

```jsx
import { PrefabEditor } from 'react-three-game';

<PrefabEditor initialPrefab={prefabData} onPrefabChange={setData} />
```

Physics activates in play mode only. Keyboard: **T** translate, **R** rotate, **S** scale.

---

## Step 5 — Mutate It: Scene API

Access the imperative scene handle from the editor ref:

```tsx
const scene = editorRef.current.scene;
```

Prefer these patterns in order (most targeted first):

```tsx
// 1. Set a single property
scene.find("ball")?.getComponent("Transform")?.set("position", [5, 0, 0]);

// 2. Update when next state depends on previous
scene.find("ball")?.getComponent("Transform")?.update(p => ({ ...p, position: [p.position[0] + 1, 0, 0] }));

// 3. Whole-entity changes (disable, lock, multi-component swaps)
scene.update("ball", node => ({ ...node, disabled: true }));

// 4. Spawn a new entity
scene.create("Cube", { geometry: { type: "Geometry", properties: { geometryType: "box" } } });

// 5. Add authored node or remove
scene.add(gameObjectNode, { parentId: "root" });
scene.remove("ball");
```

`useEditorContext()` is for editor concerns (selection, transform mode, play state) — not scene content.

---

## Step 6 — Add Behavior: Custom Components

Register before rendering `<PrefabEditor>`:

```tsx
import { Component, registerComponent, FieldRenderer } from 'react-three-game';

const Rotator: Component = {
  name: 'Rotator',
  Editor: ({ component, onUpdate }) => (
    <FieldRenderer fields={[{ name: 'speed', type: 'number', step: 0.1 }]} values={component.properties} onChange={onUpdate} />
  ),
  View: ({ properties, children }) => <group>{children}</group>,
  defaultProperties: { speed: 1 }
};

registerComponent(Rotator);
```

Use `useFrame` inside `View` for runtime behavior. Field types: `vector3`, `number`, `string`, `color`, `boolean`, `select`, `custom`.

---

## Step 7 — Wire Events

The component that causes the action defines the event name. Listeners subscribe to it.

```json
{
  "click": { "type": "Click", "properties": { "eventName": "cannon:fire" } },
  "sound": { "type": "Sound", "properties": { "path": "/sound/explode.mp3", "eventName": "cannon:fire" } }
}
```

Built-in events: `click`, `sensor:enter`, `sensor:exit`, `collision:enter`, `collision:exit`. Use custom names when one action drives multiple systems.

---

## Step 8 — Export: JSON → GLB

```tsx
import { exportGLBData } from 'react-three-game';

const glbData = await exportGLBData(editorRef.current.rootRef.current.root);
// ArrayBuffer ready for upload/storage
```

---

## Built-in Components

| Type | Key Properties |
|------|----------------|
| `Transform` | `position`, `rotation` (radians), `scale` — all `[x,y,z]` |
| `Geometry` | `geometryType`: box/sphere/plane/cylinder, `args`: dimensions |
| `Material` | `color`, `texture?`, `metalness?`, `roughness?`, `repeat?`, `repeatCount?` |
| `Physics` | `type`: dynamic/fixed/kinematicPosition/kinematicVelocity, `sensor?`, `ccd?`, `mass?` |
| `Model` | `filename` (GLB/FBX), `instanced?` for GPU batching |
| `SpotLight` | `color`, `intensity`, `angle`, `penumbra`, `castShadow?` |
| `DirectionalLight` | `color`, `intensity`, `castShadow?`, `targetOffset?` |
| `AmbientLight` | `color`, `intensity` |
| `Text` | `text`, `font`, `size`, `depth`, `color` |
| `Sound` | `path`, `eventName`, `volume?`, `loop?` |
| `Click` | `eventName` |
| `Camera` | camera override |
| `Environment` | HDRI/skybox |

### Geometry args

| geometryType | args |
|---|---|
| box | `[width, height, depth]` |
| sphere | `[radius, wSeg, hSeg]` |
| plane | `[width, height]` |
| cylinder | `[rTop, rBottom, height, segments]` |

---

## Module Internals (for library contributors)

Three files, one concern each, no circular deps:

| File | Concern |
|------|---------|
| `prefab.ts` | Pure data: construction, normalization, tree ops. No React. |
| `prefabStore.ts` | Zustand store + React hooks over `PrefabState`. |
| `scene.ts` | Imperative `Scene`/`Entity`/`EntityComponent` handles. |

---

## Exports Surface

Values: `GameCanvas`, `PrefabRoot`, `PrefabEditor`, `registerComponent`, `useEditorContext`, `createScene`, `createPrefabStore`, `denormalizePrefab`, `createModelNode`, `createImageNode`, `ground`, `loadFiles`, `loadModel`, `loadTexture`, `soundManager`, `exportGLB`, `exportGLBData`

Types: `PrefabEditorRef`, `PrefabEditorProps`, `PrefabRootRef`, `PrefabRootProps`, `Scene`, `Entity`, `EntityComponent`, `PrefabStoreApi`, `PrefabStoreState`, `Prefab`, `GameObject`, `ComponentData`, `FieldDefinition`, `Component`


