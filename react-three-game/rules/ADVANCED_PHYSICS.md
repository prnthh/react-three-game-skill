# Advanced Physics & Patterns

## Physics Type Decision Tree

```
Need physics?
├─ No  → Don't add Physics component
└─ Yes → Does it move?
    ├─ Never moves (walls, floor, static props)
    │   └─ type: "fixed"
    │
    ├─ Moves via forces/gravity (balls, boxes, ragdolls)
    │   └─ type: "dynamic"
    │       ├─ Fast moving? → ccd: true
    │       └─ Heavy? → mass: 10+
    │
    ├─ Scripted animation (moving platforms, doors)
    │   └─ type: "kinematicPosition"
    │       └─ Update transform via updateNodeById
    │
    └─ Velocity-driven (conveyor belts, wind zones)
        └─ type: "kinematicVelocity"
            └─ Set velocity via RigidBody ref
```

**Type descriptions**:
- **fixed**: Immovable, infinite mass (ground, walls, buildings)
- **dynamic**: Affected by forces and gravity (player, projectiles, props)
- **kinematicPosition**: Move via code, push dynamic bodies (elevators, doors)
- **kinematicVelocity**: Set constant velocity, push dynamic bodies (conveyors)

**Performance tip**: Use `fixed` for anything that never moves - it's cheapest.

## Physics Material Properties

Complete reference for `Physics` component properties:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `type` | `'dynamic'` \| `'fixed'` \| `'kinematicPosition'` \| `'kinematicVelocity'` | `'dynamic'` | Body type (see decision tree above) |
| `mass` | `number` | `1` | Body mass (dynamic only) |
| `restitution` | `number` | `0` | Bounciness (0 = no bounce, 1 = perfect bounce) |
| `friction` | `number` | `0.5` | Surface friction (0 = ice, 1+ = sticky) |
| `linearDamping` | `number` | `0` | Velocity decay (0 = none, 1 = full stop) |
| `angularDamping` | `number` | `0` | Rotation decay |
| `gravityScale` | `number` | `1` | Gravity multiplier (0 = floating, 2 = heavy) |
| `lockTranslations` | `boolean` | `false` | Freeze position |
| `lockRotations` | `boolean` | `false` | Freeze rotation |
| `enabledTranslations` | `[bool, bool, bool]` | `[true, true, true]` | Lock per axis (X, Y, Z) |
| `enabledRotations` | `[bool, bool, bool]` | `[true, true, true]` | Lock rotation per axis |
| `ccd` | `boolean` | `false` | Continuous collision detection (fast objects) |
| `sensor` | `boolean` | `false` | Trigger only, no collision response |
| `collisionGroups` | `number` | - | Rapier collision groups bitfield |
| `solverGroups` | `number` | - | Rapier solver groups bitfield |

**Example - Bouncy Ball**:
```json
{
  "physics": {
    "type": "Physics",
    "properties": {
      "type": "dynamic",
      "mass": 0.5,
      "restitution": 0.9,
      "friction": 0.1,
      "linearDamping": 0.05
    }
  }
}
```

**Example - Ice Surface**:
```json
{
  "physics": {
    "type": "Physics",
    "properties": {
      "type": "fixed",
      "friction": 0,
      "restitution": 0.1
    }
  }
}
```

## Force & Impulse Application

Access RigidBody refs via `PrefabRootRef.rigidBodyRefs` to apply physics forces to prefab objects:

```tsx
import { useRef, useEffect } from 'react';
import { PrefabEditor } from 'react-three-game';
import type { PrefabEditorRef } from 'react-three-game';
import type { RapierRigidBody } from '@react-three/rapier';

function ForceApplier({ editorRef }: { editorRef: React.RefObject<PrefabEditorRef> }) {
  useEffect(() => {
    const interval = setInterval(() => {
      const rootRef = editorRef.current?.rootRef.current;
      if (!rootRef) return;
      
      // Access RigidBody ref by node ID
      const rigidBody = rootRef.rigidBodyRefs.get('ball') as RapierRigidBody;
      
      if (rigidBody) {
        // Apply upward impulse
        rigidBody.applyImpulse({ x: 0, y: 5, z: 0 }, true);
        
        // Or apply continuous force
        rigidBody.addForce({ x: 0, y: 10, z: 0 }, true);
        
        // Apply torque
        rigidBody.applyTorqueImpulse({ x: 0, y: 1, z: 0 }, true);
      }
    }, 2000);
    
    return () => clearInterval(interval);
  }, [editorRef]);
  
  return null;
}

function Scene() {
  const editorRef = useRef<PrefabEditorRef>(null);
  
  return (
    <PrefabEditor ref={editorRef} initialPrefab={prefab}>
      <ForceApplier editorRef={editorRef} />
    </PrefabEditor>
  );
}
```

