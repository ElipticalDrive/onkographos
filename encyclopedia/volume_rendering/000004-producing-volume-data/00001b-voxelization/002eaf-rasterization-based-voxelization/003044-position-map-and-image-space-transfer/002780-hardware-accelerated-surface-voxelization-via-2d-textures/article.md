# Hardware-accelerated surface voxelization via 2D textures

Also called: Image-space voxelization

## Overview
This method transfers intersection computation from geometric space to image space by rasterizing model surfaces into 2D textures, enabling collision detection and volumetric rendering at real-time frame rates [S1, unofficial]. It differs from slicing-based voxelization by rasterizing the scene approximately once rather than $N$ times (where $N$ is the volume resolution), making performance insensitive to scene complexity [S1, unofficial]. Unlike Layered Depth Image (LDI) extraction, it is view-independent and avoids CPU-side sorting of depth lists [S1, unofficial].

## How it works
The method consists of two primary stages: generating a position map via rasterization and populating the 3D voxel grid from that map.

**Inputs and Outputs**
*   **Input:** A polygonal mesh (triangles) or implicit surface definition; volume bounding box $[xBounds, yBounds, zBounds]$; volume resolution `voxelDim` (e.g., $256^3$); bit-depth per voxel (e.g., 8 bits).
*   **Output:** A 3D volume texture (or worksheet) where active voxels are marked.

**Data Structures**
1.  **Position Map:** A 2D texture (typically `rgba32f` or similar high-precision format) storing world-space coordinates. Each texel $(u, v)$ stores the world-space position $pos = (pos.x, pos.y, pos.z)$ and a validity flag (often in the alpha channel or implied by depth) [S1, unofficial].
2.  **Volume Grid:** A 3D texture or array of bitmasks/bytes. For a $512^3$ resolution, a voxel may be represented by one bit in a worksheet; for $256^3$, 8 bits (1 byte) is standard [S1, unofficial].

**Numbered Steps**

1.  **Render Position Map:**
    *   Bind the 2D position map texture.
    *   Enable depth testing.
    *   Rasterize the input geometry (triangles or instanced quads for implicit surfaces) into the 2D texture.
    *   In the fragment shader, output the world-space position of the fragment. The depth test ensures only the nearest surface point updates the texture [S1, unofficial].

2.  **Populate Voxel Grid:**
    *   Bind the 3D volume texture.
    *   Disable depth testing and color masking to allow direct writes without overwriting concerns [S1, unofficial].
    *   Render a full-screen quad.
    *   For each fragment, sample the position map at the corresponding screen coordinates to retrieve $pos$.
    *   Convert $pos$ to 3D voxel indices $(x, y, z)$ using the volume bounds and resolution.
    *   Write to the corresponding voxel in the 3D texture (e.g., set to 1 or mark as active) [S1, unofficial].

**Formulas**
The conversion from world position $pos = (pos.x, pos.y, pos.z)$ to voxel indices $(x, y, z)$ for a volume of dimension `voxelDim` and bounds $[xBounds, yBounds, zBounds]$ is:

$$
x = \text{voxelDim} \times \frac{pos.x - xBounds.x}{xBounds.y - xBounds.x}
$$

$$
y = \text{voxelDim} \times \frac{pos.y - yBounds.x}{yBounds.y - yBounds.x}
$$

$$
z = \text{voxelDim} \times \frac{pos.z - zBounds.x}{zBounds.y - zBounds.x}
$$

Standard integer truncation or floor is implied by the return type `ivec3` in typical GLSL implementations [G].

