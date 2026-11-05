# Lighting

Start with one directional sun and hemisphere/environment fill. Add local lights as needed.

| Component | Main properties |
| --- | --- |
| `DirectionalLight` | `color`, `intensity`, `targetOffset`, shadow settings |
| `PointLight` | `color`, `intensity`, `distance`, `decay` |
| `SpotLight` | Point-light settings plus `angle`, `penumbra`, `targetOffset`, `map` |
| `HemisphereLight` | `skyColor`, `groundColor`, `intensity` |
| `Environment` | `intensity`, `frames`, `resolution`, `background` |
| `Fog` | `color`, `near`, `far` |

Author `RenderSettings` on the scene root for explicit exposure, tone mapping and
shadow overrides. Unspecified fields retain Three.js/host defaults. Enable renderer
shadows with `shadows:true`, then enable `castShadow` on lights and meshes; receivers need `receiveShadow`. Keep shadow-camera bounds close to the area that matters.

For static lighting, set `shadowAutoUpdate: false`. After changing scene content, set the Three light's `shadow.needsUpdate = true`; invalidate the canvas if it renders on demand. Keep automatic updates enabled for moving content.

For moving outdoor views, use `DirectionalLight.shadowCascades: 2` or `3`, with a suitable `shadowDistance`. Cascades update every frame. One cascade means an ordinary shadow map.

Enable `PostProcessing.ambientOcclusion` and/or `PostProcessing.bloom` for those
effects. Use the root `ClusteredLighting` component for many local lights.

`Environment` captures its child objects for lighting and reflections. Add a child
`Sky` for a procedural environment, or a child sphere with a basic `BackSide` image
material for an image. `background` displays the captured environment and defaults to `true`.
`DiffuseLightProbe` supplies authored scene-wide sky/ground diffuse lighting.

Use one active Fog node with `far > near`. Fog affects the whole scene.
