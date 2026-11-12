# Hardware-accelerated surface voxelization via 2D textures

Also called: Image-space voxelization

## Overview
This method transfers intersection computation from geometric space to image space by rasterizing model surfaces into 2D textures, enabling real-time collision detection and volumetric rendering [S1, unofficial]. It differs from slicing-based voxelization by rasterizing the scene approximately once rather than $N$ times (where $N$ is the volume resolution), making performance insensitive to scene complexity [S1, unofficial]. Unlike Layered Depth Image (LDI) extraction, it is view-independent and avoids CPU-side sorting of depth lists [S1, unofficial].

## How it works
**Inputs:** Polygonal mesh (triangles), volume bounds $[x_{min}, x_{max}] \times [y_{min}, y_{max}] \times [z_{min}, z_{max}]$, volume resolution `voxelDim`, and bit-depth per voxel.
**Outputs:** A 3D voxel grid (stored as 2D texture worksheets) representing the surface or solid volume.

**Data Structures:**
1.  **Position Map:** A 2D texture (`positionMap`) where each texel stores the world-space position of the nearest surface fragment. High-precision formats (e.g., `rgba32f`) are required to preserve world-space accuracy [S1, unofficial].
2.  **Voxel Storage:**
    *   For high resolutions (e.g., $512^3$), a voxel is represented using one bit to store the volume in one worksheet [S1, unofficial].
    *   For lower resolutions (e.g., $128^3$), each vertex normal is encoded using 16 bits [S1, unofficial].
    *   Standard configuration: 8 bits per voxel for $256^3$ resolution [S1, unofficial].

**Steps:**
1.  **Position Map Generation:** Rasterize the geometry into the `positionMap` 2D texture. Each fragment writes its world-space position to the corresponding texel. This step relies on the parent method's principles [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md).
2.  **Volume Population:** Render a full-screen quad. For each fragment, sample the `positionMap` to retrieve the world-space position $(pos.x, pos.y, pos.z)$.
3.  **Index Conversion:** Convert the sampled world position to voxel indices $(x, y, z)$ using the volume bounds and resolution. The formula is:
    $$
    x = \text{voxelDim} \times \frac{pos.x - x_{min}}{x_{max} - x_{min}}
    $$
    $$
    y = \text{voxelDim} \times \frac{pos.y - y_{min}}{y_{max} - y_{min}}
    $$
    $$
    z = \text{voxelDim} \times \frac{pos.z - z_{min}}{z_{max} - z_{min}}
    $$
    Standard integer truncation or floor is implied by the return type `ivec3` in typical GLSL implementations [G].
4.  **Voxel Update:** Write the result to the voxel storage. If performing solid voxelization, fill the interior. If performing surface voxelization, mark only the surface voxels.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Position Map Generation (Fragment Shader)
# Input: World-space position of the fragment
# Output: positionMap texture

def rasterize_to_position_map(world_pos: vec3):
    """
    Rasterize geometry into a 2D texture storing world-space positions.
    This is handled by the graphics pipeline's depth test.
    """
    # The hardware ensures only the nearest fragment writes to the texel.
    # Format: rgba32f to preserve precision [S1, unofficial].
    write_to_texture(positionMap, world_pos)

# Step 2 & 3: Volume Population and Index Conversion (Fragment Shader)
# Input: positionMap texture, volume bounds, voxelDim
# Output: Voxel grid (stored in 2D texture worksheets)

def populate_voxel_grid(positionMap: texture, 
                        bounds: tuple, 
                        voxelDim: int, 
                        bitDepth: int):
    """
    Convert position map samples to voxel indices and write to storage.
    """
    # Sample the position map
    pos = sample_texture(positionMap, screen_coords)
    
    # Extract world-space coordinates
    pos_x, pos_y, pos_z = pos.x, pos.y, pos.z
    
    # Unpack bounds
    x_min, x_max = bounds[0]
    y_min, y_max = bounds[1]
    z_min, z_max = bounds[2]
    
    # Calculate voxel indices
    # Note: Explicit clamping is required for positions outside bounds [G]
    x = int(floor(voxelDim * (pos_x - x_min) / (x_max - x_min)))
    y = int(floor(voxelDim * (pos_y - y_min) / (y_max - y_min)))
    z = int(floor(voxelDim * (pos_z - z_min) / (z_max - z_min)))
    
    # Clamp indices to valid range [0, voxelDim-1] [G]
    x = clamp(x, 0, voxelDim - 1)
    y = clamp(y, 0, voxelDim - 1)
    z = clamp(z, 0, voxelDim - 1)
    
    # Write to voxel storage
    # The storage layout depends on resolution and bitDepth [S1, unofficial]
    # For 512^3, use 1-bit per voxel in a worksheet [S1, unofficial]
    # For 128^3, use 16-bit per voxel (encoding normals) [S1, unofficial]
    # For 256^3, use 8-bit per voxel [S1, unofficial]
    write_to_voxel_storage(x, y, z, bitDepth)