**Parameters**
*   **Volume Resolution:** $256^3$ is identified as a good choice under current hardware conditions [S1, unofficial].
*   **Bit-depth:** 8 bits per voxel is used for the standard $256^3$ resolution configuration [S1, unofficial]. For $512^3$, 1 bit per voxel may be used to fit in one worksheet [S1, unofficial]. For $128^3$, vertex normals may be encoded using 16 bits [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Stage 1: Render Position Map
def render_position_map(geometry, position_map_texture):
    """
    Rasterizes geometry into a 2D texture storing world-space positions.
    """
    enable_depth_test()
    bind_texture(position_map_texture)
    for triangle in geometry.triangles:
        rasterize_triangle(triangle)
        # Fragment shader logic:
        # world_pos = interpolate_world_positions(triangle.vertices)
        # write_rgba(position_map_texture, world_pos, 1.0) # 1.0 as validity flag

# Stage 2: Populate Voxel Grid
def populate_voxel_grid(position_map_texture, volume_bounds, voxel_dim, bit_depth, volume_grid):
    """
    Converts the 2D position map into a 3D voxel grid.
    """
    disable_depth_test()
    disable_color_masking() # Allow direct writes to volume grid
    bind_volume(volume_grid)
    
    # Render full-screen quad
    render_fullscreen_quad()
    
    # Fragment shader logic for each screen pixel (u, v)
    def fragment_shader(u, v):
        # 1. Sample position map
        pos = sample_rgba(position_map_texture, u, v)
        
        # Check validity (e.g., alpha channel or depth)
        if not is_valid(pos):
            return

        # 2. Convert world position to voxel indices
        # Using the formula from the parent article
        x = int(voxel_dim * (pos.x - volume_bounds.x_min) / (volume_bounds.x_max - volume_bounds.x_min))
        y = int(voxel_dim * (pos.y - volume_bounds.y_min) / (volume_bounds.y_max - volume_bounds.y_min))
        z = int(voxel_dim * (pos.z - volume_bounds.z_min) / (volume_bounds.z_max - volume_bounds.z_min))
        
        # 3. Mark voxel as active
        # Implementation depends on bit_depth and storage format
        if bit_depth == 1:
            set_bit(volume_grid, x, y, z, 1)
        elif bit_depth == 8:
            set_byte(volume_grid, x, y, z, 1)
        else:
            # Handle other bit depths or multi-value storage
            pass

# Helper functions (abstracted)
def is_valid(pos):
    # Check if the sampled position is valid (e.g., alpha > 0)
    return pos.a > 0.0

def set_bit(grid, x, y, z, value):
    # Set the bit at index (x,y,z) in the bit-packed grid
    pass

def set_byte(grid, x, y, z, value):
    # Set the byte at index (x,y,z) in the grid
    pass
```

**Edge Cases and Pitfalls**
*   **Aliasing:** Aliasing effects exist in solid voxelization for closed polygonal models, appearing as prolonged lines where some voxels at the rasterization stage are missing [S1, unofficial].
*   **Bit-Operation Cost:** Bit-operations are more expensive than byte-operations. Using 1-bit depth for high resolutions ($512^3$) increases computational cost compared to 8-bit depth at lower resolutions [S1, unofficial].
*   **Memory Constraints:** For multivalued voxelization at high resolution, multiple worksheets may be required if video memory is insufficient [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The rasterization time depends mainly on the number of triangles. Texelization and synthesis are image-space operations dominated by volume resolution and bit-depth [S1, unofficial].
*   **Measured Performance:**
    *   On a PC with 2.4 GHz Pentium IV CPU, 2GB RAM, and ATI Radeon 9800 Pro:
        *   Blade model (1,765,388 triangles) at $256^3$ resolution: 75ms rasterization [S1, unofficial].
        *   Wagner model (60,246 triangles) at $512^3$ resolution (1-bit depth): 500ms [S1, unofficial].
        *   Wagner model at $256^3$ resolution (8-bit depth): 30ms [S1, unofficial].
        *   Wagner model at $128^3$ resolution (16-bit depth): 24ms [S1, unofficial].
        *   Wagner model at $64^3$ resolution (8-bit depth): 21ms [S1, unofficial].
        *   Implicit surface voxelization at $256^3$: 25ms [S1, unofficial].
    *   Visualization of the voxelized volume costs about 8ms on average, depending mainly on volume resolution [S1, unofficial].
*   **Scaling:** Performance is insensitive to scene complexity (triangle count) for the texelization/synthesis stages, but rasterization scales with triangle count [S1, unofficial].
*   **Strengths:** Real-time frame rates for collision detection and rendering; view-independent; simple and robust implementation [S1, unofficial].
*   **Limitations:** Aliasing artifacts in solid voxelization; bit-depth trade-offs between memory and performance; requires hardware support for 3D textures or multiple 2D worksheets [S1, unofficial].
*   **When to use:** Surgical planning, intra-operation navigation, radiation therapy, and applications requiring real-time collision detection with complex deformable models [S1, unofficial].
*   **When not to use:** When high-precision solid voxelization without aliasing is required without post-processing; when CPU-side sorting is acceptable and view-dependent methods are preferred.

## Background
*   **Origin:** The method was presented in "Real-time Voxelization for Complex Models" [S1, unofficial].
*   **History:** Implemented using Vertex/Pixel Shader 2.0 of Direct3D 9.0b [S1, unofficial].
*   **Usage:** Used in surgical planning, intra-operation navigation, and radiation therapy [S1, unofficial].

## Sources
[S1] Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
