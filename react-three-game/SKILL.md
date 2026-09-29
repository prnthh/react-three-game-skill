---
name: react-three-game
description: Build and edit React Three Game scenes using WebGPU rendering, React Three Fiber composition, prefab components and the editor API for agents.
---

# React Three Game

Use the existing component schemas and demos as the API reference. Prefer composing built-ins to creating new abstractions.

React Three Game owns rendering and scene authoring. The host application owns gameplay state, world ticks, simulation scheduling, input, networking and rules. `SceneRuntime` supplies resource/context providers, not a game loop. Ordinary R3F JSX can compose alongside prefab content; not every React component needs a serialized definition.

## Start in a running editor

For live scene authoring, start in the editor frame's JavaScript context:

```js
window.reactThreeGame.help();
window.reactThreeGame.listEditors();
const api = window.reactThreeGame.editors[chosenId]; // Choose an ID from that list.
api.help();
api.describeComponents({ names: ['Geometry', 'Mesh', 'Material', 'Model'] });
api.analyzeScene();
```

Prefer targeted `findNodes()` / `getNodes()` reads and small batches over rewriting the whole document. Read instance keys before patching components. Pass the read's `revision` as `expectedRevision` to `validateBatch()` and `applyBatch()`; a mismatch means reread and replan. Validation is read-only; applying is one undo step in Edit mode. Both return authoring advisories, which are document heuristics, not measured FPS.

Use `focusNode()` and `captureView()` to check the result. `exportScene()` returns data without saving; `saveScene()` needs a host adapter (`getSceneInfo().canSave`). The registry may be absent before mount or when tools are disabled. This is an ordinary page API, not an MCP server.

Copy the grouped geometry, texture and model example from [Editor API for agents](https://prnth.com/react-three-game/editor-api-for-agents.md). In this library checkout, the canonical source is `docs/public/editor-api-for-agents.md`; consult it when the deployed guide lags local changes.

Edit/Play is a mode signal, not a game-session snapshot. Live motion is not serialized or undoable; hosts own pause/reset behavior.

## Compose or extend an application

1. Define a prefab with a root node, stable node IDs, and sparse component properties.
2. Register custom components before mounting the scene.
3. Render with `GameCanvas` and `PrefabRoot` from `react-three-game/viewer`.
4. Use `PrefabEditor` from `react-three-game/editor` for authoring.
5. Verify the changed behavior in the browser and run relevant checks.

```tsx
import { GameCanvas, PrefabRoot } from 'react-three-game/viewer';
import type { Prefab } from 'react-three-game/core';
import scene from './scene.json';

<GameCanvas><PrefabRoot data={scene as Prefab} /></GameCanvas>
```

## Prefab conventions

```json
{
  "materials": { "stone": { "color": "#999999" } },
  "root": {
    "id": "world",
    "children": [{
      "id": "box",
      "components": {
        "transform": { "type": "Transform", "properties": { "position": [0, 1, 0] } },
        "mesh": { "type": "Mesh", "properties": {} },
        "geometry": { "type": "Geometry", "properties": {} },
        "material": { "type": "Material", "properties": { "materialId": "stone" } }
      }
    }]
  }
}
```

Organize assemblies under named parents with stable IDs. Repeated boxes should share unit geometry `[1,1,1]`, with dimensions in `Transform.scale` and matching material settings. Built-in materials automatically share GPU resources across different IDs; reuse `Material.materialId` only when edits should be linked. Scaling existing nodes also scales children and colliders; preserve their meaning.

Load meshes through `Model.filename` and textures through material `texture` / `normalMapTexture`, using known URLs. Imported models keep their embedded materials. URLs respect `basePath` and browser CORS; batch validation does not verify asynchronous asset loads.

Use local transforms and radians. Defaults come from schemas. Material IDs and node IDs are local to a prefab. `PrefabRef` with a `url` composes another document. Asset paths respect `basePath`.

Use one active `Camera` and one `Fog` node per scene. `CameraFollow` targets a node in the same prefab; its offsets are world-space. Edit mode uses editor camera controls.

## Custom components

One file contains the schema and view. Ordinary editor fields are generated automatically.

```tsx
import { useFrame } from '@react-three/fiber';
import { registerComponent, useNode, useGameObject,
  type Component, type ComponentViewProps } from 'react-three-game/viewer';

type SpinProps = { speed: number };
function SpinView({ properties, children }: ComponentViewProps<SpinProps>) {
  const object = useGameObject();
  const { editMode } = useNode();
  useFrame((_, delta) => {
    if (!editMode && object.transform) object.transform.rotation.y += properties.speed * delta;
  });
  return <>{children}</>;
}
const Spin: Component<SpinProps> = {
  name: 'Spin', View: SpinView,
  properties: { speed: { default: 1, step: 0.1 } },
};
registerComponent(Spin);
```

Views receive resolved properties. Do not duplicate defaults or write a custom inspector for ordinary fields. Numeric fields infer their type; other fields specify `type`. Select fields supply `options` with `value` and `label`.

Most behaviors need no slot. Use `slot: 'object'`, `'geometry'`, or `'material'` when providing those parts of a node. Slots are exclusive; the view implements actual R3F attachment and renders children.

Custom inspector UI belongs in an editor-only module registered with `registerComponentEditor(component, Inspector)`. Runtime modules must not import editor UI.

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

`PrefabInstance` prepares before activation. Use a stable `id`, `url`, `onStatus` for errors, and `onActivate` for activation. `active={false}` stages content. Unmount to release it. `GameCanvas` shares the runtime across sibling chunks automatically. Use `static` only for immutable chunks.

Asset-backed components declare `dependencies(properties)` returning `{ kind: 'model' | 'texture' | 'sound' | 'prefab', path }`. Custom shared materials use `useSharedMaterialResource`; treat shared materials as immutable.

## Focused references

Read only the reference needed for the task:

- [Runtime communication](rules/RUNTIME_INTEGRATIONS.md)
- [Physics](rules/ADVANCED_PHYSICS.md)
- [Lighting](rules/LIGHTING.md)
- [Performance](rules/PERFORMANCE.md)