# Step 4: Collision Detection (Optional)
def perform_collision_detection(voxelGrid: texture, query_pos: vec3):
    """
    Transfer intersection computation to image space.
    Query the voxel grid at the query position.
    """
    # Convert query position to voxel indices
    # ... (same conversion logic as above) ...
    # Check if the voxel is occupied
    is_occupied = read_from_voxel_storage(x, y, z)
    return is_occupied
```

**Parameters:**
*   **Volume Resolution:** $256^3$ is identified as a good choice under current hardware conditions (ATI Radeon 9800 Pro) [S1, unofficial].
*   **Bit-depth:** 8 bits per voxel is used for the standard $256^3$ resolution configuration [S1, unofficial]. For $512^3$, 1 bit per voxel is used [S1, unofficial]. For $128^3$, 16 bits per voxel are used [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Aliasing:** Solid voxelization of closed polygonal models may exhibit aliasing effects, appearing as prolonged lines where some voxels are missing at the rasterization stage [S1, unofficial].
*   **Precision:** Insufficient precision in the position map leads to incorrect voxel indexing [S1, unofficial].
*   **Boundary Conditions:** Positions outside the volume bounds result in negative or out-of-bounds indices. Explicit clamping is required [G].

## Subtopics

### Position Map Voxelization
Uses the position map to populate voxels. Sets itself apart by decoupling rasterization from voxelization. Choose when high precision is needed. [Position Map Voxelization](2604-position-map-voxelization.md)

### Billboard Voxelization
Uses billboards to represent geometry. Sets itself apart by handling transparent or complex surfaces differently. Choose when geometry is not manifold. [Billboard Voxelization](2659-billboard-voxelization.md)

## Operating characteristics
**Time Complexity:** The algorithm traverses the model approximately once, making performance insensitive to scene complexity [S1, unofficial]. Performance is dominated by volume resolution and bit-depth [S1, unofficial].

**Memory Complexity:** Depends on volume resolution and bit-depth. For $512^3$ with 1-bit depth, memory is 16MB [S1, unofficial]. For $256^3$ with 8-bit depth, memory is 16MB [S1, unofficial]. For $128^3$ with 16-bit depth, memory is 4MB [S1, unofficial]. For $64^3$ with 8-bit depth, memory is 256KB [S1, unofficial].

**Measured Performance:**
*   **Hardware:** PC with 2.4 GHz Pentium IV CPU, 2GB RAM, ATI Radeon 9800 Pro graphics card with 256MB RAM [S1, unofficial].
*   **Blade Model (1,765,388 triangles):** 75ms for rasterization at $256^3$ resolution [S1, unofficial]. Slicing-based voxelization for the same model at $256^3$ costs about 5 seconds (20.8ms x 256) [S1, unofficial].
*   **Wagner Model (60,246 triangles):**
    *   $512^3$ resolution, 1-bit depth: 500ms [S1, unofficial].
    *   $256^3$ resolution, 8-bit depth: 30ms [S1, unofficial].
    *   $128^3$ resolution, 16-bit depth: 24ms [S1, unofficial].
    *   $64^3$ resolution, 8-bit depth: 21ms [S1, unofficial].
*   **Implicit Surface ($x^4-5x^2+y^4-5y^2+z^4-5z^2+11.8=0$):** Both surface and solid voxelization take 25ms at $256^3$ resolution [S1, unofficial].
*   **Visualization:** Costs about 8ms on average, depending mainly on volume resolution [S1, unofficial].
*   **Hybrid Rendering:** Transparent cerebra model (36,758 triangles) hybrid volume rendering with MRI head volume ($128^3$) achieves about 10 fps [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Real-time frame rates for collision detection and volumetric rendering. Simple, robust, and easy-to-implement. Performance is dependent entirely on volume resolution, not scene complexity [S1, unofficial].
*   **Limitations:** Aliasing effects in solid voxelization for closed polygonal models. Bit-operations are more expensive than byte-operations [S1, unofficial].

**When to Use:**
*   Surgical planning, intra-operation navigation, and radiation therapy [S1, unofficial].
*   When real-time collision detection is required for complex and deformable models [S1, unofficial].
*   When scene complexity is high and slicing-based methods are too slow [S1, unofficial].

**When Not to Use:**
*   When high precision is required and hardware does not support floating-point blending [S1, unofficial].
*   When view-dependent rendering is preferred and LDI extraction is more suitable [S1, unofficial].

## Background
This technique was presented in "Real-time Voxelization for Complex Models" [S1, unofficial]. It was implemented using Vertex/Pixel Shader 2.0 of Direct3D 9.0b [S1, unofficial]. The method addresses the need for fast or real-time voxelization in interactive graphics applications [S1, unofficial]. It is particularly useful in fields such as surgical planning, intra-operation navigation, and radiation therapy [S1, unofficial].

## Sources
[S1] Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
