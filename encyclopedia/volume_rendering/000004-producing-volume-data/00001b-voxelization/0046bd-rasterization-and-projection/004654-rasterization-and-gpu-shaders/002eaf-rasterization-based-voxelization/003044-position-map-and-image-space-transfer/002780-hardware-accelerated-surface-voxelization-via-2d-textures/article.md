# Hardware-accelerated surface voxelization via 2D textures

Also called: Image-space voxelization

## Overview
This method transfers intersection computation from geometric space to image space by rasterizing model surfaces into 2D textures, enabling real-time collision detection and volumetric rendering [S1, unofficial]. It differs from slicing-based voxelization by rasterizing the scene approximately once rather than $N$ times (where $N$ is the volume resolution), making image-space performance insensitive to scene complexity [S1, unofficial]. Unlike Layered Depth Image (LDI) extraction, it is view-independent and avoids CPU-side sorting of depth lists [S1, unofficial].

## How it works
**Inputs:** Polygonal mesh (triangles), volume bounds $[x_{min}, x_{max}] \times [y_{min}, y_{max}] \times [z_{min}, z_{max}]$, volume resolution `voxelDim`, and bit-depth per voxel [G].
**Outputs:** A 3D voxel grid stored as 2D texture worksheets representing the surface or solid volume [S1, unofficial].

**Data Structures:**
1.  **Position Map:** A 2D texture where each texel stores the world-space position of the nearest surface fragment [G].
2.  **Voxel Storage:**
    *   For $512^3$ resolution, a voxel is represented using one bit to store the volume in one worksheet [S1, unofficial].
    *   For $128^3$ resolution, each vertex normal is encoded using 16 bits [S1, unofficial].
    *   Standard configuration uses 8 bits per voxel for $256^3$ resolution [S1, unofficial].

