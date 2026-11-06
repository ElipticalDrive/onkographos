# Volume Atlas Texture Update

Also called: Atlas Texture Update, Volume Atlas Update

## Overview
A method to populate a 3D volume atlas texture with sampled data values, tracking the minimum and maximum values encountered during the update process [S1, unofficial]. It differs from standard single-volume uploads by mapping a sequence of timesteps into a multi-volume 3D texture array (atlas), allowing temporal data to be stored in a single GPU texture object for efficient rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `sampler`: A function with signature `(xi, yi, zi, x, y, z, t) => number` that returns the scalar value for a given local voxel index, world position, and timestep [S1, unofficial].
    *   `timeOffset`: The starting timestep index for the update loop. Defaults to 0 if null [S1, unofficial].
    *   `timeCount`: The number of timesteps to update. Defaults to `this.uniforms.timeCount.value` if null [S1, unofficial].
    *   Uniforms containing `atlasResolutionX`, `atlasResolutionY`, `atlasResolutionZ` (dimensions of the atlas in volumes) and `volumeResolutionX`, `volumeResolutionY`, `volumeResolutionZ` (dimensions of each individual volume) [S1, unofficial].
    *   Uniforms containing `voxelSizeX`, `voxelSizeY`, `voxelSizeZ` (physical size of a voxel) and `volumeOriginX`, `volumeOriginY`, `volumeOriginZ` (world origin of the volume) [S1, unofficial].
*   **Outputs:** An object containing the computed `minValue` and `maxValue` of the sampled data [S1, unofficial].

**Data Structures:**
*   The texture data is stored in a 1D array representing the 3D texture, where each element is a half-float (16-bit floating-point number) [S1, unofficial].
*   The atlas texture is marked for update via `this.uniforms.volumeAtlas.value.needsUpdate = true` [S1, unofficial].

**Steps:**
1.  Extract atlas resolution (`atlasResolutionX`, `atlasResolutionY`, `atlasResolutionZ`) and volume resolution (`volumeResolutionX`, `volumeResolutionY`, `volumeResolutionZ`) from uniforms [S1, unofficial].
2.  Calculate total texture dimensions in voxels:
    $$
    \begin{aligned}
    textureSizeX &= volumeResolutionX \times atlasResolutionX \\
    textureSizeY &= volumeResolutionY \times atlasResolutionY \\
    textureSizeZ &= volumeResolutionZ \times atlasResolutionZ
    \end{aligned}
    $$
    [S1, unofficial]
3.  Initialize `minValue` to positive infinity and `maxValue` to negative infinity [S1, unofficial].
4.  Mark the volume atlas texture for update: `this.uniforms.volumeAtlas.value.needsUpdate = true` [S1, unofficial].
5.  Determine loop bounds:
    $$
    \begin{aligned}
    start &= timeOffset \text{ ?? } 0 \\
    count &= timeCount \text{ ?? } this.uniforms.timeCount.value \\
    end &= start + \min(count, this.uniforms.timeCount.value)
    \end{aligned}
    $$
    [S1, unofficial]
6.  Iterate timesteps $t$ from $start$ to $end$ (inclusive) [S1, unofficial].
7.  For each timestep $t$, calculate the atlas voxel indices (which volume in the atlas this timestep maps to):
    $$
    \begin{aligned}
    volumeIndexX &= t \pmod{atlasResolutionX} \\
    volumeIndexY &= \lfloor t / atlasResolutionX \rfloor \pmod{atlasResolutionY} \\
    volumeIndexZ &= \lfloor t / (atlasResolutionX \times atlasResolutionY) \rfloor
    \end{aligned}
    $$
    [S1, unofficial]
8.  Iterate local voxel indices $xi$ from 0 to $volumeResolutionX$, $yi$ from 0 to $volumeResolutionY$, and $zi$ from 0 to $volumeResolutionZ$ [S1, unofficial].
9.  Compute world position:
    $$
    \begin{aligned}
    x &= xi \times voxelSizeX + volumeOriginX \\
    y &= yi \times voxelSizeY + volumeOriginY \\
    z &= zi \times voxelSizeZ + volumeOriginZ
    \end{aligned}
    $$
    [S1, unofficial]
10. Call `sampler(xi, yi, zi, x, y, z, t)` to get the value [S1, unofficial].
11. Update `minValue` and `maxValue` using `Math.min` and `Math.max` with the returned value [S1, unofficial].
12. Calculate atlas texture indices (global coordinates in the 3D texture array):
    $$
    \begin{aligned}
    xai &= volumeIndexX \times volumeResolutionX + xi \\
    yai &= volumeIndexY \times volumeResolutionY + yi \\
    zai &= volumeIndexZ \times volumeResolutionZ + zi
    \end{aligned}
    $$
    [S1, unofficial]
13. Calculate linear index in the 1D array representing the 3D texture:
    $$
    i = xai + yai \times textureSizeX + zai \times textureSizeX \times textureSizeY
    $$
    [S1, unofficial]
