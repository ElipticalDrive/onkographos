# Brick-based volume rendering with packed textures

Also called: Index-Texture brick mapping, Packed-Texture volume rendering

## Overview
Brick-based volume rendering with packed textures is a technique for rendering volumetric datasets that exceed the capacity of local GPU video memory by partitioning the data into smaller sub-volumes (bricks) and storing only the non-transparent bricks in a compact, packed 2D or 3D texture [S1, unofficial][S2, unofficial]. It differs from standard brick-based rendering by decoupling the spatial layout of the bricks from their memory layout: instead of mapping texture coordinates directly to physical memory addresses, it uses a secondary "Index-Texture" to map sample positions to the compacted brick indices, allowing the renderer to skip empty or fully transparent regions and fit more data into limited memory [S2, unofficial].

## How it works
**Inputs:** A volumetric dataset (typically a cube $[0, 1]^3$), a transfer function, and GPU memory constraints.
**Outputs:** A rendered image.

**Data Structures:**
1.  **Packed-Texture:** A 2D or 3D texture containing only the voxels from bricks that are not fully transparent. The bricks are stored contiguously or in a specific packing order.
2.  **Index-Texture:** A texture (typically 3D or 2D depending on the brick grid) that stores the integer index of the brick in the Packed-Texture for every spatial location. The resolution of the Index-Texture matches the resolution of the brick grid (e.g., if the volume is divided into $N \times N \times N$ bricks, the Index-Texture has dimensions $N \times N \times N$).

**Algorithm Steps:**

1.  **Brick Partitioning:** Divide the volume cube into $N \times N \times N$ equal-sized bricks [S2, unofficial].
2.  **Transparency Culling:** Evaluate the transfer function for each brick. If a brick is determined to be fully transparent (or empty), it is excluded from the Packed-Texture [S2, unofficial].
3.  **Packing:** Store the voxels of all non-transparent bricks into the Packed-Texture. The order of storage defines the brick indices (0, 1, 2, ...).
4.  **Index Map Creation:** Create the Index-Texture. For each brick position $(i, j, k)$ in the global grid, store the index of that brick in the Packed-Texture. If the brick is transparent/empty, store a sentinel value (e.g., -1) [S2, unofficial].
5.  **Rendering (Ray Casting):** For each sample point along a ray:
    a.  Calculate the global fractional coordinate $p \in [0, 1]^3$.
    b.  Determine the brick index and the local fractional position within that brick.
    c.  Sample the Index-Texture at the brick index to retrieve the packed brick index.
    d.  If the retrieved index is valid, sample the Packed-Texture at the local fractional position to get the voxel value.
    e.  If the retrieved index is invalid (transparent brick), skip sampling or treat as background.

**Formulas:**
Let $N$ be the number of bricks along each axis. Let $p$ be the fractional coordinate of a sample point in the global volume $[0, 1]^3$.

1.  **Scaled Position:**
    $$ p_{scaled} = p \times N $$
2.  **Brick Index (Integer):**
    $$ idx = \lfloor p_{scaled} \rfloor $$
    Where $\lfloor \cdot \rfloor$ is the floor function. The resulting $idx$ is a 3D vector $(i, j, k)$ representing the brick coordinates [S2, unofficial].
3.  **Local Sub-block Position:**
    $$ p_{local} = p_{scaled} - idx $$
    This yields the fractional position within the specific brick, ranging from $[0, 1)^3$ [S2, unofficial].

**Parameters:**
*   $N$: Number of bricks per dimension. The source suggests keeping bricks small enough so that more than one brick fits into video memory [S1, unofficial].
*   Transfer Function: Determines brick transparency.

