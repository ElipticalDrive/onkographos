# Object-Order Empty Space Skipping

Also called: Block min-max culling, Front-to-back ray-casting with bounding box culling, GPU-based Ray-Casting with Object-Order Empty Space Skipping

## Overview
Object-Order Empty Space Skipping is a GPU-based volume rendering optimization that reduces volume texture samples by skipping inactive regions using a secondary data structure of block min-max values [S1]. It differs from image-order methods (like [Space Leaping](5863-space-leaping.md)) by computing ray entry and exit bounds via rasterized bounding geometry of active blocks, allowing the fragment shader to skip large inactive segments without per-sample overhead [S1]. It builds on [Empty Space Skipping via Data Structures](5619-empty-space-skipping-via-data-structures.md) by subdividing the volume into regular blocks and using their min-max metadata to cull inactive regions against an isovalue threshold [S1].

## How it works
**Inputs:**
*   Volume texture $V$ containing scalar values (e.g., density) [S1].
*   Isovalue threshold $\tau$ defining active regions [S1].
*   Camera position $c$ and screen coordinates $(x,y)$ [S1].
*   Block size $B$ (e.g., $4^3$ or $8^3$ voxels) [S1].

**Outputs:**
*   Final image with rays skipping inactive regions [S1].

**Data Structures:**
1.  **Block Min-Max Structure:** A 3D array storing the minimum and maximum voxel values for each block in the volume [S1]. Let $M_{min}(i,j,k)$ and $M_{max}(i,j,k)$ be the min/max values for the block at grid index $(i,j,k)$ [S1].
2.  **Bounding Geometry:** A triangle mesh representing the faces of active blocks [S1]. A block is active if $M_{max} > \tau$ [S1]. The geometry consists of quads forming the boundary between active and inactive blocks [S2, unofficial].

**Steps:**

1.  **Initialization (Pre-computation):**
    *   Subdivide the volume into regular blocks of size $B \times B \times B$ [S1].
    *   Compute $M_{min}$ and $M_{max}$ for each block by scanning the volume texture [S1].
    *   Identify active blocks where $M_{max} > \tau$ [S1].
    *   Generate bounding geometry: For each pair of neighboring blocks, if one is active and the other is inactive, generate a quad representing the shared face [S2, unofficial].

2.  **Ray Start/Exit Depth Computation (Per Frame):**
    *   Compute ray direction $d(x,y)$ for each pixel from camera position $c$ and screen coordinates $(x,y)$:
        $$ p(t, x, y) = c + t \cdot d(x, y) $$
        [S1]
    *   **Rasterize Front Faces:** Render the front-facing polygons of the bounding geometry to an off-screen buffer using a depth test that keeps the closest intersection (e.g., `GL_LESS`) [S1]. This yields the ray start depth $t_{start}(x,y)$ [S1].
    *   **Rasterize Back Faces:** Render the back-facing polygons using a depth test that keeps the farthest intersection (e.g., `GL_GREATER`) [S1]. This yields the ray exit depth $t_{exit}(x,y)$ [S1].
    *   *Note:* If the volume does not fit in GPU memory, a second level of coarser bricks is maintained for caching, but the skipping logic remains similar [S1].

