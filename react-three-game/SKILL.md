---
name: react-three-game
description: Build and edit React Three Game scenes using WebGPU rendering, React Three Fiber composition, prefab components and the editor API for agents.
---

# React Three Game

React Three Game is a game creation toolbox for the web. A **prefab** is a JSON scene document: nodes describe objects, and components give them settings and behavior. Agents can work through the running editor, edit those documents directly, or add components in application source. Choose the path that fits the task and available files.

## Choose how to work

| Task or context | Approach |
| --- | --- |
| Adjust a visible scene and inspect placement | [Use the editor API](#edit-a-scene) |
| Change a saved scene, generate content, or review a file diff | [Edit prefab JSON directly](#edit-prefab-json-directly) |
| Add reusable behavior, hooks, or a new rendering feature | [Add custom components in source](#custom-components) |
| Store a small script with the scene | [Use a Runtime component](#scripted-behavior) |
| Render JSON alongside the app's own canvas content | [Embed PrefabRoot](#render-json-in-the-apps-canvas) |
| Connect gameplay, physics, lighting, or loading | [State and resources](#state-and-resources) and [Focused references](#focused-references) |

These approaches can be combined: add a component in source, configure instances
in JSON, then inspect them in the editor or viewer. Use the existing schemas and
nearby application code to discover conventions before inventing new ones.

## Edit a scene

### Connect and find the object

Run JavaScript in the frame containing `PrefabEditor`:

```js
const scene = window.scene;
scene.info(); // Check mode and how this scene can be saved.
const { node } = scene.find({ query: 'north wall' }); // Requires one match.
await scene.setMode({ mode: 'edit' });
scene.look({ id: node.id });
```

Read only what the edit needs:

```js
scene.search({ query: 'wall', limit: 20 }); // Choose an ID when find is ambiguous.
scene.get({ id: node.id, resolved: true }); // Full properties and defaults.
scene.components({ names: ['Material'] }); // Supported settings for a type.
```

Component instance keys belong to the node and can differ from type names.

If `window.scene` is missing, check the frame and wait for the editor to mount. `agentTools={false}` disables exposure; `showUI={false}` only hides panels. A viewer alone has no window API. Only one agent-enabled editor is supported per page; reacquire it after navigation.

### Make one change and inspect it

Writes require Edit mode. Start with a suitable existing object and preserve the scene's hierarchy and visual conventions.

```js
await scene.setMode({ mode: 'edit' });
scene.update({ id: node.id, transform: { position: [2, 1, 0] } });
scene.look({ id: node.id });
const image = await scene.capture({ helpers: false });
```

Wait for assets, then display `image.dataUrl` and inspect the result. Each update is one undo step.

```js
scene.undo(); // Reverse an authored edit when needed.
await scene.setMode({ mode: 'play' }); // Check animation or physics.
// Observe the behavior before continuing.
await scene.setMode({ mode: 'edit' });
await scene.reset(); // Restore live objects from current JSON; keep edits/history.
```

Changing mode alone does not reset live state.

### Save the result

```js
if (scene.info().saveMethod === 'save') {
  await scene.save(); // Host's onSaveScene callback.
} else {
  const json = scene.exportJSON(); // Write this string to the scene's source file.
}
```

Exporting alone does not save. Report what changed, what you verified, and where it was saved.

### Find a more specific operation

`scene.help()` lists methods. Read the relevant section of [Editor scene for agents](https://prnth.com/react-three-game/editor-scene-for-agents.md) for capture options, component edits, placement, downloads, packing, or atomic batches. In this checkout, use `docs/public/editor-scene-for-agents.md`; deployed docs may lag.

Individual edits use the current revision automatically. Use `expectedRevision` when coordinating against a previous read; `validate()` and `batch()` require it. If a guarded write conflicts, reread and replan. Prefer batches when dependent changes must land together, rather than for every edit.

## Edit prefab JSON directly

Find the file the app actually loads; in this repository, scenes live under `docs/public/prefabs`.

```js
// In the consuming app: follow this import (or the PrefabInstance URL).
import level from './level.json';

// Edit level.json directly; no running editor is needed.
// Preserve node IDs, component keys, and unrelated fields.
// Check component schemas in source before adding properties.
// Check JSON syntax, unique IDs, and component registration in the app.
// Reload the editor/viewer, inspect the result, and review the file diff.

// File edits do not update an already-open editor's in-memory document.
// Reload before continuing API work so a stale save cannot overwrite the file.
```

## Render JSON in the app's canvas

Render JSON beside the user's own JSX and gameplay systems. Import custom registrations before mounting; keep `data` stable between renders.

### With GameCanvas

```tsx
import './components'; // Application component registrations.
import { GameCanvas, PrefabRoot } from 'react-three-game/viewer';
import type { Prefab } from 'react-three-game/core';
import level from './level.json';

<GameCanvas>
  <ambientLight intensity={1} />
  <PrefabRoot data={level as Prefab} />
  <mesh position={[3, 0, 0]}>
    <boxGeometry />
    <meshStandardMaterial color="orange" />
  </mesh>
</GameCanvas>
```

### With an existing R3F Canvas

Keep the app's existing WebGPU setup. This shows a minimal renderer callback if needed:

```tsx
import { Canvas } from '@react-three/fiber';
import { WebGPURenderer } from 'three/webgpu';
import { PrefabRoot } from 'react-three-game/viewer';
import type { Prefab } from 'react-three-game/core';
import './components';
import level from './level.json';

<Canvas gl={async ({ canvas }) => {
  const renderer = new WebGPURenderer({ canvas });
  await renderer.init();
  return renderer;
}}>
  <PrefabRoot data={level as Prefab} />
  {/* Existing JSX, lights, controls, and gameplay systems stay here. */}
</Canvas>
```

### Share resources or access the prefab

Inside the canvas, using the app's `Gameplay` component:

```tsx
import { SceneRuntime, PrefabRoot } from 'react-three-game/viewer';

<SceneRuntime>
  <PrefabRoot id="level" data={level}>
    <Gameplay /> {/* Can use usePrefab() for this document. */}
  </PrefabRoot>
  <PrefabRoot id="props" data={propsPrefab} />
  {/* Sibling userland content shares runtime resources, not a prefab context. */}
</SceneRuntime>
```

`GameCanvas` already supplies `SceneRuntime`; a standalone `PrefabRoot` supplies
its own runtime when needed. These providers do not supply a game loop.

No editor or window API is required. For visual authoring, mount `PrefabEditor`
from `react-three-game/editor`; see the [editor guide](https://prnth.com/react-three-game/editor-scene-for-agents.md).

## Prefab conventions

Annotated JSON below uses comments for teaching; omit comments in `.json` files.

```jsonc
{
  // Shared material IDs link edits across nodes using that ID.
  "materials": { "stone": { "color": "#999999" } },
  "root": {
    "id": "world", // IDs are unique within this prefab.
    "children": [{
      "id": "box", // Keep IDs stable across edits.
      "components": {
        // Instance keys (left) can differ from registered type names (right).
        "transform": {
          "type": "Transform",
          "properties": {
            "position": [0, 1, 0], // Local to the parent; Y is up.
            "rotation": [0, 0, 0], // XYZ Euler radians.
            "scale": [2, 1, 1]     // Also scales children and colliders.
          }
        },
        "mesh": { "type": "Mesh", "properties": {} },
        "geometry": {
          "type": "Geometry",
          "properties": { "geometryType": "box", "args": [1, 1, 1] }
        },
        // Repeated boxes share unit geometry; set dimensions with scale.
        "material": { "type": "Material", "properties": { "materialId": "stone" } }
      }
    }]
  }
}
```

Common asset entries under `components`:

```jsonc
// Model assets retain their embedded materials; a sibling Material won't override them.
"model": { "type": "Model", "properties": { "filename": "/models/tree.glb" } }

// For primitive meshes: texture is the color map, normalMapTexture the normal map.
"surface": { "type": "Material", "properties": { "texture": "/textures/stone.jpg" } }

// Compose another scene document. Relative asset URLs resolve against basePath.
"building": { "type": "PrefabRef", "properties": { "url": "/prefabs/building.json" } }
```

Use known asset URLs; remote assets need CORS. Validate loading in the browser.
Keep one active Camera and Fog per scene. `CameraFollow` targets a local node;
its offsets are world-space. Edit mode uses editor camera controls.

## Scripted behavior

Add this entry under a node's `components`, through JSON or the editor API:

```json
"spin": {
  "type": "Runtime",
  "properties": {
    "data": { "speed": 1 },
    "setup": "const y = object.rotation.y; return () => { object.rotation.y = y; };",
    "update": "object.rotation.y += data.speed * delta;"
  }
}
```

`setup` returns optional cleanup; `update` receives seconds as `delta`. Both run
while enabled in Play after preparation. Code changes restart setup; data changes
do not. Use `context.data` for current inputs inside setup-created callbacks.

Scripts share `state` and receive `nodeId`, `node`, `object`, `data`, `prefab`,
`events`, and `context`. They execute as trusted page code without React hooks.
Read the Runtime schema for field details; use a source component when hooks are needed.

## Custom components

Use application source for reusable behavior, React hooks, and new visual components.

### Settings only

`name` and `properties` are required. `View` is optional; settings alone do not run behavior.

```ts
import { registerComponent, type Component } from 'react-three-game/viewer';

const Health: Component<{ max: number }> = {
  name: 'Health',
  properties: { max: { default: 100 } },
};
registerComponent(Health);
```

### Settings with behavior

Add a `View` to use React hooks and the live node object. Return `children` to preserve composition.

```tsx
import { useFrame } from '@react-three/fiber';
import { registerComponent, useNode, useGameObject,
  type Component, type ComponentViewProps } from 'react-three-game/viewer';

type SpinProps = { speed: number };
function SpinView({ properties, enabled, children }: ComponentViewProps<SpinProps>) {
  const object = useGameObject();
  const { editMode, preparing } = useNode();
  // Animate the live object; document writes are for authored edits.
  useFrame((_, delta) => {
    if (enabled && !editMode && !preparing && object.transform) {
      object.transform.rotation.y += properties.speed * delta;
    }
  });
  return <>{children}</>;
}
const Spin: Component<SpinProps> = {
  name: 'Spin', View: SpinView,
  // Defaults feed both the view and the generated inspector.
  properties: { speed: { default: 1, step: 0.1 } },
};
registerComponent(Spin);
```

### Attach to a node

Import the registration module before mounting the editor or viewer, then add an
entry under the node's `components`. The instance key is local to the node;
`type` matches the registered name.

```json
"spin": { "type": "Spin", "properties": { "speed": 2 } }
```

### Property and rendering contracts

Fields inside a component definition:

```ts
properties: {
  speed: { default: 1 },                       // Numbers infer their type.
  label: { type: 'string', default: 'Box' },    // Other values specify a type.
  mode: {
    type: 'select', default: 'walk',
    options: [{ value: 'walk', label: 'Walk' }, { value: 'run', label: 'Run' }],
  },
},
// View receives resolved defaults; don't duplicate them in the view.
// Ordinary fields get an inspector automatically.

// Omit slot for ordinary behavior. When supplying a render-graph part:
slot: 'geometry', // Or 'object' / 'material'; each slot is exclusive on its node.
// The View must implement attachment and preserve children where appropriate.
```

For a custom inspector, in an editor-only module:

```ts
import { registerComponentEditor } from 'react-three-game/editor';

registerComponentEditor(Spin, SpinInspector); // App-defined component and inspector.
// Keep this module out of runtime imports.
```

## State and resources

| Need | Use |
| --- | --- |
| Current node object | `useGameObject()` |
| Node/edit state | `useNode()` |
| Document and local objects | `usePrefab()` |
| Shared scene/mode | `useScene()` |
| Serializable edit | Prefab mutations |
| View animation | Refs/live Three objects in R3F `useFrame` |
| Gameplay simulation | Host-owned state and tick loop |
| URL-backed chunk | `PrefabInstance` |

### Load and activate chunks

Inside the canvas:

```tsx
import { PrefabInstance } from 'react-three-game/viewer';

// Prepare without activating; set active to true when wanted.
<PrefabInstance id="courtyard" url="/prefabs/courtyard.json" active={false} />
// onStatus: preparation status/errors. onActivate: gameplay is active.
// Unmount to release. Use static only when the chunk will remain immutable.
```

### Modify geometry

```ts
const Stretch: Component<{ amount: number }> = {
  name: 'Stretch',
  properties: { amount: { default: 1 } },
  modifyGeometry: (source, { amount }) => source.clone().scale(1, amount, 1),
};
registerComponent(Stretch);
```

Return a new owned geometry; leave the source untouched. Modifiers run in component
order. The optional third argument supplies node scale as `context.scale`.

### Declare assets and share materials

In an asset-backed component definition, declare each resource used by its view:

```ts
dependencies: ({ filename }) => [{ kind: 'model', path: filename }],
```

Kinds are `model`, `texture`, `sound`, and `prefab`; paths respect `basePath`.
Custom shared materials use `useSharedMaterialResource` and remain immutable.

## Focused references

Read only the reference needed for the task:

- [Runtime communication](rules/RUNTIME_INTEGRATIONS.md)
- [Physics](rules/ADVANCED_PHYSICS.md)
- [Lighting](rules/LIGHTING.md)
- [Performance](rules/PERFORMANCE.md)
