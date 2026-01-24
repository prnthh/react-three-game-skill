---
name: react-three-game
description: react-three-game, a JSON-first 3D game engine built on React Three Fiber, WebGPU, and Rapier Physics.
---

# react-three-game

Instructions for the agent to follow when this skill is activated.

## When to use

generate 3D scenes, games and physics simulations in React.

## Agent Workflow: JSON → GLB

Agents can programmatically generate 3D assets:

1. Create a JSON prefab following the GameObject schema
2. Load it in `PrefabEditor` to render the Three.js scene
3. Export the scene to GLB format using `exportGLB` or `exportGLBData`

```tsx
import { useRef, useEffect } from 'react';
import { PrefabEditor, exportGLBData } from 'react-three-game';
import type { PrefabEditorRef } from 'react-three-game';

const jsonPrefab = {
  root: {
    id: "scene",
    children: [
      {
        id: "cube",
        components: {
          transform: { type: "Transform", properties: { position: [0, 0, 0] } },
          geometry: { type: "Geometry", properties: { geometryType: "box", args: [1, 1, 1] } },
          material: { type: "Material", properties: { color: "#ff0000" } }
        }
      }
    ]
  }
};

function AgentExporter() {
  const editorRef = useRef<PrefabEditorRef>(null);

  useEffect(() => {
    const timer = setTimeout(async () => {
      const sceneRoot = editorRef.current?.rootRef.current?.root;
      if (!sceneRoot) return;
      
      const glbData = await exportGLBData(sceneRoot);
      // glbData is an ArrayBuffer ready for upload/storage
    }, 1000); // Wait for scene to render
    
    return () => clearTimeout(timer);
  }, []);

  return <PrefabEditor ref={editorRef} initialPrefab={jsonPrefab} />;
}
```

## Core Concepts

### Asset Paths and Public Directory

**All asset paths are relative to `/public`** and omit the `/public` prefix:

```json
{
  "texture": "/textures/floor.png",
  "model": "/models/car.glb",
  "font": "/fonts/font.ttf"
}
```

Path `"/any/path/file.ext"` refers to `/public/any/path/file.ext`.
### GameObject Structure

Every game object follows this schema:

```typescript
interface GameObject {
  id: string;
  disabled?: boolean;
  hidden?: boolean;
  components?: Record<string, { type: string; properties: any }>;
  children?: GameObject[];
}
```

### Prefab JSON Format

Scenes are defined as JSON prefabs with a root node containing children:

```json
{
  "root": {
    "id": "scene",
    "children": [
      {
        "id": "my-object",
        "components": {
          "transform": { "type": "Transform", "properties": { "position": [0, 0, 0] } },
          "geometry": { "type": "Geometry", "properties": { "geometryType": "box" } },
          "material": { "type": "Material", "properties": { "color": "#ff0000" } }
        }
      }
    ]
  }
}
```

## Built-in Components

| Component | Type | Key Properties |
|-----------|------|----------------|
| Transform | `Transform` | `position: [x,y,z]`, `rotation: [x,y,z]` (radians), `scale: [x,y,z]` |
| Geometry | `Geometry` | `geometryType`: box/sphere/plane/cylinder, `args`: dimension array |
| Material | `Material` | `color`, `texture?`, `metalness?`, `roughness?`, `repeat?`, `repeatCount?` |
| Physics | `Physics` | `type`: "dynamic" or "fixed" |
| Model | `Model` | `filename` (GLB/FBX path), `instanced?` for GPU batching |
| SpotLight | `SpotLight` | `color`, `intensity`, `angle`, `penumbra`, `distance?`, `castShadow?` |
| DirectionalLight | `DirectionalLight` | `color`, `intensity`, `castShadow?`, `targetOffset?: [x,y,z]` |
| AmbientLight | `AmbientLight` | `color`, `intensity` |
| Text | `Text` | `text`, `font`, `size`, `depth`, `width`, `align`, `color` |