**Alternative: Custom R3F components**

For fully custom physics objects, create R3F components with their own RigidBody refs:

```tsx
import { useRef } from 'react';
import { RigidBody } from '@react-three/rapier';
import type { RapierRigidBody } from '@react-three/rapier';
import { useFrame } from '@react-three/fiber';
import { PrefabEditor } from 'react-three-game';

function PhysicsBall() {
  const rigidBodyRef = useRef<RapierRigidBody>(null);
  
  useFrame(() => {
    if (rigidBodyRef.current) {
      // Apply jump force on interval
      rigidBodyRef.current.applyImpulse({ x: 0, y: 5, z: 0 }, true);
    }
  });
  
  return (
    <RigidBody ref={rigidBodyRef} position={[0, 5, 0]} type="dynamic">
      <mesh castShadow>
        <sphereGeometry args={[0.5, 32, 32]} />
        <meshStandardMaterial color="orange" />
      </mesh>
    </RigidBody>
  );
}

<PrefabEditor initialPrefab={prefab}>
  <PhysicsBall />
</PrefabEditor>
```

**Alternative: Kinematic position updates**

For smooth animated movement without forces, use `kinematicPosition` and update via `updateNodeById`:

```tsx
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import { PrefabEditor, updateNodeById } from 'react-three-game';
import type { PrefabEditorRef } from 'react-three-game';

function KinematicMover({ editorRef }: { editorRef: React.RefObject<PrefabEditorRef> }) {
  useFrame(({ clock }) => {
    const prefab = editorRef.current?.prefab;
    if (!prefab) return;
    
    const y = 2 + Math.sin(clock.elapsedTime * 2) * 3;
    
    const newRoot = updateNodeById(prefab.root, "platform", node => ({
      ...node,
      components: {
        ...node.components,
        transform: {
          ...node.components!.transform!,
          properties: {
            ...node.components!.transform!.properties,
            position: [0, y, 0]
          }
        }
      }
    }));
    
    editorRef.current!.setPrefab({ ...prefab, root: newRoot });
  });
  
  return null;
}
```

**Rapier RigidBody methods**:
- `applyImpulse(vector, wakeUp)` - Instantaneous velocity change
- `addForce(vector, wakeUp)` - Continuous force application
- `applyTorqueImpulse(vector, wakeUp)` - Rotational impulse
- `addTorque(vector, wakeUp)` - Continuous torque
- `setLinvel(vector, wakeUp)` - Set linear velocity directly
- `setAngvel(vector, wakeUp)` - Set angular velocity directly

## Tilted Surfaces & Containment

**⚠️ Tilted walls don't contain objects** - physics objects slide off angled surfaces.

### ❌ Wrong Approach
```json
{
  "id": "tilted-wall",
  "components": {
    "transform": { "type": "Transform", "properties": { "rotation": [0, 0, 0.3] } },
    "geometry": { "type": "Geometry", "properties": { "geometryType": "box", "args": [10, 5, 1] } },
    "physics": { "type": "Physics", "properties": { "type": "fixed" } }
  }
}
```
Objects will **slide off** the tilted surface.

