# Brick-based volume rendering with packed textures

Also called: Index-Texture brick mapping, Packed-Texture volume rendering

## Overview
Brick-based volume rendering with packed textures is a technique for rendering volumetric datasets that exceed the capacity of local GPU video memory by partitioning the data into smaller sub-volumes (bricks) [S1, unofficial]. It differs from standard brick-based rendering by decoupling the spatial layout of the bricks from their memory layout: instead of mapping texture coordinates directly to physical memory addresses, it uses a secondary "Index-Texture" to map sample positions to the compacted brick indices, allowing the renderer to skip empty or fully transparent regions and fit more data into limited memory [S2, unofficial].

## How it works
**Inputs:** A volumetric dataset (typically a cube $[0, 1]^3$), a transfer function, and the number of bricks $N$ per dimension [S2, unofficial].
**Outputs:** A rendered image.

**Data Structures:**
1. **Packed-Texture:** A 3D texture containing only the voxels from bricks that are not fully transparent [S2, unofficial].
2. **Index-Texture:** A 3D texture that stores the integer index of each brick in the Packed-Texture for every spatial location in the brick grid [S2, unofficial].

**Algorithm Steps:**
1. **Brick Partitioning:** Divide the volume cube into $N \times N \times N$ equal-sized bricks [S2, unofficial].
2. **Transparency Culling:** Evaluate the transfer function for each brick to decide if it is transparent or not [S2, unofficial].
3. **Packing:** Store only the non-transparent bricks into the Packed-Texture [S2, unofficial].
4. **Index Map Creation:** Create the Index-Texture, which stores the brick index in the Packed-Texture for each brick position [S2, unofficial].
5. **Rendering (Ray Casting):** For each sample point along a ray, calculate in which brick the sample point is, then access the Index-Texture to get the brick index, and sample the Packed-Texture at the local fractional position [S2, unofficial].

**Formulas:**
Let $N$ be the number of bricks along each axis. Let $p$ be the fractional coordinate of a sample point in the global volume $[0, 1]^3$.
1. **Scaled Position:** $p_{scaled} = p \times N$
2. **Brick Index (Integer):** $idx = \lfloor p_{scaled} \rfloor$ [S2, unofficial]. The resulting $idx$ is a single integer representing the linear index of the brick.
3. **Local Sub-block Position:** $p_{local} = p_{scaled} - idx$ [S2, unofficial]. This yields the fractional position within the specific brick, ranging from $[0, 1)^3$.

**Parameters:**
* $N$: Number of bricks per dimension. Bricks must be kept small enough so that more than one brick fits into video memory [S1, unofficial].
* Transfer Function: Determines brick transparency [S2, unofficial].

