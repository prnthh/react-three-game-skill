# Lighting and shadows

Authored lights serialize in prefab JSON and expose their properties in the editor. Edit mode renders light helpers and selected shadow-camera volumes.

## Light table

| Component | Main properties |
|---|---|
| `AmbientLight` | `color`, `intensity` |
| `DirectionalLight` | `color`, `intensity`, `targetOffset`, shadow camera bounds |
| `SpotLight` | `color`, `intensity`, `distance`, `angle`, `penumbra`, `targetOffset`, `map` |
| `PointLight` | `color`, `intensity`, `distance`, `decay` |
| `Environment` | `intensity`, `resolution` |

Shadow-capable lights share these properties:

| Property | Purpose |
|---|---|
| `castShadow` | Enables the shadow pass |
| `shadowMapSize` | Shadow texture resolution |
| `shadowBias` | Depth offset |
| `shadowNormalBias` | Normal-space offset |
| `shadowAutoUpdate` | Continuous or requested refresh |
| `shadowCameraNear`, `shadowCameraFar` | Shadow depth range |

## Authored sun

```json
{
  "id": "sun",
  "components": {
    "transform": {
      "type": "Transform",
      "properties": { "position": [30, 60, 20] }
    },
    "light": {
      "type": "DirectionalLight",
      "properties": {
        "color": "#fff2d0",
        "intensity": 2,
        "castShadow": true,
        "shadowMapSize": 1024,
        "shadowAutoUpdate": false,
        "shadowCameraNear": 0.5,
        "shadowCameraFar": 140,
        "shadowCameraTop": 45,
        "shadowCameraBottom": -45,
        "shadowCameraLeft": -45,
        "shadowCameraRight": 45,
        "targetOffset": [0, -20, -20]
      }
    }
  }
}
```

## Requested shadow refresh

```tsx
import type { PrefabApi } from 'react-three-game/viewer';
import type { DirectionalLight } from 'three';

function refreshSunShadow(prefab: PrefabApi) {
  const light = prefab
    .getObject('sun')
    ?.getObjectByProperty('isDirectionalLight', true) as DirectionalLight | undefined;

  if (light?.isDirectionalLight) {
    light.shadow.autoUpdate = false;
    light.shadow.needsUpdate = true;
  }
}
```

## Scene patterns

| Scene | Lighting shape |
|---|---|
| Outdoor world | One directional shadow caster, ambient fill, environment |
| Interior | Focused spot lights, ambient fill, selected shadow casters |
| Dense repeated props | Shared lighting with mesh `castShadow` / `receiveShadow` flags |
| Large streaming world | A snapped camera-following directional light with a fitted shadow frustum |

Static scenes pair well with `shadowAutoUpdate: false` and a requested refresh after authored light or geometry changes.