3.  **Ray Marching (Fragment Shader):**
    *   For each pixel, retrieve $t_{start}$ and $t_{exit}$ [S1].
    *   Initialize ray position $t = t_{start}$ [S1].
    *   **Loop:** While $t < t_{exit}$:
        *   Map current ray position $t$ to block coordinates $(i,j,k)$ [S1].
        *   Query the block min-max structure for the current block [S1].
        *   **Check Activity:** If $M_{max}(i,j,k) \le \tau$, the block is inactive.
            *   Skip to the next block boundary along the ray [S1].
            *   Update $t$ to the exit distance of the current block [S1].
        *   If $M_{max}(i,j,k) > \tau$, the block is active.
            *   Perform standard ray marching/sampling within the active block [S1].
            *   Update $t$ based on the sampling step size [S1].
    *   *Pitfall:* Inactive blocks enclosed by active blocks (e.g., a hollow shell) are not excluded by the bounding geometry. Samples in these blocks are skipped on a per-sample basis early in the ray-casting loop when the block min-max check fails [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_block_metadata(volume, block_size, threshold):
    """
    Computes max values for each block and prepares for bounding geometry generation.
    """
    dims = volume.shape
    num_blocks = [math.ceil(dims[i] / block_size) for i in range(3)]
    block_max = [[[float('-inf') for _ in range(num_blocks[2])] for _ in range(num_blocks[1])] for _ in range(num_blocks[0])]

    for z in range(num_blocks[2]):
        for y in range(num_blocks[1]):
            for x in range(num_blocks[0]):
                z0, z1 = z * block_size, min((z + 1) * block_size, dims[2])
                y0, y1 = y * block_size, min((y + 1) * block_size, dims[1])
                x0, x1 = x * block_size, min((x + 1) * block_size, dims[0])
                max_val = float('-inf')
                for vz in range(z0, z1):
                    for vy in range(y0, y1):
                        for vx in range(x0, x1):
                            if volume[vx, vy, vz] > max_val:
                                max_val = volume[vx, vy, vz]
                block_max[x][y][z] = max_val
                
    # Bounding geometry generation: iterate blocks, compare with neighbors, 
    # and emit quads where one is active and the other is inactive [S2, unofficial].
    return block_max

def fragment_shader(ray_origin, ray_direction, t_start, t_exit, block_max, block_size, threshold, sample_step):
    """
    Casts ray with object-order empty space skipping.
    """
    t = t_start
    color = [0.0, 0.0, 0.0]
    alpha = 0.0

    while t < t_exit:
        # Current voxel position
        pos = [ray_origin[i] + t * ray_direction[i] for i in range(3)]
        # Block indices
        bx = int(math.floor(pos[0] / block_size))
        by = int(math.floor(pos[1] / block_size))
        bz = int(math.floor(pos[2] / block_size))

        # Check bounds
        if bx < 0 or by < 0 or bz < 0 or \
           bx >= len(block_max) or by >= len(block_max[0]) or bz >= len(block_max[0][0]):
            break

        # Query block metadata
        max_val = block_max[bx][by][bz]

        if max_val <= threshold:
            # Block is inactive, skip to next block boundary
            t_exit_block = float('inf')
            for i in range(3):
                if ray_direction[i] != 0:
                    t_exit_block = min(t_exit_block, ((bx + 1) * block_size - pos[i]) / ray_direction[i])
                else:
                    t_exit_block = min(t_exit_block, float('inf'))
            t += t_exit_block
        else:
            # Block is active, perform sampling until block exit or t_exit
            t_block_exit = float('inf')
            for i in range(3):
                if ray_direction[i] != 0:
                    t_block_exit = min(t_block_exit, ((bx + 1) * block_size - pos[i]) / ray_direction[i])
                else:
                    t_block_exit = min(t_block_exit, float('inf'))

            while t < t_exit and t < t_block_exit:
                # Perform volume integration/sampling here
                # ... (standard ray casting logic)
                t += sample_step
    return color, alpha
```

## Subtopics

### [Early Shader Termination](969-early-shader-termination.md)
An optimization that discards complex decoding shaders for fragments likely to represent empty space using an early z-test in a two-pass approach. It sets itself apart by adding a lightweight pre-check to avoid expensive shader execution in empty regions. Choose it when fragment shaders are computationally heavy and empty space is frequent.

### [Rasterizing Bounding Geometry for Empty Space Skipping](5141-rasterizing-bounding-geometry-for-empty-space-skipping.md)
A technique that rasterizes bounding geometry to compute start and end positions for ray marching, skipping empty space by discarding fragments not at the minimum depth value. It sets itself apart by moving the major cost of empty space skipping into a rasterization stage that produces per-pixel lists of non-empty ray segments, making it agnostic of the underlying volume data structure. Choose it when hardware rasterization is available and tree traversal overhead is prohibitive.

### Overlapped Min-Max Block
A data structure for empty-space skipping in pre-integrated volume rendering [S3]. It modifies the region covered by each block so that each block covers some region of which thickness is at least the sampling distance [S3]. By overlapping the region of each block, a line segment formed by two sampling points can be tested for transparency using only one block, avoiding the overhead of determining which of multiple intersecting blocks are transparent [S3]. Unlike previous methods where a line segment may intersect at most three blocks and requires testing all intersecting blocks (creating overhead to determine cases like Fig. 1a vs 1b), this method tests only one block due to the overlapping region design [S3].

### Min-Max Block Culling
An object-order empty space skipping technique that uses min-max values of volume blocks to cull empty regions against the transfer function or isovalue [S4, unofficial]. The steps are: 1. Rasterize front and back faces of active (non-culled) blocks. 2. Start rays on the block front faces. 3. Terminate rays when full opacity is reached or the back face of the block is reached [S4, unofficial]. A key limitation is that this method does not skip all empty space, only the empty blocks that were culled during the rasterization setup phase [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels, with a significantly reduced constant factor due to skipping [S1]. Most empty space is skipped with no cost using modified ray segments $[t_{start}(x, y), t_{exit}(x, y)]$ [S1]. Only a small number of samples require skipping on a per-sample basis, which is accelerated via an adaptive sampling strategy [S1].
*   **Memory Complexity:** $O(V/B^3)$ for the block min-max structure, where $V$ is the total number of voxels and $B$ is the block size [S1].
*   **Strengths:** Efficient for large volumes with significant empty space. Works well with isosurface rendering. Uses a simple regular grid data structure compared to trees [S1].
*   **Limitations:** Inactive blocks enclosed by active blocks are not excluded by the bounding geometry and must be skipped per-sample, which can be inefficient for hollow objects [S1].
*   **When to use:** When rendering isosurfaces in large volumes with significant empty space and when hardware rasterization is available [S1].

## Background
Originated in the context of GPU-based ray-casting for real-time visualization of discrete isosurfaces [S1]. The method was introduced to overcome the limitations of earlier multi-pass approaches and to reduce the overhead of empty space skipping [S1]. It leverages the graphics pipeline's rasterization capabilities to compute ray bounds, integrating object-order and image-order skipping techniques [S1].

## Sources
[S1] Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces, https://cgl.ethz.ch/people/archive/siggc/publications/eg05.pdf, fetched 2026-10-26
[S2] implement volume rendering with object order empty space skipping, https://www.stackoverflow.com/questions/12216008, fetched 2026-10-27 (unofficial)
[S3] Efficient Interactive Pre-integrated Volume Rendering, https://doi.org/10.1007/11428862_118, fetched 2026-10-26
[S4] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