### ✅ Correct Pattern - Perpendicular Walls
```json
{
  "id": "container",
  "children": [
    {
      "id": "floor",
      "components": {
        "transform": { "type": "Transform", "properties": { "position": [0, 0, 0] } },
        "geometry": { "type": "Geometry", "properties": { "geometryType": "box", "args": [20, 1, 20] } },
        "physics": { "type": "Physics", "properties": { "type": "fixed" } }
      }
    },
    {
      "id": "wall-north",
      "components": {
        "transform": { "type": "Transform", "properties": { "position": [0, 2.5, -10] } },
        "geometry": { "type": "Geometry", "properties": { "geometryType": "box", "args": [20, 5, 1] } },
        "physics": { "type": "Physics", "properties": { "type": "fixed" } }
      }
    },
    {
      "id": "wall-south",
      "components": {
        "transform": { "type": "Transform", "properties": { "position": [0, 2.5, 10] } },
        "geometry": { "type": "Geometry", "properties": { "geometryType": "box", "args": [20, 5, 1] } },
        "physics": { "type": "Physics", "properties": { "type": "fixed" } }
      }
    },
    {
      "id": "wall-east",
      "components": {
        "transform": { "type": "Transform", "properties": { "position": [10, 2.5, 0] } },
        "geometry": { "type": "Geometry", "properties": { "geometryType": "box", "args": [1, 5, 20] } },
        "physics": { "type": "Physics", "properties": { "type": "fixed" } }
      }
    },
    {
      "id": "wall-west",
      "components": {
        "transform": { "type": "Transform", "properties": { "position": [-10, 2.5, 0] } },
        "geometry": { "type": "Geometry", "properties": { "geometryType": "box", "args": [1, 5, 20] } },
        "physics": { "type": "Physics", "properties": { "type": "fixed" } }
      }
    }
  ]
}
```

**Key principle**: Walls must be **perpendicular to gravity** to contain dynamic objects.

## Instanced Physics

When using `"instanced": true` on models, physics behaves differently than standard objects. **All instances of the same model share a single `InstancedRigidBodies` component** for optimal GPU performance.

### Standard vs Instanced Physics

| Aspect | Standard Physics | Instanced Physics |
|--------|------------------|-------------------|
| RigidBody Component | Individual `<RigidBody>` per object | Single `<InstancedRigidBodies>` for all instances |
| Ref Access | `rigidBodyRefs.get(nodeId)` returns single RigidBody | Not accessible via `rigidBodyRefs` |
| Force Application | Direct per-object | Must access via InstancedRigidBodies ref |
| Collider Type | `hull` (dynamic) or `trimesh` (fixed) | Same, auto-selected |
| Performance | One draw call per object | One draw call for all instances |

### Defining Instanced Objects

Set `"instanced": true` in the model component. **All instances of the same model+physics type are automatically batched**:

```json
{
  "id": "tree1",
  "components": {
    "transform": { "type": "Transform", "properties": { "position": [0, 0, 0] } },
    "model": { "type": "Model", "properties": { "filename": "models/tree.glb", "instanced": true } },
    "physics": { "type": "Physics", "properties": { "type": "fixed" } }
  }
}
```

Add multiple instances - they'll be automatically batched:

```json
{
  "id": "tree2",
  "components": {
    "transform": { "type": "Transform", "properties": { "position": [5, 0, 3] } },
    "model": { "type": "Model", "properties": { "filename": "models/tree.glb", "instanced": true } },
    "physics": { "type": "Physics", "properties": { "type": "fixed" } }
  }
}
```

### Force Application on Instanced Objects

**Instanced physics bodies are not individually accessible.** For objects requiring force/impulse control, use non-instanced physics (`"instanced": false` or omit the property).

### When to Use Instanced Physics

✅ **Good for:**
- Many copies of the same static object (trees, rocks, buildings)
- Large scenes with 100+ similar objects
- Fixed physics bodies that never move
- Background props and decorations

❌ **Avoid for:**
- Objects requiring individual force/impulse control
- Dynamic objects with unique behaviors
- Objects that need to be individually removed/spawned
- Fewer than ~20 instances (overhead not worth it)

### Performance Notes

- **Batching**: All instances with the same `filename` and `physics.type` are rendered in a single draw call
- **Scale handling**: Visual scale is applied per-instance, but collider scale may differ
- **Transform updates**: Use `updateNodeById` to move instances (triggers re-sync)
- **Memory**: One set of GPU buffers shared across all instances
