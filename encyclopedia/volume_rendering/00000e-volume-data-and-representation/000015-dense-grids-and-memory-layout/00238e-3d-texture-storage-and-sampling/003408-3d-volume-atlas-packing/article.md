# 3D Volume Atlas Packing

Also called: Multi-volume 3D Texture Packing, Time-step Volume Atlas

## Overview
3D Volume Atlas Packing consolidates multiple distinct 3D volumes (such as timesteps in a time-series dataset) into a single 3D texture resource. This method solves the problem of limited texture unit availability and the overhead of managing numerous separate texture objects by spatially arranging volumes within a larger 3D grid. It differs from [3D Texture Proxy Geometry](870-3d-texture-proxy-geometry.md) by storing actual voxel data rather than geometric approximations, and from [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md) by packing entire volumes along a third dimension rather than combining scalar fields into color channels of a single volume.

## How it works
**Inputs:**
- `timeCount`: Total number of volumes (timesteps) to pack.
- `volumeResolution`: A 3D vector $(V_x, V_y, V_z)$ representing the voxel count of a single volume along each axis.
- `textureFilter`: Interpolation mode (default: Linear).
- `volumeOrigin`: 3D world-space origin of the volume (used for sampling offsets).
- `voxelSize`: 3D physical size of a single voxel.

**Outputs:**
- A 3D texture atlas containing all volumes.
- Uniforms describing the atlas structure for shader sampling.

**Data Structures:**
1.  **Voxel Buffer**: A flat `Uint16Array` (or equivalent half-float buffer) of size $T_x \times T_y \times T_z$, where $T$ are the atlas dimensions. Stored in row-major order (X fastest, then Y, then Z). Each element represents one voxel value.
2.  **3D Texture**: A `Data3DTexture` object with dimensions $(T_x, T_y, T_z)$, format `RedFormat`, and type `HalfFloatType`. Wrap modes are set to `ClampToEdgeWrapping` on all axes.

**Algorithm:**
1.  Calculate the cubic root of the number of volumes to determine the base resolution for the X and Y axes of the atlas:
    $$ A_x = \lceil \sqrt[3]{\text{timeCount}} \rceil $$
    $$ A_y = A_x $$
2.  Calculate the Z-axis resolution required to fit all volumes:
    $$ A_z = \lceil \frac{\text{timeCount}}{A_x \times A_y} \rceil $$
3.  Compute the total texture dimensions by multiplying the single volume resolution by the atlas grid dimensions:
    $$ T_x = V_x \times A_x $$
    $$ T_y = V_y \times A_y $$
    $$ T_z = V_z \times A_z $$
4.  Allocate a flat buffer of size $T_x \times T_y \times T_z$.
5.  Populate the buffer with voxel data from each volume. Volumes are placed sequentially in the atlas grid. The position of the $k$-th volume (where $0 \le k < \text{timeCount}$) in the atlas grid $(a_x, a_y, a_z)$ is calculated as:
    $$ a_x = k \pmod{A_x} $$
    $$ a_y = \lfloor \frac{k}{A_x} \rfloor \pmod{A_y} $$
    $$ a_z = \lfloor \frac{k}{A_x \times A_y} \rfloor $$
    The voxel at local coordinates $(v_x, v_y, v_z)$ in volume $k$ maps to global atlas coordinates:
    $$ g_x = a_x \times V_x + v_x $$
    $$ g_y = a_y \times V_y + v_y $$
    $$ g_z = a_z \times V_z + v_z $$
