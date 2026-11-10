# 3D Volume Atlas Packing

Also called: Multi-volume 3D Texture Packing, Time-step Volume Atlas

## Overview
3D Volume Atlas Packing consolidates multiple discrete 3D volumes (such as animation timesteps or separate datasets) into a single contiguous 3D texture resource. This method solves the problem of excessive texture unit usage and state changes when rendering multiple volumes by mapping them into a unified spatial grid. It differs from [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md), which combines semantic channels into RGBA components of a single volume, by instead arranging distinct volumetric data sets into a larger 3D grid structure. It is related to [Image Texture Atlas Packing](11896-image-texture-atlas-packing.md), which applies similar tiling principles to 2D textures.

## How it works
**Inputs:**
- `timeCount`: Integer, total number of volumes (timesteps) to pack.
- `volumeResolution`: 3D vector $(V_x, V_y, V_z)$, the voxel resolution of a single input volume.
- `textureFilter`: Sampling mode (default: Linear).

**Outputs:**
- A flat voxel buffer (`Uint16Array`) containing the packed data.
- A 3D texture object configured with specific dimensions, format, and wrapping.

**Data Structures:**
1.  **Voxel Buffer:** A flat `Uint16Array` of size $T_x \times T_y \times T_z$. Elements are stored in row-major order: X is the fastest index, followed by Y, then Z. Each element is a 16-bit unsigned integer representing a single voxel value.
2.  **3D Texture:** A `Data3DTexture` with dimensions $(T_x, T_y, T_z)$, format `RedFormat`, type `HalfFloatType`, and `ClampToEdgeWrapping` on all axes.

**Steps:**
1.  Calculate the cubic root of the number of volumes to determine the grid dimensions:
    $$ A_x = \lceil \sqrt[3]{\text{timeCount}} \rceil $$
    $$ A_y = A_x $$
    $$ A_z = \lceil \text{timeCount} / (A_x \times A_y) \rceil $$
    Where $A_x, A_y, A_z$ are the atlas grid dimensions (number of volumes along each axis).
2.  Compute the total texture dimensions by multiplying the single volume resolution by the atlas grid dimensions:
    $$ T_x = V_x \times A_x $$
    $$ T_y = V_y \times A_y $$
    $$ T_z = V_z \times A_z $$
3.  Allocate a flat `Uint16Array` of size $T_x \times T_y \times T_z$.
4.  Populate the buffer by copying voxel data from each input volume into the corresponding sub-block within the atlas. The mapping from input volume index $k$ ($0 \le k < \text{timeCount}$) to atlas coordinates $(a_x, a_y, a_z)$ is:
    $$ a_x = k \pmod{A_x} $$
    $$ a_y = \lfloor k / A_x \rfloor \pmod{A_y} $$
    $$ a_z = \lfloor k / (A_x \times A_y) \rfloor $$
    The voxel at local coordinates $(v_x, v_y, v_z)$ in volume $k$ maps to atlas coordinates:
    $$ (a_x \times V_x + v_x, \ a_y \times V_y + v_y, \ a_z \times V_z + v_z) $$
