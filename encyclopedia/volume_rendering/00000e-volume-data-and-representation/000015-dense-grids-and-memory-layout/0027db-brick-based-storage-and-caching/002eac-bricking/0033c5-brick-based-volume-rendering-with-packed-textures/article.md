# Brick-based volume rendering with packed textures

Also called: Index-Texture brick mapping, Packed-Texture volume rendering

## Overview
Brick-based volume rendering with packed textures is a technique for rendering volumetric datasets that exceed the capacity of local GPU video memory by dividing the data into smaller sub-volumes (bricks) and packing only the non-transparent bricks into a dense 3D texture [S1, unofficial]. It distinguishes itself from standard bricking by using an auxiliary "Index-Texture" to map spatial coordinates to the compacted physical layout of the Packed-Texture, thereby reducing memory bandwidth and storage requirements compared to storing all bricks in a regular grid [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A large volumetric dataset (typically a cube from $[0, 1]^3$), a transfer function, and the number of bricks $N$ along each axis.
*   **Output:** A rendered image composed of ray-cast samples from the Packed-Texture, guided by the Index-Texture.

**Data Structures:**
1.  **Packed-Texture:** A 3D texture containing only the bricks determined to be non-transparent. The bricks are stored contiguously (packed) to minimize memory footprint.
2.  **Index-Texture:** A 3D texture of the same dimensions as the original volume (or the brick grid), where each voxel stores the integer index of the corresponding brick in the Packed-Texture.

**Algorithm Steps:**

1.  **Brick Division:** Divide the volume cube into $N \times N \times N$ equal-sized bricks [S2, unofficial].
2.  **Transparency Culling:** Evaluate the transfer function for each brick. If a brick is determined to be fully transparent, it is excluded from the Packed-Texture [S2, unofficial].
3.  **Texture Construction:**
    *   Store all non-transparent bricks into the Packed-Texture in a contiguous block [S2, unofficial].
    *   Populate the Index-Texture such that every voxel within a specific brick contains the index of that brick's position in the Packed-Texture [S2, unofficial].
4.  **Ray Casting (Shader Execution):**
    For each sample point along a ray:
    *   Calculate the global fractional coordinate $p_{global} \in [0, 1]^3$.
    *   Determine the brick index and local sub-block position using the following logic [S2, unofficial]:
        $$
        \begin{aligned}
        p_{scaled} &= p_{global} \times N \\
        brick\_idx &= \lfloor p_{scaled} \rfloor \\
        p_{local} &= p_{scaled} - brick\_idx
        \end{aligned}
        $$
    *   Access the Index-Texture at $p_{global}$ (or $brick\_idx$ depending on implementation granularity) to retrieve the physical index in the Packed-Texture [S2, unofficial].
    *   Sample the Packed-Texture using the retrieved index and $p_{local}$.

**Parameters:**
*   $N$: Number of bricks per dimension. Sources suggest keeping bricks small enough so that more than one fits in video memory [S1, unofficial].
*   Transfer Function: Determines brick transparency.

**Pitfalls and Edge Cases:**
*   **Interpolation Artifacts:** Incorrect interpolation can occur across brick boundaries if one plane of voxels is not duplicated at the edges of each brick [S1, unofficial].
*   **Filtering:** When storing blocks into a regular, packed 3D texture, hardware filtering must be disabled. The developer must handle the "fencepost problem" and perform all filtering/interpolation manually in the shader [S2, unofficial].
*   **Bus Bandwidth:** Bricking can suffer from unbalanced load for the GPU and memory bus, making it inefficient if not managed carefully [S1, unofficial].
*   **CPU Load:** Parallel transfer and rendering increase CPU load due to intersection calculations [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-processing (CPU)
def build_textures(volume_data, transfer_function, N):
    """
    volume_data: 3D array of size (V, V, V)
    transfer_function: function that returns alpha for a value
    N: integer, number of bricks per dimension
    """
    brick_size = V // N
    packed_bricks = []
    index_map = np.zeros((V, V, V), dtype=int)
    
    # Iterate over all bricks
    for x in range(N):
        for y in range(N):
            for z in range(N):
                # Extract brick data
                x_start, x_end = x * brick_size, (x + 1) * brick_size
                y_start, y_end = y * brick_size, (y + 1) * brick_size
                z_start, z_end = z * brick_size, (z + 1) * brick_size
                
                brick_data = volume_data[x_start:x_end, y_start:y_end, z_start:z_end]
                
                # Check transparency
                is_transparent = True
                for v in brick_data.flatten():
                    if transfer_function(v).alpha > 0:
                        is_transparent = False
                        break
                
                if not is_transparent:
                    brick_idx = len(packed_bricks)
                    packed_bricks.append(brick_data)
                    
                    # Fill index map for this brick
                    index_map[x_start:x_end, y_start:y_end, z_start:z_end] = brick_idx
                else:
                    # Mark as invalid or -1 in index map
                    index_map[x_start:x_end, y_start:y_end, z_start:z_end] = -1

    return packed_bricks, index_map

# Rendering (Shader)
def ray_cast_sample(global_pos, index_texture, packed_texture, N):
    """
    global_pos: float3, coordinate in [0, 1]^3
    index_texture: 3D texture storing brick indices
    packed_texture: 3D texture storing packed brick data
    N: integer, number of bricks per dimension
    """
    # 1. Calculate scaled position
    scaled_pos = global_pos * N
    
    # 2. Get brick index (integer part)
    brick_idx_vec = floor(scaled_pos)
    
    # 3. Get local position within brick (fractional part)
    local_pos = scaled_pos - brick_idx_vec
    
    # 4. Fetch brick index from Index-Texture
    # Note: In practice, one might sample the index texture at global_pos
    # or use the integer brick_idx to look up a pre-computed index array.
    # Based on [S2, unofficial], we access Index-Texture to get the value.
    packed_idx = index_texture.sample(global_pos)
    
    if packed_idx == -1:
        return 0.0 # Transparent
    
    # 5. Sample Packed-Texture
    # Since filtering is disabled, we must handle interpolation manually
    # or rely on nearest neighbor if no overlap/duplication is used.
    # However, [S1, unofficial] warns of incorrect interpolation across boundaries.
    # [S2, unofficial] says "do all filtering interpolation yourself".
    
    # Assuming nearest neighbor for simplicity as per "disable filtering"
    # A robust implementation would implement trilinear interpolation 
    # using the local_pos and handling boundary duplication.
    value = packed_texture.sample(packed_idx, local_pos)
    
    return value
```

## Subtopics

### Brick Entry Point Offsetting
A technique to correct sampling seam visibility during interaction-time when sampling distance is increased [S3, unofficial]. It involves offsetting the entry point in each brick to be a multiple along the entire ray [S3, unofficial]. The entry point is computed from the eye pose to ensure it is a multiple for each different brick [S3, unofficial]. A pitfall is that during interaction-time, the sampling distance is increased, which causes the sampling seams to become visible again if not corrected [S3, unofficial].

## Operating characteristics
*   **Memory Efficiency:** Reduces memory footprint by storing only non-transparent bricks [S2, unofficial].
*   **Performance:** Transfer and rendering can be performed in parallel to balance load, but this increases CPU load for intersection calculations [S1, unofficial].
*   **Limitations:** Inefficient if bus bandwidth is not managed carefully due to unbalanced load for GPU and memory bus [S1, unofficial]. Requires manual implementation of filtering and handling of fencepost problems [S2, unofficial].
*   **Complexity:** Increases implementation complexity due to the need for index mapping and manual interpolation [S2, unofficial].

## Background
Bricking is a general technique to handle volumetric data larger than GPU memory by dividing the volume into smaller sub-volumes (bricks) that fit into GPU memory [Parent]. The specific variant using packed textures and index mapping was discussed in community forums as a solution for GPU-based ray casting of large datasets [S2, unofficial]. The issue of sampling seams in parallel rendered volumes, which this method can exacerbate if not handled with boundary duplication, was analyzed in the context of ParaView and scientific computing [S3, unofficial].

## Sources
[S1] Real-Time Volume Graphics [03] GPU-Based Volume Rendering, https://www.slideserve.com/risa/real-time-volume-graphics-03-gpu-based-volume-rendering, fetched 2026-10-25 (unofficial)
[S2] volume rendering of large data by packing texture, https://stackoverflow.com/questions/13045663/volume-rendering-of-large-data-by-packing-texture, fetched 2026-10-26 (unofficial)
[S3] Correcting Sampling Seams in Parallel Rendered Volumes, https://www.kitware.com/sampling-seams-in-parallel-rendered-volumes/, fetched 2026-10-29 (unofficial)
