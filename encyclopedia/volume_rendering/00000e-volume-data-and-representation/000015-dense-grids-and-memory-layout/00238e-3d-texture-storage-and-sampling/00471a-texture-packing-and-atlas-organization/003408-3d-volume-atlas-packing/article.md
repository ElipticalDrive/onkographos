# 3D Volume Atlas Packing

Also called: Multi-volume 3D Texture Packing, Time-step Volume Atlas

## Overview
3D Volume Atlas Packing consolidates multiple discrete 3D volumes (such as animation timesteps or separate datasets) into a single physical 3D texture resource. This method reduces the number of active texture units and state transitions required during rendering by mapping logical volume indices into a unified coordinate space [S1, unofficial]. It differs from [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md), which combines semantic channels of a single volume, by handling distinct volumetric datasets along a dedicated atlas dimension.

## How it works
**Inputs:**
- `timeCount`: Integer, total number of volumes/timesteps to pack.
- `volumeResolution`: A 3D vector $(V_x, V_y, V_z)$ representing the voxel dimensions of a single source volume.
- `textureFilter`: Sampling mode (default: Linear).

**Outputs:**
- A flat buffer containing interleaved voxel data.
- A 3D texture object configured with specific dimensions and format.

**Data Structures:**
1.  **Voxel Buffer**: A flat `Uint16Array` of length $T_x \times T_y \times T_z$. Stored in row-major order: X is the fastest index, followed by Y, then Z. Each element is a 16-bit unsigned integer representing a single voxel value [S1, unofficial].
2.  **3D Texture**: A `Data3DTexture` with dimensions $(T_x, T_y, T_z)$, format `RedFormat`, type `HalfFloatType`, and wrapping set to `ClampToEdgeWrapping` on all axes [S1, unofficial].

**Algorithm:**
1.  Calculate the cubic root of the volume count to determine the atlas grid dimensions.
    $$ R = \lceil \sqrt[3]{\text{timeCount}} \rceil $$
2.  Set the atlas X and Y resolutions to $R$:
    $$ A_x = R, \quad A_y = R $$
3.  Calculate the atlas Z resolution to accommodate all volumes:
    $$ A_z = \lceil \text{timeCount} / (A_x \times A_y) \rceil $$
4.  Compute the final texture dimensions by scaling the single volume resolution by the atlas grid:
    $$ T_x = V_x \times A_x $$
    $$ T_y = V_y \times A_y $$
    $$ T_z = V_z \times A_z $$
5.  Allocate the flat buffer of size $T_x \times T_y \times T_z$.
6.  Populate the buffer with voxel data from each source volume. The mapping from logical volume index $k$ (where $0 \le k < \text{timeCount}$) and local voxel coordinates $(v_x, v_y, v_z)$ to global texture coordinates $(t_x, t_y, t_z)$ is:
    $$ \text{grid}_x = k \pmod{A_x} $$
    $$ \text{grid}_y = \lfloor k / A_x \rfloor \pmod{A_y} $$
    $$ \text{grid}_z = \lfloor k / (A_x \times A_y) \rfloor $$
    $$ t_x = v_x + \text{grid}_x \times V_x $$
    $$ t_y = v_y + \text{grid}_y \times V_y $$
    $$ t_z = v_z + \text{grid}_z \times V_z $$
7.  Create the 3D texture from the buffer with the calculated dimensions and format.
8.  Configure texture parameters: set min/mag filters to `textureFilter` and wrap modes to `ClampToEdgeWrapping` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_volumes(time_count, volume_resolution, texture_filter='linear'):
    # 1. Calculate atlas grid dimensions
    # R = ceil(cbrt(time_count))
    r = math.ceil(time_count ** (1/3))
    atlas_x = r
    atlas_y = r
    atlas_z = math.ceil(time_count / (atlas_x * atlas_y))

    # 2. Compute texture dimensions
    vol_x, vol_y, vol_z = volume_resolution
    tex_x = vol_x * atlas_x
    tex_y = vol_y * atlas_y
    tex_z = vol_z * atlas_z

    # 3. Allocate buffer
    buffer_size = tex_x * tex_y * tex_z
    voxels = [0] * buffer_size

    # 4. Populate buffer
    # Assumes source_volumes is a list of arrays, each of size vol_x * vol_y * vol_z
    # in row-major order (x fastest, then y, then z)
    for k in range(time_count):
        src_vol = source_volumes[k]
        
        # Determine grid position for this volume
        grid_x = k % atlas_x
        grid_y = (k // atlas_x) % atlas_y
        grid_z = (k // (atlas_x * atlas_y))
        
        # Offset in the global texture
        offset_x = grid_x * vol_x
        offset_y = grid_y * vol_y
        offset_z = grid_z * vol_z
        
        # Copy voxels
        for v_z in range(vol_z):
            for v_y in range(vol_y):
                for v_x in range(vol_x):
                    # Source index (row-major)
                    src_idx = (v_z * vol_y * vol_x) + (v_y * vol_x) + v_x
                    
                    # Target index (row-major)
                    # Global Z is slowest, then Y, then X
                    tgt_idx = ((v_z + offset_z) * tex_y * tex_x) + \
                              ((v_y + offset_y) * tex_x) + \
                              (v_x + offset_x)
                    
                    voxels[tgt_idx] = src_vol[src_idx]

    # 5. Create texture object
    # Implementation specific: create Data3DTexture(voxels, tex_x, tex_y, tex_z)
    # Set format: RedFormat
    # Set type: HalfFloatType
    # Set filter: texture_filter
    # Set wrap: ClampToEdgeWrapping for S, T, R
    return voxels
```

**Edge Cases:**
- If `timeCount` is 0, the grid dimensions should default to 1 or handle as empty [G].
- Precision issues in `cbrt` calculation may require epsilon adjustment for perfect cubes, though `ceil` generally handles integer inputs correctly [G].

## Subtopics

### [3D Atlas Texture Creation](7164-3d-atlas-texture-creation.md)
Handles the specific API calls to create the half-precision 3D texture and update shader uniforms based on the packed atlas parameters. It relies on the grid resolution and buffer allocation performed by this method.

## Operating characteristics
- **Memory Complexity**: $O(N \times V)$, where $N$ is the number of volumes and $V$ is the voxel count per volume. The packing adds minimal overhead beyond the raw data storage.
- **Time Complexity**: $O(N \times V)$ for the packing process, as each voxel must be copied to its new location.
- **Performance**: Reduces texture unit usage from $N$ to 1, significantly lowering state transition costs in rendering loops [S1, unofficial].
- **Limitations**: The method assumes the hardware supports 3D textures with the specified format (e.g., `RedFormat`, `HalfFloatType`). Interpolation across volume boundaries may produce artifacts if not handled carefully in the shader, as adjacent voxels in texture space may belong to different logical volumes [G].

## Background
This method is implemented in the `three.js-volume-renderer` project by Donitzo, which provides tools for rendering NIfTI medical images and other volumetric data in the browser [S1, unofficial]. It addresses the need to efficiently manage multiple time-steps or datasets within WebGL constraints.

## Sources
[S1] three.js-volume-renderer, https://github.com/Donitzo/three.js-volume-renderer, fetched 2026-10-28 (unofficial)