5.  Create the 3D texture from the buffer with dimensions $(T_x, T_y, T_z)$, format `RedFormat`, and type `HalfFloatType`.
6.  Set filtering to `textureFilter` and wrapping to `ClampToEdgeWrapping` for all axes.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_volumes(time_count, volume_resolution, texture_filter='linear'):
    # 1. Calculate atlas grid dimensions
    # A_x = ceil(cbrt(time_count))
    # A_y = A_x
    # A_z = ceil(time_count / (A_x * A_y))
    import math
    ax = math.ceil(time_count ** (1/3))
    ay = ax
    az = math.ceil(time_count / (ax * ay))

    # 2. Compute total texture dimensions
    vx, vy, vz = volume_resolution
    tx = vx * ax
    ty = vy * ay
    tz = vz * az

    # 3. Allocate buffer
    # Uint16Array of size tx * ty * tz
    buffer_size = tx * ty * tz
    voxels = [0] * buffer_size

    # 4. Populate buffer
    # This step assumes access to input volumes. 
    # In a real implementation, input volumes would be passed as a list of arrays.
    # For each volume k:
    for k in range(time_count):
        # Determine atlas position for this volume
        ax_pos = k % ax
        ay_pos = (k // ax) % ay
        az_pos = (k // (ax * ay))
        
        # Calculate offset in the flat buffer
        # Buffer layout: X fastest, then Y, then Z
        # Offset for a voxel at (x, y, z) in atlas is: x + y*tx + z*tx*ty
        base_offset = (az_pos * tz + ay_pos * ty + ax_pos * tx) * (vx * vy * vz) # This is incorrect logic for flat buffer indexing of sub-blocks
        
        # Correct indexing for sub-block copy:
        # The sub-block for volume k starts at atlas coordinates:
        # x_start = ax_pos * vx
        # y_start = ay_pos * vy
        # z_start = az_pos * vz
        
        # We need to copy data from input_volume[k] into voxels
        # input_volume[k] is assumed to be a flat array of size vx*vy*vz
        # mapped to atlas coordinates (x_start + i, y_start + j, z_start + l)
        
        x_start = ax_pos * vx
        y_start = ay_pos * vy
        z_start = az_pos * vz
        
        # Copying logic:
        # For each voxel (i, j, l) in the single volume:
        # atlas_x = x_start + i
        # atlas_y = y_start + j
        # atlas_z = z_start + l
        # buffer_index = atlas_x + atlas_y * tx + atlas_z * tx * ty
        
        # Since we don't have input volumes in this signature, we mark this step
        # as requiring external data access.
        pass 

    # 5. Create texture
    # Create Data3DTexture with dimensions (tx, ty, tz)
    # Set format to RedFormat, type to HalfFloatType
    # Set minFilter/magFilter to texture_filter
    # Set wrapS/wrapT/wrapR to ClampToEdgeWrapping
    pass

    return voxels
```

**Edge Cases:**
- **Precision:** Using `Uint16Array` limits voxel values to 0–65535. If input data exceeds this range, it must be normalized or scaled before packing. The texture type `HalfFloatType` suggests floating-point storage, but the buffer is `Uint16Array`. This implies the renderer may interpret the 16-bit integers as half-floats or the source data is already quantized. [S1, unofficial]
- **Boundaries:** `ClampToEdgeWrapping` is used to prevent sampling artifacts at the edges of the atlas or individual volumes.

## Subtopics

### [3D Atlas Texture Creation](7164-3d-atlas-texture-creation.md)
Creates a half-precision 3D atlas texture and updates shader uniforms based on volume parameters. It builds on [3D Volume Atlas Packing](7158-3d-volume-atlas-packing.md) by handling the actual texture object creation and uniform management after the grid resolution and buffer allocation are determined.

## Operating characteristics
- **Time Complexity:** Grid calculation is $O(1)$. Buffer allocation is $O(N)$ where $N$ is the total number of voxels. Populating the buffer is $O(N)$.
- **Memory Complexity:** $O(N)$ for the voxel buffer and texture storage.
- **Performance:** Reduces the number of active texture units to one, minimizing state changes. However, it increases the total texture size, which may exceed maximum texture dimensions if the number of volumes or resolution is too high.
- **Strengths:** Efficient for rendering animated volumes or multiple datasets with a single draw call.
- **Limitations:** Limited by maximum 3D texture size. Requires careful coordinate mapping in shaders.

## Background
This method is implemented in the `three.js-volume-renderer` project by Donitzo, which is used for rendering NIfTI medical imaging data and other volumetric datasets. It is designed to handle animated smoke and other time-varying volumes efficiently. [S1, unofficial]

## Sources
[S1] three.js-volume-renderer, https://github.com/Donitzo/three.js-volume-renderer, fetched 2026-10-28 (unofficial)