**Steps:**
1.  **Position Map Generation:** Follow the shared principles of [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md).
2.  **Volume Population:** Follow the shared principles of [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md).
3.  **Index Conversion:** Convert the sampled world position to voxel indices $(x, y, z)$ using the volume bounds and resolution [G]. The formula is:
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
4.  **Voxel Update:** Write the result to the voxel storage [G]. If performing solid voxelization, fill the interior [G]. If performing surface voxelization, mark only the surface voxels [S1, unofficial].
5.  **Collision Detection:** Carry out collision queries using the generated 2D textures in image space [S1, unofficial].
6.  **Rendering:** Render the volume data and voxelized slices orderly in 3D space to show complex deformable geometric models semi-transparently [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def clamp(val, min_val, max_val):
    return max(min_val, min(val, max_val))

def set_bit_in_worksheet(worksheet, bit_index, value):
    byte_idx = bit_index // 8
    bit_offset = bit_index % 8
    if value:
        worksheet[byte_idx] |= (1 << bit_offset)
    else:
        worksheet[byte_idx] &= ~(1 << bit_offset)

def get_bit_from_worksheet(worksheet, bit_index):
    byte_idx = bit_index // 8
    bit_offset = bit_index % 8
    return (worksheet[byte_idx] >> bit_offset) & 1

def hardware_accelerated_voxelization(mesh, bounds, voxel_dim, bit_depth):
    # Step 1 & 2: Position Map Generation & Volume Population
    # Follows [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md)
    position_map = rasterize_geometry_to_2d_texture(mesh)
    
    # Initialize voxel storage as 2D texture worksheets [S1, unofficial]
    # Total bits = voxel_dim^3. Worksheets store packed bits.
    total_bits = voxel_dim ** 3
    worksheets = [0] * ((total_bits + 7) // 8)
    
    # Step 3 & 4: Volume Population, Index Conversion & Voxel Update
    for y_screen in range(position_map.width):
        for x_screen in range(position_map.height):
            pos = sample_2d_texture(position_map, x_screen, y_screen)
            if pos is None:
                continue
                
            # Convert world position to voxel indices
            x = int(voxel_dim * (pos.x - bounds.x_min) / (bounds.x_max - bounds.x_min))
            y = int(voxel_dim * (pos.y - bounds.y_min) / (bounds.y_max - bounds.y_min))
            z = int(voxel_dim * (pos.z - bounds.z_min) / (bounds.z_max - bounds.z_min))
            
            # Clamp indices to valid range [0, voxelDim-1] [G]
            x = clamp(x, 0, voxel_dim - 1)
            y = clamp(y, 0, voxel_dim - 1)
            z = clamp(z, 0, voxel_dim - 1)
            
            # Calculate linear bit index for 3D voxel
            bit_index = x + y * voxel_dim + z * voxel_dim * voxel_dim
            
            # Write to voxel storage based on bit_depth [S1, unofficial]
            if bit_depth == 1:
                set_bit_in_worksheet(worksheets, bit_index, 1)
            elif bit_depth == 8:
                # For 8-bit depth, a separate byte array or texture is used per voxel
                # Simplified here as a direct byte write for clarity
                pass 
            elif bit_depth == 16:
                # For 16-bit depth, normals are encoded per vertex [S1, unofficial]
                pass
                
    return worksheets

def perform_collision_detection(worksheets, voxel_dim, query_pos, bounds):
    # Convert query position to voxel indices
    x = int(voxel_dim * (query_pos.x - bounds.x_min) / (bounds.x_max - bounds.x_min))
    y = int(voxel_dim * (query_pos.y - bounds.y_min) / (bounds.y_max - bounds.y_min))
    z = int(voxel_dim * (query_pos.z - bounds.z_min) / (bounds.z_max - bounds.z_min))
    x = clamp(x, 0, voxel_dim - 1)
    y = clamp(y, 0, voxel_dim - 1)
    z = clamp(z, 0, voxel_dim - 1)
    
    bit_index = x + y * voxel_dim + z * voxel_dim * voxel_dim
    is_occupied = get_bit_from_worksheet(worksheets, bit_index)
    return bool(is_occupied)
```

**Parameters:**
*   **Volume Resolution:** $256^3$ is identified as a good choice under current hardware conditions (ATI Radeon 9800 Pro) [S1, unofficial].
*   **Bit-depth:** 8 bits per voxel is used for the standard $256^3$ resolution configuration [S1, unofficial]. For $512^3$, 1 bit per voxel is used [S1, unofficial]. For $128^3$, 16 bits per vertex normal are used [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Aliasing:** Solid voxelization of closed polygonal models may exhibit aliasing effects, appearing as prolonged lines where some voxels are missing at the rasterization stage [S1, unofficial].
*   **Precision:** Insufficient precision in the position map leads to incorrect voxel indexing [G].
*   **Boundary Conditions:** Positions outside the volume bounds result in negative or out-of-bounds indices, requiring explicit clamping [G].

## Subtopics

### Position Map Voxelization
Uses the position map to populate voxels. Sets itself apart by decoupling rasterization from voxelization. Choose when high precision is needed. [Position Map Voxelization](2604-position-map-voxelization.md)

### Billboard Voxelization
Uses billboards to represent geometry. Sets itself apart by handling transparent or complex surfaces differently. Choose when geometry is not manifold. [Billboard Voxelization](2659-billboard-voxelization.md)

## Operating characteristics
**Time Complexity:** The algorithm traverses the model approximately once, making image-space performance insensitive to scene complexity [S1, unofficial]. Performance of the image-space operations depends entirely on volume resolution, while rasterization depends on triangle count [S1, unofficial].

**Memory Complexity:** Depends on volume resolution and bit-depth [S1, unofficial]. For $512^3$ with 1-bit depth, memory is 16MB [S1, unofficial]. For $256^3$ with 8-bit depth, memory is 16MB [S1, unofficial]. For $128^3$ with 16-bit depth, memory is 4MB [S1, unofficial]. For $64^3$ with 8-bit depth, memory is 256KB [S1, unofficial].

**Measured Performance:**
*   **Hardware:** PC with 2.4 GHz Pentium IV CPU, 2GB RAM, ATI Radeon 9800 Pro graphics card with 256MB RAM [S1, unofficial].
*   **Blade Model (1,765,388 triangles):** 75ms for rasterization at $256^3$ resolution [S1, unofficial]. Slicing-based voxelization for the same model at $256^3$ costs about 5 seconds (20.8ms × 256) [S1, unofficial].
*   **Wagner Model (60,246 triangles):**
    *   $512^3$ resolution, 1-bit depth: 500ms [S1, unofficial].
    *   $256^3$ resolution, 8-bit depth: 30ms [S1, unofficial].
    *   $128^3$ resolution, 16-bit depth: 24ms [S1, unofficial].
    *   $64^3$ resolution, 8-bit depth: 21ms [S1, unofficial].
*   **Implicit Surface ($x^4-5x^2+y^4-5y^2+z^4-5z^2+11.8=0$):** Both surface and solid voxelization take 25ms at $256^3$ resolution [S1, unofficial].
*   **Visualization:** Costs about 8ms on average, depending mainly on volume resolution [S1, unofficial].
*   **Hybrid Rendering:** Transparent cerebra model (36,758 triangles) hybrid volume rendering with MRI head volume ($128^3$) achieves about 10 fps [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Real-time frame rates for collision detection and volumetric rendering [S1, unofficial]. Simple, robust, and easy-to-implement [S1, unofficial].
*   **Limitations:** Aliasing effects in solid voxelization for closed polygonal models [S1, unofficial]. Bit-operations are more expensive than byte-operations [S1, unofficial].

**When to Use:**
*   Surgical planning, intra-operation navigation, and radiation therapy [S1, unofficial].
*   When real-time collision detection is required for complex and deformable models [S1, unofficial].
*   When scene complexity is high and slicing-based methods are too slow [S1, unofficial].

## Background
This technique was presented in "Real-time Voxelization for Complex Models" [S1, unofficial]. It was implemented using Vertex/Pixel Shader 2.0 of Direct3D 9.0b [S1, unofficial]. The method addresses the need for fast or real-time voxelization in interactive graphics applications [S1, unofficial]. It is particularly useful in fields such as surgical planning, intra-operation navigation, and radiation therapy [S1, unofficial].

## Sources
[S1] Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
