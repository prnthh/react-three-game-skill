For large scenes, bake the shadows.

// trigger a shadow update when needed
envMesh.castShadow = true;
directionalLight.current.shadow.autoUpdate = false;
directionalLight.current.shadow.needsUpdate = true;