### Text Component

Requires `hb.wasm` and a font file (TTF/WOFF) in `/public/fonts/`:
- hb.wasm: https://github.com/prnthh/react-three-game/raw/refs/heads/main/docs/public/fonts/hb.wasm
- Sample font: https://github.com/prnthh/react-three-game/raw/refs/heads/main/docs/public/fonts/NotoSans-Regular.ttf

Font property: `"font": "/fonts/NotoSans-Regular.ttf"`

### Geometry Args by Type

| geometryType | args array |
|--------------|------------|
| `box` | `[width, height, depth]` |
| `sphere` | `[radius, widthSegments, heightSegments]` |
| `plane` | `[width, height]` |
| `cylinder` | `[radiusTop, radiusBottom, height, radialSegments]` |

### Material Textures

```json
{
  "material": {
    "type": "Material",
    "properties": {
      "color": "white",
      "texture": "/textures/floor.png",
      "repeat": true,
      "repeatCount": [4, 4]
    }
  }
}
```

### Rotations

Use radians: `1.57` = 90°, `3.14` = 180°, `-1.57` = -90°

## Common Patterns

### Usage Modes

**GameCanvas + PrefabRoot**: Production gameplay. Requires explicit `<Physics>` wrapper. Physics always active. Can compose with other R3F components. For headless mode, use `<PrefabRoot>` without GameCanvas.

```jsx
import { Physics } from '@react-three/rapier';
import { GameCanvas, PrefabRoot } from 'react-three-game';

<GameCanvas>
  <Physics>
    <PrefabRoot data={prefabData} />
  </Physics>
</GameCanvas>
```

**PrefabEditor**: Level editors, scene authoring, prototyping. Includes canvas, physics, UI. Physics activates in play mode only.

```jsx
import { PrefabEditor } from 'react-three-game';

<PrefabEditor initialPrefab={prefabData} />
```

### Tree Utilities

```typescript
import { updateNodeById, findNode, deleteNode, cloneNode, exportGLBData } from 'react-three-game';

const updated = updateNodeById(root, nodeId, node => ({ ...node, disabled: true }));
const node = findNode(root, nodeId);
const afterDelete = deleteNode(root, nodeId);
const cloned = cloneNode(node);
const glbData = await exportGLBData(sceneRoot);
```

## Level Patterns

### Floor

```json
{
  "id": "floor",
  "components": {
    "transform": { "type": "Transform", "properties": { "position": [0, -0.5, 0] } },
    "geometry": { "type": "Geometry", "properties": { "geometryType": "box", "args": [40, 1, 40] } },
    "material": { "type": "Material", "properties": { "texture": "/textures/floor.png", "repeat": true, "repeatCount": [20, 20] } },
    "physics": { "type": "Physics", "properties": { "type": "fixed" } }
  }
}
```

### Platform

```json
{
  "id": "platform",
  "components": {
    "transform": { "type": "Transform", "properties": { "position": [-8, 2, -5] } },
    "geometry": { "type": "Geometry", "properties": { "geometryType": "box", "args": [6, 0.5, 4] } },
    "physics": { "type": "Physics", "properties": { "type": "fixed" } }
  }
}
```

### Ramp

```json
{
  "id": "ramp",
  "components": {
    "transform": { "type": "Transform", "properties": { "position": [-12, 1, -5], "rotation": [0, 0, 0.3] } },
    "geometry": { "type": "Geometry", "properties": { "geometryType": "box", "args": [5, 0.3, 3] } },
    "physics": { "type": "Physics", "properties": { "type": "fixed" } }
  }
}
```

### Wall Pattern
### Wall

```json
{
  "id": "wall",
  "components": {
    "transform": { "type": "Transform", "properties": { "position": [0, 3, -20] } },
    "geometry": { "type": "Geometry", "properties": { "geometryType": "box", "args": [40, 7, 1] } },
    "physics": { "type": "Physics", "properties": { "type": "fixed" } }
  }
}
```

