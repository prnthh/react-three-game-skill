# Lighting

Start with one directional sun and hemisphere/environment fill. Add local lights as needed.

| Component | Main properties |
| --- | --- |
| `DirectionalLight` | `color`, `intensity`, `targetOffset`, shadow settings |
| `PointLight` | `color`, `intensity`, `distance`, `decay` |
| `SpotLight` | Point-light settings plus `angle`, `penumbra`, `targetOffset`, `map` |
| `HemisphereLight` | `skyColor`, `groundColor`, `intensity` |
| `Environment` | `intensity`, `resolution`, `background` |
| `Fog` | `color`, `near`, `far` |

Enable `castShadow` on lights and meshes; receivers need `receiveShadow`. Keep shadow-camera bounds close to the area that matters.

For static lighting, set `shadowAutoUpdate: false`. After changing casters/lights or chunks, set the Three light's `shadow.needsUpdate = true`; invalidate the canvas if it renders on demand. Keep automatic updates enabled for moving content.

For moving outdoor views, use `DirectionalLight.shadowCascades: 2` or `3`, with a suitable `shadowDistance`. Cascades update every frame. One cascade means an ordinary shadow map.

Use one active Fog node with `far > near`. Its location in the hierarchy does not limit its scene-wide effect.