**Edge Cases and Pitfalls:**
*   **Interpolation Across Boundaries:** Incorrect interpolation can occur across brick boundaries if one plane of voxels is not duplicated at the edges of each brick [S1, unofficial].
*   **Filtering Artifacts:** When storing blocks into a regular packed texture, hardware filtering must be disabled. The renderer must perform all filtering interpolation manually to avoid artifacts caused by the discontinuous memory layout [S2, unofficial].
*   **Fencepost Problem:** Care must be taken with boundary conditions when mapping coordinates to brick indices to avoid accessing out-of-bounds memory or incorrect bricks [S2, unofficial].
*   **Bus Bandwidth:** Bricking can suffer from bus-bandwidth problems and unbalanced load between the GPU and memory bus if not managed carefully [S1, unofficial]. Parallel transfer and rendering can help balance the load but increases CPU load for intersection calculations [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computation (CPU)
def prepare_bricks(volume_data, transfer_function, N):
    """
    volume_data: 3D array of shape (D, D, D)
    transfer_function: function that returns alpha for a voxel value
    N: number of bricks per dimension
    """
    brick_size = volume_data.shape[0] // N
    packed_voxels = []
    index_map = np.zeros((N, N, N), dtype=np.int32)
    
    current_packed_index = 0
    
    for i in range(N):
        for j in range(N):
            for k in range(N):
                # Extract brick
                x_start, x_end = i * brick_size, (i + 1) * brick_size
                y_start, y_end = j * brick_size, (j + 1) * brick_size
                z_start, z_end = k * brick_size, (k + 1) * brick_size
                
                brick = volume_data[x_start:x_end, y_start:y_end, z_start:z_end]
                
                # Check transparency
                # Note: In practice, this might involve checking max alpha or similar
                is_transparent = check_bricks_transparency(brick, transfer_function)
                
                if not is_transparent:
                    # Store brick data
                    # Flatten brick to 1D for packing, or keep 3D if texture supports it
                    # Assuming 3D texture or flattened 1D array in packed texture
                    packed_voxels.append(brick)
                    index_map[i, j, k] = current_packed_index
                    current_packed_index += 1
                else:
                    index_map[i, j, k] = -1 # Sentinel for transparent/empty
    
    # Create Packed Texture from packed_voxels
    # Create Index Texture from index_map
    return packed_voxels, index_map

def ray_cast_sample(p_global, index_texture, packed_texture, N):
    """
    p_global: fractional coordinate in [0, 1]^3
    index_texture: 3D texture storing brick indices
    packed_texture: texture storing packed brick data
    N: number of bricks per dimension
    """
    # 1. Calculate scaled position
    p_scaled = p_global * N
    
    # 2. Calculate brick index (integer)
    idx = floor(p_scaled)
    
    # 3. Calculate local position within brick
    p_local = p_scaled - idx
    
    # 4. Sample Index Texture to get packed brick index
    # Note: Index texture coordinates should be integer indices or normalized
    # Assuming index texture is sampled at integer coordinates or normalized [0,1]
    # If normalized: idx_norm = idx / N
    brick_index = index_texture.sample(idx) 
    
    if brick_index == -1:
        return 0.0, 0.0 # Transparent/Empty
    
    # 5. Sample Packed Texture
    # The packed texture contains the actual voxel data
    # We need to map p_local to the coordinates within the specific brick in the packed texture
    # This depends on how the packed texture is structured (e.g., 3D texture of bricks, or 1D array)
    # Assuming a 3D packed texture where each "voxel" is a brick, or a 1D array with stride
    # For simplicity, assuming packed_texture allows direct access via brick_index and p_local
    # In reality, this requires careful coordinate transformation based on packing layout
    
    voxel_value = packed_texture.sample(brick_index, p_local)
    
    return voxel_value, alpha
```

## Subtopics

### Brick Entry Point Offsetting
A technique to correct sampling seam visibility during interaction-time when sampling distance is increased. It involves offsetting the entry point in each brick to be a multiple along the entire ray [S3, unofficial]. During interaction-time, the sampling distance is increased, which causes the sampling seams to become visible again. It is possible to further correct the visibility of this artifact during interaction-time by offsetting the entry point in each brick to be a multiple along the entire ray (entry would be computed from the eye pose to make sure it is a multiple for each different brick) [S3, unofficial]. Additional work is needed to address the gradient computation at the edges in order to correct other artifacts present when shading is on [S3, unofficial].

## Operating characteristics
*   **Memory Efficiency:** Allows rendering of datasets larger than GPU memory by packing only non-transparent bricks [S1, unofficial][S2, unofficial].
*   **Performance:** Parallel transfer and rendering can balance the load, but increases CPU load for intersection calculations [S1, unofficial]. Inefficient if bus-bandwidth is not managed carefully [S1, unofficial].
*   **Complexity:** Requires manual interpolation and handling of fencepost problems, increasing shader complexity [S2, unofficial].
*   **Artifacts:** Prone to sampling seams and interpolation artifacts across brick boundaries if not handled correctly (e.g., voxel duplication, manual filtering) [S1, unofficial][S2, unofficial][S3, unofficial].

## Background
This method is a variation of brick-based volume rendering, which was introduced to handle datasets too large for local video memory [S1, unofficial]. The specific use of packed textures and index textures for spatial indexing is discussed in community forums as a solution for GPU-based ray casting of large datasets [S2, unofficial]. The issue of sampling seams in parallel rendered volumes and techniques to correct them, such as entry point offsetting, are documented in technical reports from Kitware [S3, unofficial].

## Sources
[S1] Real-Time Volume Graphics [03] GPU-Based Volume Rendering Real-Time Volume Graphics [03] GPU-Based Volume Rendering. Volume Rendering. Image order approach:. Data Set. Image Plane. Eye. For each pixel, https://www.slideserve.com/risa/real-time-volume-graphics-03-gpu-based-volume-rendering, fetched 2026-10-25 (unofficial)
[S2] volume rendering of large data by packing texture, https://stackoverflow.com/questions/13045663/volume-rendering-of-large-data-by-packing-texture, fetched 2026-10-26 (unofficial)
[S3] Correcting Sampling Seams in Parallel Rendered Volumes, https://www.kitware.com/sampling-seams-in-parallel-rendered-volumes/, fetched 2026-10-29 (unofficial)
