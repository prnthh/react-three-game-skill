---
name: react-three-game
description: react-three-game, a JSON-first 3D game engine built on React Three Fiber, WebGPU, and Rapier Physics.
---

# react-three-game

Use this skill when building scenes or tools with `react-three-game`.

## Framework Shape

`react-three-game` is a wrapper over normal Three.js and React Three Fiber composition.

Think in layers:

1. Three.js and R3F are still the runtime.
2. `PrefabRoot` renders prefab JSON into normal R3F structure.
3. `PrefabEditor` wraps `PrefabRoot` with editor UI, selection, transform controls, and play/edit mode.

The framework should help with scene structure and tooling, then get out of the way.

## Choose The Lowest Layer

Use the lowest layer that solves the task:

1. plain R3F / Three.js for runtime behavior or visual composition
2. `PrefabRoot` for JSON-authored scene structure
3. `PrefabEditor` for authoring UI and editor tooling

Do not force runtime logic into prefab abstractions if a normal R3F component is clearer.

## Main Pieces

### `PrefabRoot`

Use `PrefabRoot` when the scene comes from prefab JSON but still lives inside a normal R3F app.

```tsx
<GameCanvas>
  <Physics>
    <PrefabRoot data={prefab} />
    <CustomController />
  </Physics>
</GameCanvas>
```

### `PrefabEditor`

Use `PrefabEditor` when the user needs authoring features.

- built-in canvas
- hierarchy and inspector UI
- selection and transform gizmos
- play/edit mode

It still accepts normal R3F children.

```tsx
<PrefabEditor initialPrefab={prefab}>
  <CustomController />
</PrefabEditor>
```

## State Split

Keep these separate:

- editor state: transform mode, selection, play/edit mode, focus, export actions
- scene state: prefab tree content, node transforms, materials, disabled flags, children

Use the matching tool:

- `useEditorContext()` for editor state
- `updateNode(...)` / `updateNodeById(...)` for scene state

## Quick Component Map

| Need | Component / Tool | Key props / note |
|---|---|---|
| local transform | `Transform` | `position`, `rotation`, `scale` |
| primitive mesh | `Geometry` + `Material` | `geometryType`, `args`, `color`, `texture` |
| imported asset | `Model` | `filename`, `instanced?` |
| rigid body | `Physics` | `type`, `mass`, `restitution`, `friction`, `linearVelocity?`, `angularVelocity?`, `sensor?` |
| authored camera | `Camera` | camera properties authored in prefab data |
| authored lights | `SpotLight`, `DirectionalLight`, `AmbientLight` | `color`, `intensity`, light-specific props |
| text mesh | `Text` | `text`, `font`, `size`, `depth` |
| sky / scene backdrop | `Environment` | wrapper-style scene backdrop / lighting |
| click interaction | `Click` | emits prefab click events for controllers |
| editor-only control | `useEditorContext()` | editor state only |
| prefab tree mutation | `updateNode(...)`, `updateNodeById(...)` | scene state only |

## Composition Patterns

### Editor State Bridge

For custom editor controls, render a tiny helper component inside `PrefabEditor` and read editor state from context.

```tsx
function EditorStateBridge({ onReady }) {
  const editorState = useEditorContext();

  useEffect(() => {
    onReady(editorState);
  }, [editorState, onReady]);

  return null;
}
```

### Scene State Updates

For scene changes, update the prefab tree immutably.

```tsx
const root = updateNodeById(prefab.root, 'node-id', node => ({
  ...node,
  disabled: true,
}));
```

Then push the new root back through `setPrefab` or `replacePrefab`.

### Hybrid Pattern

Use prefab JSON for static structure and normal R3F children for dynamic behavior.

Good fits:

- controllers
- procedural animation
- postprocessing
- debug helpers
- runtime-only scene logic

## Cookbook

| Goal | Use | Key props / note |
|---|---|---|
| authored scene + runtime logic | `PrefabRoot` or `PrefabEditor` + normal R3F children | prefab for structure, children for behavior |
| custom editor shell | `PrefabEditor showUI={false}` + `useEditorContext()` | bridge editor state inside a child component |
| embedded viewer / import tool | `replacePrefab`, `addModel`, `addTexture`, `exportGLBData` | host app owns UI |
| runtime scene change | `updateNode(...)` / `updateNodeById(...)` + `setPrefab(...)` | mutate one node, then apply new prefab |

## Embedded Tools

For custom tools or viewers:

- use `showUI={false}` if the host app provides its own UI
- use `replacePrefab(prefab)` to load a new scene
- use `setPrefab(prefab)` to update current scene state
- use `addModel(...)` / `addTexture(...)` for runtime asset injection
- use `clearSelection()` when the host shell needs to reset editor focus state
- use `screenshot()` or `exportGLB(...)` when the host shell needs an immediate capture/export action
- use `exportGLBData()` for raw bytes

Use `rootRef` only when lower-level scene or rigid-body access is actually needed.

## Physics

Physics is still Rapier plus R3F composition.

- use prefab `Physics` components for authored rigid bodies
- use `linearVelocity` / `angularVelocity` on prefab `Physics` when the initial motion can be declared in scene data
- use custom R3F components when direct composition is simpler
- keep detailed collision and sensor patterns in `rules/ADVANCED_PHYSICS.md`

## Important Conventions

- asset paths are relative to `/public`
- transforms are local to parent nodes
- `disabled` is the main visibility toggle in prefab data
- component keys are lowercase in JSON
- component `type` values are TitleCase

## What To Avoid

- Do not treat the framework like a sealed engine separate from R3F.
- Do not overuse `PrefabEditor` when `PrefabRoot` or plain R3F is enough.
- Do not mix editor state and scene state into one abstraction.
- Do not bypass simple React composition when it already solves the problem.