14. Store the value in the texture data array converted to half-float: `voxels[i] = THREE.DataUtils.toHalfFloat(value)` [S1, unofficial].
15. After all timesteps and voxels are processed, return an object containing the computed `minValue` and `maxValue` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_volume_atlas_texture(sampler, timeOffset=None, timeCount=None):
    # Extract resolutions from uniforms
    atlas_res_x = uniforms.atlasResolutionX.value
    atlas_res_y = uniforms.atlasResolutionY.value
    atlas_res_z = uniforms.atlasResolutionZ.value
    
    vol_res_x = uniforms.volumeResolutionX.value
    vol_res_y = uniforms.volumeResolutionY.value
    vol_res_z = uniforms.volumeResolutionZ.value
    
    # Calculate total texture dimensions
    texture_size_x = vol_res_x * atlas_res_x
    texture_size_y = vol_res_y * atlas_res_y
    texture_size_z = vol_res_z * atlas_res_z
    
    # Initialize min/max
    min_value = float('inf')
    max_value = float('-inf')
    
    # Mark texture for update
    uniforms.volumeAtlas.value.needsUpdate = True
    
    # Determine loop bounds
    start = timeOffset if timeOffset is not None else 0
    count = timeCount if timeCount is not None else uniforms.timeCount.value
    end = start + min(count, uniforms.timeCount.value)
    
    # Extract voxel size and origin
    voxel_size_x = uniforms.voxelSizeX.value
    voxel_size_y = uniforms.voxelSizeY.value
    voxel_size_z = uniforms.voxelSizeZ.value
    volume_origin_x = uniforms.volumeOriginX.value
    volume_origin_y = uniforms.volumeOriginY.value
    volume_origin_z = uniforms.volumeOriginZ.value
    
    # Get the texture data array (assumed to be pre-allocated and accessible)
    voxels = uniforms.volumeAtlas.value.image.data
    
    for t in range(start, end + 1):
        # Calculate atlas voxel indices for the current timestep
        volume_index_x = t % atlas_res_x
        volume_index_y = (t // atlas_res_x) % atlas_res_y
        volume_index_z = t // (atlas_res_x * atlas_res_y)
        
        for xi in range(vol_res_x):
            for yi in range(vol_res_y):
                for zi in range(vol_res_z):
                    # Compute world position
                    x = xi * voxel_size_x + volume_origin_x
                    y = yi * voxel_size_y + volume_origin_y
                    z = zi * voxel_size_z + volume_origin_z
                    
                    # Get value from sampler
                    value = sampler(xi, yi, zi, x, y, z, t)
                    
                    # Update min/max
                    if value < min_value:
                        min_value = value
                    if value > max_value:
                        max_value = value
                    
                    # Calculate atlas texture indices
                    xai = volume_index_x * vol_res_x + xi
                    yai = volume_index_y * vol_res_y + yi
                    zai = volume_index_z * vol_res_z + zi
                    
                    # Calculate linear index
                    i = xai + yai * texture_size_x + zai * texture_size_x * texture_size_y
                    
                    # Store as half-float
                    voxels[i] = to_half_float(value)
    
    return {'minValue': min_value, 'maxValue': max_value}

def to_half_float(value):
    # Implementation of IEEE 754 half-precision conversion
    # [G] Standard conversion logic: sign, exponent, mantissa packing
    import struct
    # Pack as 32-bit float, then extract and pack as 16-bit half
    f32 = struct.pack('f', value)
    i32 = struct.unpack('I', f32)[0]
    sign = (i32 >> 16) & 0x8000
    exp = ((i32 >> 23) & 0xff) - 127 + 15
    frac = (i32 >> 13) & 0x3ff
    if exp <= 0:
        return 0 # Handle subnormals/zero as zero for simplicity
    if exp >= 0x1f:
        return sign | 0x7c00 # Infinity or NaN
    return sign | (exp << 10) | frac
```

**Parameters:**
*   `sampler`: Function `(xi, yi, zi, x, y, z, t) => number` [S1, unofficial].
*   `timeOffset`: Starting timestep index. Default 0 [S1, unofficial].
*   `timeCount`: Number of timesteps. Default `this.uniforms.timeCount.value` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Values are converted to half-floats, which may introduce quantization errors for high-precision data [S1, unofficial].
*   **Boundaries:** The loop bounds calculation uses `Math.min` to ensure `end` does not exceed the available time count, preventing out-of-bounds access [S1, unofficial].
*   **Degenerate Input:** If `timeCount` is 0, the loop range is empty, and min/max remain infinity/negative infinity [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(T \times V_x \times V_y \times V_z)$, where $T$ is the number of timesteps updated and $V_x, V_y, V_z$ are the volume resolutions [S1, unofficial].
*   **Memory Complexity:** $O(A_x \times A_y \times A_z \times V_x \times V_y \times V_z)$ for the texture data, where $A_x, A_y, A_z$ are the atlas resolutions [S1, unofficial].
*   **Strengths:** Efficiently stores temporal data in a single texture object, reducing texture binding overhead [S1, unofficial].
*   **Limitations:** Requires CPU-side iteration to populate the texture, which can be a bottleneck for large volumes or many timesteps [S1, unofficial].
*   **When to use:** When rendering time-varying volume data and the atlas size fits within GPU texture limits [S1, unofficial].
*   **When not to use:** When the total number of voxels exceeds the maximum 3D texture size supported by the hardware [S1, unofficial].

## Background
This method is part of the `three.js-volume-renderer` library, which provides volume rendering capabilities for Three.js [S1, unofficial]. It was developed to support dynamic volume data, such as medical imaging (NIfTI files) and geometry-based SDFs, by allowing efficient updates of multi-volume textures [S1, unofficial].

## Sources
[S1] here, https://github.com/Donitzo/three.js-volume-renderer, fetched 2026-10-28 (unofficial)