### Lighting

```json
[
  { "id": "spot", "components": { "transform": { "properties": { "position": [10, 15, 10] } }, "spotlight": { "type": "SpotLight", "properties": { "intensity": 200, "angle": 0.8, "castShadow": true } } } },
  { "id": "ambient", "components": { "ambientlight": { "type": "AmbientLight", "properties": { "intensity": 0.4 } } } }
]
```

### Text

```json
{
  "id": "text",
  "components": {
    "transform": { "type": "Transform", "properties": { "position": [0, 3, 0] } },
    "text": { "type": "Text", "properties": { "text": "Welcome", "font": "/fonts/font.ttf", "size": 1, "depth": 0.1 } }
  }
}
```

### Model

```json
{
  "id": "model",
  "components": {
    "transform": { "type": "Transform", "properties": { "position": [0, 0, 0], "scale": [1.5, 1.5, 1.5] } },
    "model": { "type": "Model", "properties": { "filename": "/models/tree.glb" } }
  }
}
```

## Editor

### Basic Usage

```jsx
import { PrefabEditor } from 'react-three-game';

<PrefabEditor initialPrefab={sceneData} onPrefabChange={setSceneData} />
```

Keyboard shortcuts: **T** (Translate), **R** (Rotate), **S** (Scale)

### Programmatic Updates

```jsx
import { useRef } from 'react';
import { PrefabEditor, updateNodeById } from 'react-three-game';
import type { PrefabEditorRef } from 'react-three-game';

function Scene() {
  const editorRef = useRef<PrefabEditorRef>(null);

  const moveBall = () => {
    const prefab = editorRef.current!.prefab;
    const newRoot = updateNodeById(prefab.root, "ball", node => ({
      ...node,
      components: {
        ...node.components,
        transform: {
          ...node.components!.transform!,
          properties: { ...node.components!.transform!.properties, position: [5, 0, 0] }
        }
      }
    }));
    editorRef.current!.setPrefab({ ...prefab, root: newRoot });
  };

  return <PrefabEditor ref={editorRef} initialPrefab={sceneData} />;
}
```

**PrefabEditorRef**: `prefab`, `setPrefab()`, `screenshot()`, `exportGLB()`, `rootRef`

### GLB Export

```tsx
import { exportGLBData } from 'react-three-game';

const glbData = await exportGLBData(editorRef.current!.rootRef.current!.root);
```

### Runtime Animation

```tsx
import { useRef } from "react";
import { useFrame } from "@react-three/fiber";
import { PrefabEditor, updateNodeById } from "react-three-game";

function Animator({ editorRef }) {
  useFrame(() => {
    const prefab = editorRef.current!.prefab;
    const newRoot = updateNodeById(prefab.root, "ball", node => ({
      ...node,
      components: {
        ...node.components,
        transform: {
          ...node.components!.transform!,
          properties: { ...node.components!.transform!.properties, position: [x, y, z] }
        }
      }
    }));
    editorRef.current!.setPrefab({ ...prefab, root: newRoot });
  });
  return null;
}

function Scene() {
  const editorRef = useRef(null);
  return (
    <PrefabEditor ref={editorRef} initialPrefab={data}>
      <Animator editorRef={editorRef} />
    </PrefabEditor>
  );
}
```

### Custom Component

```tsx
import { Component, registerComponent, FieldRenderer } from 'react-three-game';

const MyComponent: Component = {
  name: 'MyComponent',
  Editor: ({ component, onUpdate }) => (
    <FieldRenderer fields={[{ name: 'speed', type: 'number', step: 0.1 }]} values={component.properties} onChange={onUpdate} />
  ),
  View: ({ properties, children }) => <group>{children}</group>,
  defaultProperties: { speed: 1 }
};

registerComponent(MyComponent);
```

**Field types**: `vector3`, `number`, `string`, `color`, `boolean`, `select`, `custom`