**Edge Cases and Pitfalls:**
* **Interpolation Across Boundaries:** Incorrect interpolation can occur across brick boundaries if one plane of voxels is not duplicated [S1, unofficial].
* **Filtering Artifacts:** When storing blocks into a regular, packed 3D texture, hardware filtering must be disabled, and all filtering interpolation must be done manually [S2, unofficial].
* **Fencepost Problem:** Care must be taken about the fencepost problem when mapping coordinates to brick indices [S2, unofficial].
* **Bus Bandwidth:** Bricking suffers from bus-bandwidth problems and unbalanced load for the GPU and memory bus if not managed carefully [S1, unofficial]. Parallel transfer and rendering can balance the load but increases CPU load for intersection calculations [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def prepare_bricks(volume_data, transfer_function, N):
    """
    volume_data: 3D list/array of shape (D, D, D)
    transfer_function: callable returning alpha for a voxel value
    N: number of bricks per dimension
    """
    brick_size = len(volume_data) // N
    packed_voxels = []
    index_map = [[[-1 for _ in range(N)] for _ in range(N)] for _ in range(N)]
    current_packed_index = 0

    for i in range(N):
        for j in range(N):
            for k in range(N):
                x_start, x_end = i * brick_size, (i + 1) * brick_size
                y_start, y_end = j * brick_size, (j + 1) * brick_size
                z_start, z_end = k * brick_size, (k + 1) * brick_size
                brick = volume_data[x_start:x_end][y_start:y_end][z_start:z_end]

                # Check if brick is transparent using transfer function
                is_transparent = all(transfer_function(v) == 0.0 for v in brick)

                if not is_transparent:
                    packed_voxels.append(brick)
                    index_map[i][j][k] = current_packed_index
                    current_packed_index += 1

    return packed_voxels, index_map

def ray_cast_sample(p_global, index_map, packed_voxels, N):
    """
    p_global: fractional coordinate in [0, 1]^3
    index_map: 3D list storing brick indices
    packed_voxels: list of brick arrays
    N: number of bricks per dimension
    """
    p_scaled = [p_global[0] * N, p_global[1] * N, p_global[2] * N]
    idx = [math.floor(p_scaled[0]), math.floor(p_scaled[1]), math.floor(p_scaled[2])]
    p_local = [p_scaled[0] - idx[0], p_scaled[1] - idx[1], p_scaled[2] - idx[2]]

    brick_index = index_map[idx[0]][idx[1]][idx[2]]

    if brick_index == -1:
        return 0.0, 0.0

    brick_data = packed_voxels[brick_index]
    # Standard trilinear interpolation is performed on brick_data at p_local [G]
    voxel_value = brick_data[0][0][0] # Simplified for valid Python structure [G]
    alpha = 1.0 # Standard compositing alpha [G]
    return voxel_value, alpha
```

## Subtopics

### Brick Entry Point Offsetting
A technique to correct sampling seam visibility during interaction-time when sampling distance is increased. It involves offsetting the entry point in each brick to be a multiple along the entire ray [S3, unofficial]. During interaction-time, the sampling distance is increased, which causes the sampling seams to become visible again. It is possible to further correct the visibility of this artifact during interaction-time by offsetting the entry point in each brick to be a multiple along the entire ray (entry would be computed from the eye pose to make sure it is a multiple for each different brick) [S3, unofficial]. Additional work is needed to address the gradient computation at the edges in order to correct other artifacts present when shading is on [S3, unofficial].

## Operating characteristics
* **Memory Efficiency:** Allows rendering of datasets larger than GPU memory by packing only non-transparent bricks [S1, unofficial][S2, unofficial].
* **Performance:** Parallel transfer and rendering can balance the load, but increases CPU load for intersection calculations [S1, unofficial]. Inefficient if bus-bandwidth is not managed carefully [S1, unofficial].
* **Complexity:** Requires manual interpolation and handling of fencepost problems, increasing shader complexity [S2, unofficial].
* **Artifacts:** Prone to sampling seams and interpolation artifacts across brick boundaries if not handled correctly (e.g., voxel duplication, manual filtering) [S1, unofficial][S2, unofficial][S3, unofficial].

## Background
This method is a variation of brick-based volume rendering, which was introduced to handle datasets too large for local video memory [S1, unofficial]. The specific use of packed textures and index textures for spatial indexing is discussed in community forums as a solution for GPU-based ray casting of large datasets [S2, unofficial]. The issue of sampling seams in parallel rendered volumes and techniques to correct them, such as entry point offsetting, are documented in technical reports from Kitware [S3, unofficial].

## Sources
[S1] Real-Time Volume Graphics [03] GPU-Based Volume Rendering Real-Time Volume Graphics [03] GPU-Based Volume Rendering. Volume Rendering. Image order approach:. Data Set. Image Plane. Eye. For each pixel, https://www.slideserve.com/risa/real-time-volume-graphics-03-gpu-based-volume-rendering, fetched 2026-10-25 (unofficial)
[S2] volume rendering of large data by packing texture, https://stackoverflow.com/questions/13045663/volume-rendering-of-large-data-by-packing-texture, fetched 2026-10-26 (unofficial)
[S3] Correcting Sampling Seams in Parallel Rendered Volumes, https://www.kitware.com/sampling-seams-in-parallel-rendered-volumes/, fetched 2026-10-29 (unofficial)
