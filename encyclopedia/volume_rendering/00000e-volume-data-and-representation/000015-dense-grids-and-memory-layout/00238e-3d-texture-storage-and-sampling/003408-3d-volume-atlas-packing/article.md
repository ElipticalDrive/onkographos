# 3D Volume Atlas Packing

Also called: Multi-volume 3D Texture Packing, Time-step Volume Atlas

## Overview
3D Volume Atlas Packing consolidates multiple discrete volumes (such as animation timesteps or separate datasets) into a single contiguous 3D texture resource. This method solves the problem of managing numerous individual texture objects by reducing GPU state changes and memory fragmentation. It differs from [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md), which packs different data channels into one volume, by packing identical channel structures across the depth dimension. It builds on [3D Texture Storage and Sampling](3064-3d-texture-storage-and-sampling.md) by defining the specific layout and allocation strategy for these multi-volume containers.

## How it works
**Inputs:**
- `timeCount`: Integer, total number of volumes/timesteps to pack.
- `volumeResolution`: 3D vector $(V_x, V_y, V_z)$, the voxel resolution of a single volume.
- `textureFilter`: Sampling mode (default: Linear).

**Outputs:**
- A flat `Uint16Array` buffer containing all voxel data.
- A 3D texture object configured with specific dimensions and format.

**Data Structures:**
1.  **Voxel Buffer**: A flat `Uint16Array` of size $T_x \times T_y \times T_z$. Elements are stored in row-major order: X is the fastest index, followed by Y, then Z. Each element is a 16-bit unsigned integer.
2.  **3D Texture**: A `Data3DTexture` object with dimensions $(T_x, T_y, T_z)$, format `RedFormat`, type `HalfFloatType`, and wrap modes set to `ClampToEdgeWrapping` on all axes.

**Algorithm:**
1.  Calculate the cubic root of the number of volumes to determine the atlas grid dimensions.
    $$ R = \lceil \sqrt[3]{\text{timeCount}} \rceil $$
2.  Set the atlas width and height in volume units:
    $$ A_x = R $$
    $$ A_y = R $$
3.  Calculate the atlas depth in volume units, ensuring all volumes fit:
    $$ A_z = \lceil \frac{\text{timeCount}}{A_x \times A_y} \rceil $$
4.  Compute the final texture dimensions by scaling the single volume resolution by the atlas grid size:
    $$ T_x = V_x \times A_x $$
    $$ T_y = V_y \times A_y $$
    $$ T_z = V_z \times A_z $$
5.  Allocate the flat buffer:
    $$ \text{buffer} = \text{new Uint16Array}(T_x \times T_y \times T_z) $$
6.  Create the 3D texture using the buffer and dimensions $(T_x, T_y, T_z)$.
7.  Configure texture properties:
    -   Format: `RedFormat`
    -   Type: `HalfFloatType`
    -   Min/Mag Filter: `textureFilter`
    -   Wrap S/T/R: `ClampToEdgeWrapping`

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_volume_atlas(time_count, volume_resolution, texture_filter="linear"):
    # 1. Calculate atlas grid dimensions
    # R = ceil(time_count^(1/3))
    r = math.ceil(time_count ** (1/3))
    
    atlas_x = r
    atlas_y = r
    
    # A_z = ceil(time_count / (atlas_x * atlas_y))
    atlas_z = math.ceil(time_count / (atlas_x * atlas_y))
    
    # 2. Compute texture dimensions
    tex_x = volume_resolution.x * atlas_x
    tex_y = volume_resolution.y * atlas_y
    tex_z = volume_resolution.z * atlas_z
    
    # 3. Allocate buffer
    # Size is total number of voxels in the atlas
    buffer_size = tex_x * tex_y * tex_z
    buffer = [0] * buffer_size
    
    # 4. Create texture object
    # In a real implementation, this would call the graphics API constructor
    texture = create_data_3d_texture(
        width=tex_x,
        height=tex_y,
        depth=tex_z,
        data=buffer,
        format="RedFormat",
        type="HalfFloatType",
        min_filter=texture_filter,
        mag_filter=texture_filter,
        wrap_s="ClampToEdgeWrapping",
        wrap_t="ClampToEdgeWrapping",
        wrap_r="ClampToEdgeWrapping"
    )
    
    return buffer, texture
```

**Edge Cases and Pitfalls:**
-   **Precision**: The source uses `Uint16Array` for storage but `HalfFloatType` for the texture. Implementers must ensure the voxel values fit within the 16-bit unsigned integer range and are correctly interpreted as half-precision floats by the GPU. [G] If the source data exceeds 16-bit integer precision, values will be truncated or wrapped.
-   **Boundaries**: Wrap modes are explicitly set to `ClampToEdgeWrapping`. Sampling outside the atlas bounds will clamp to the edge values, which is critical for preventing artifacts when raymarching near the boundaries of individual volumes within the atlas.
-   **Degenerate Input**: If `timeCount` is 0, the resolution calculation results in 0 dimensions. A guard should prevent allocation of zero-sized textures.

## Subtopics

### [3D Atlas Texture Creation](7164-3d-atlas-texture-creation.md)
Creates the actual half-precision 3D atlas texture and updates shader uniforms. It relies on the resolution calculation and buffer allocation performed by this method.

## Operating characteristics
-   **Time Complexity**: The resolution calculation is $O(1)$. Buffer allocation is $O(N)$ where $N$ is the total number of voxels. Populating the buffer depends on the source data transfer method.
-   **Memory Complexity**: $O(N)$, where $N = V_x \times V_y \times V_z \times A_x \times A_y \times A_z$. The memory usage is strictly proportional to the total voxel count.
-   **Performance**: Packing volumes into a single texture reduces the number of texture bindings required during rendering, which can improve performance for animated volumes compared to binding multiple separate textures.
-   **Limitations**: The method assumes all volumes have identical resolutions. It does not handle volumes of different sizes. The use of `Uint16Array` limits the dynamic range of stored values compared to 32-bit floats, though `HalfFloatType` in the texture provides some flexibility.

## Background
This method is implemented in the `three.js-volume-renderer` project by Donitzo [S1, unofficial]. It is designed for WebGL-based volume rendering where efficient texture management is crucial. The approach is specific to the Three.js library's `Data3DTexture` API.

## Sources
[S1] three.js-volume-renderer, https://github.com/Donitzo/three.js-volume-renderer, fetched 2026-10-28 (unofficial)