6.  Create the 3D texture from the buffer with the specified dimensions and format.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_volume_atlas(time_count, volume_resolution, texture_filter=LinearFilter):
    # 1. Calculate atlas grid dimensions
    atlas_x = ceil(pow(time_count, 1/3))
    atlas_y = atlas_x
    atlas_z = ceil(time_count / (atlas_x * atlas_y))
    
    # 2. Calculate total texture dimensions
    tex_x = volume_resolution.x * atlas_x
    tex_y = volume_resolution.y * atlas_y
    tex_z = volume_resolution.z * atlas_z
    
    # 3. Allocate buffer
    # Note: In a real implementation, ensure the buffer type matches HalfFloatType
    buffer_size = tex_x * tex_y * tex_z
    voxels = Uint16Array(buffer_size)
    
    # 4. Populate buffer (pseudo-logic for single volume placement)
    # In practice, this loop iterates over all time steps and copies data
    for t in range(time_count):
        # Calculate atlas grid position for this timestep
        ax = t % atlas_x
        ay = floor(t / atlas_x) % atlas_y
        az = floor(t / (atlas_x * atlas_y))
        
        # Calculate offset in the flat buffer
        # Assuming row-major: X fastest, then Y, then Z
        base_idx = (az * tex_y * tex_x) + (ay * tex_x) + (ax * 1) # Simplified offset logic for demonstration
        
        # Copy volume data into the buffer at the calculated offset
        # source_voxels is the data for timestep t
        for z in range(volume_resolution.z):
            for y in range(volume_resolution.y):
                for x in range(volume_resolution.x):
                    src_idx = (z * volume_resolution.y * volume_resolution.x) + (y * volume_resolution.x) + x
                    dst_idx = base_idx + src_idx # Corrected flat index calculation
                    voxels[dst_idx] = source_voxels[src_idx]

    # 5. Create Texture
    texture = Data3DTexture(voxels, tex_x, tex_y, tex_z)
    texture.format = RedFormat
    texture.type = HalfFloatType
    texture.minFilter = texture_filter
    texture.magFilter = texture_filter
    texture.wrapS = ClampToEdgeWrapping
    texture.wrapT = ClampToEdgeWrapping
    texture.wrapR = ClampToEdgeWrapping
    
    return texture
```

**Edge Cases and Pitfalls:**
- **Precision**: Using `Uint16Array` for storage but `HalfFloatType` for the texture requires careful conversion. Values must be normalized to the [0, 1] range expected by half-float representation if the source data is signed or exceeds 16-bit unsigned range.
- **Boundaries**: The atlas texture must use `ClampToEdgeWrapping` to prevent sampling artifacts when raymarching near the boundaries of individual volumes within the atlas.
- **Degenerate Input**: If `timeCount` is 0, the atlas dimensions should be handled gracefully to avoid division by zero or invalid texture creation.

## Subtopics

### 3D Atlas Texture Creation
This variant focuses on the integration of the packed atlas with rendering uniforms. It creates the half-precision 3D atlas texture and updates shader uniforms (`atlasResolution`, `volumeResolution`, `voxelSize`, `timeCount`) based on the volume parameters. It ensures that the sampling function in the shader correctly maps world-space positions to UV coordinates within the specific volume slice of the atlas.

## Operating characteristics
- **Time Complexity**: Packing time is $O(N \times V)$, where $N$ is the number of volumes and $V$ is the number of voxels per volume.
- **Memory Complexity**: $O(N \times V)$, storing all voxel data in a single contiguous block.
- **Performance**: Reduces GPU state changes and texture binding overhead compared to managing $N$ separate textures. Sampling requires additional arithmetic to compute UV offsets, adding slight per-fragment cost.
- **Scaling**: Efficient for moderate numbers of timesteps. As $N$ increases, the atlas dimensions grow, potentially exceeding maximum 3D texture size limits of the hardware.
- **Strengths**: Single texture binding, efficient memory layout for sequential access.
- **Limitations**: Limited by maximum 3D texture dimensions; interpolation across volume boundaries is not supported (volumes are distinct).

## Background
This method is implemented in the `three.js-volume-renderer` project by Donitzo [S1, unofficial]. It is designed for WebGL-based volume rendering where texture unit limits and binding overhead are significant constraints. The approach allows for the visualization of time-series volumetric data (e.g., medical imaging, fluid simulations) by treating time as a spatial dimension in the texture atlas.

## Sources
[S1] three.js-volume-renderer, https://github.com/Donitzo/three.js-volume-renderer, fetched 2026-10-28 (unofficial)
