# 3D Atlas Texture Creation

Also called: createAtlasTexture

## Overview
Creates a half-precision 3D atlas texture and updates shader uniforms based on volume parameters [S1, unofficial]. It packs multiple volumes or timesteps into a single 3D texture, building on [3D Volume Atlas Packing](7158-3d-volume-atlas-packing.md) which handles the initial grid resolution calculation and buffer allocation.

## How it works
**Inputs:**
- `volumeResolution`: `THREE.Vector3`. 3D resolution (voxel count) of one volume [S1, unofficial].
- `volumeOrigin`: `THREE.Vector3`. 3D world-space origin of the volume [S1, unofficial].
- `voxelSize`: `THREE.Vector3`. Physical 3D size of one voxel [S1, unofficial].
- `timeCount`: `number`. Total number of timesteps [S1, unofficial].
- `textureFilter`: `number`. Texture interpolation mode, default `THREE.LinearFilter` [S1, unofficial].
- `atlasBuffer`: `Uint16Array`. Flat buffer allocated by the parent packing step [S1, unofficial].

**Outputs:**
- A GPU 3D texture object (e.g., `THREE.DataTexture3D`) configured with `Uint16` type and `HalfFloatType` format [S1, unofficial].
- Updated shader uniforms: `volumeAtlas`, `atlasResolution`, `volumeResolution`, `voxelSize`, `timeCount` [S1, unofficial].

**Data Structures:**
The `atlasBuffer` is a flat `Uint16Array` containing the voxel data. The parent packing step determines the `atlasResolution` (dimensions of the 3D grid of volumes) [S1, unofficial].

**Steps:**
1. Create a 3D texture object using the WebGL/WebGPU context.
2. Set the texture type to `HalfFloatType` and format to `RedFormat` (or equivalent single-channel half-float format) [S1, unofficial].
3. Set the texture data to the `atlasBuffer` provided by the packing step.
4. Set the texture dimensions to the `atlasResolution` calculated by the parent [S1, unofficial].
5. Set the texture filter to `textureFilter` (default `LinearFilter`) [S1, unofficial].
6. Upload the texture to GPU memory.
7. Update the following shader uniforms:
   - `volumeAtlas`: The created texture object [S1, unofficial].
   - `atlasResolution`: The 3D resolution of the atlas grid [S1, unofficial].
   - `volumeResolution`: The input `volumeResolution` [S1, unofficial].
   - `voxelSize`: The input `voxelSize` [S1, unofficial].
   - `timeCount`: The input `timeCount` [S1, unofficial].
   - `volumeOrigin`: The input `volumeOrigin` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_atlas_texture(volume_resolution, volume_origin, voxel_size, time_count, texture_filter, atlas_buffer, atlas_resolution, gl_context):
    # Create texture object
    texture = gl_context.create_texture_3d()
    
    # Set type and format
    texture.set_type('HALF_FLOAT')
    texture.set_format('RED')
    
    # Set dimensions
    texture.set_dimensions(atlas_resolution.x, atlas_resolution.y, atlas_resolution.z)
    
    # Set data
    texture.set_data(atlas_buffer)
    
    # Set filter
    if texture_filter is None:
        texture_filter = 'LINEAR'
    texture.set_filter(texture_filter)
    
    # Upload to GPU
    texture.upload()
    
    # Update uniforms
    uniforms['volumeAtlas'] = texture
    uniforms['atlasResolution'] = atlas_resolution
    uniforms['volumeResolution'] = volume_resolution
    uniforms['voxelSize'] = voxel_size
    uniforms['timeCount'] = time_count
    uniforms['volumeOrigin'] = volume_origin
    
    return texture
```

**Edge Cases:**
- If `timeCount` is 0, the atlas may be empty or invalid [S1, unofficial].
- Precision loss occurs due to half-float storage [S1, unofficial].

## Subtopics

### [Atlas Budget Validation](8813-atlas-budget-validation.md)
Ensures the coarsest pyramid level's brick grid fits within the GPU's available VRAM budget, preventing out-of-memory (OOM) crashes during initialization, particularly on mobile GPUs. It acts as a gatekeeper before texture allocation, calculating the required memory for the base level and comparing it against a configurable limit.

## Operating characteristics
- **Memory:** Uses half-precision (16-bit) storage, reducing memory usage by 50% compared to single-precision [S1, unofficial].
- **Performance:** Sampling performance depends on the texture filter and GPU hardware [S1, unofficial].
- **Limitations:** Half-precision may cause precision issues for high dynamic range data [S1, unofficial].

## Background
Originated in the three.js-volume-renderer project [S1, unofficial].

## Sources
[S1] three.js-volume-renderer, https://github.com/Donitzo/three.js-volume-renderer, fetched 2026-10-28 (unofficial)
