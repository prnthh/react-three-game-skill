---
name: react-three-game
description: react-three-game, a JSON-first 3D game engine built on React Three Fiber, WebGPU, and Rapier Physics.
---

# react-three-game

Use this skill when building scenes or tools with `react-three-game`.

Use it to keep generated code and explanations aligned with the current scene/entity API.

## Working Model

Treat `react-three-game` as an authored scene layer on top of React Three Fiber, not as a separate engine.

- Use plain R3F / Three.js for behavior, controllers, and procedural logic.
- Use `PrefabRoot` for rendering authored scene JSON.
- Use `PrefabEditor` for authoring UI, transform gizmos, and play/edit mode.

Prefer the lowest layer that solves the request.

## Terminology

Use scene/entity language in explanations and generated code.

- `PrefabEditor`: editor shell
- `PrefabRoot`: pure renderer
- `Scene`: imperative scene handle from the editor ref
- `Entity`: a node handle inside the scene
- `EntityComponent`: handle returned by `entity.getComponent(...)`

The serialized document type is `Prefab`, and each node is a `GameObject`.

## Default API Choices

When generating code or suggesting changes, prefer these patterns in order:

1. `entity.getComponent(...).set(path, value)` for targeted property edits
2. `component.update(...)` when next state depends on previous component state
3. `entity.update(...)` or `scene.update(...)` for whole-entity changes like `disabled`, `locked`, or multi-component swaps
4. `scene.add(node, options?)` for authored entity creation
5. `editor.load(...)` and `editor.save()` for whole-scene replacement and persistence

Prefer scene handles over rebuilding and replacing the whole scene tree.

## State Split

Keep editor state and scene state separate.

- `useEditorContext()` is for editor concerns like transform mode, selection, focus, and edit/play state.
- `editor.scene`, `entity`, and `entity.getComponent(...)` are for scene content changes.

Route scene content edits through scene handles, not editor-context helpers.

## Export Surface Guidance

Prefer examples that use the actual root exports.

Common root exports worth suggesting:

- `GameCanvas`
- `PrefabRoot`
- `PrefabEditor`
- `registerComponent`
- `useEditorContext`
- `ground`
- `loadFiles`, `loadModel`, `loadTexture`
- `soundManager`

Useful type exports:

- `PrefabEditorRef`, `PrefabEditorProps`
- `PrefabRootRef`, `PrefabRootProps`
- `Scene`, `Entity`, `EntityComponent`
- `FieldDefinition`, `Component`

Stay within the current root export surface.

## Scene Data Rules

- transforms are local to the parent
- `disabled` is the visibility toggle
- component keys are lowercase in JSON
- component `type` values are TitleCase
- use `crypto.randomUUID()` for new entity ids
- asset paths are relative to `/public`

## Events

Keep event naming simple.

- the component that causes the action should define the event name
- components that react to the action should only listen for that event name
- use built-in defaults like `click`, `sensor:enter`, and `collision:enter` unless you need something more specific
- use a custom name only when one action should drive multiple systems, like sound, spawning, score, or UI

Rule of thumb:

- `Click` and `Physics` emit events
- `Sound` and controller components listen to events

Good pattern:

```json
{
	"click": { "type": "Click", "properties": { "eventName": "cannon:fire" } },
	"sound": { "type": "Sound", "properties": { "path": "/sound/explode.mp3", "eventName": "cannon:fire" } }
}
```

In that example, the click component defines the meaning of the action, and anything else can subscribe to it.

That lets one click trigger multiple reactions without hard-wiring those reactions together.

## Guidance

- Prefer scene/entity language consistently in both prose and code.
- Keep runtime behavior in normal R3F components when that is clearer than pushing it into scene JSON.
- Reach for `PrefabEditor` only when the request actually needs authoring UI or editor interaction.
- Use the current scene-handle APIs instead of whole-scene replacement for targeted mutations.


