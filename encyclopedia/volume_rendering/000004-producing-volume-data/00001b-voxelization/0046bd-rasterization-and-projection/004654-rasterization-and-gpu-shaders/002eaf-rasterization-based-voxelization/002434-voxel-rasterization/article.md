# Voxel Rasterization

Also called: CUDA Voxel Rasterization, CUDA Volume Rasterization

## Overview
Voxel Rasterization converts a triangle mesh into a 3D voxel grid by iterating over triangle surfaces, sampling points and normals, and writing them into a voxel buffer [S6, unofficial]. It supports simple occupancy mode for binary marking and normal/color mapping mode for geometric attribute assignment [S6, unofficial]. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by explicitly sampling triangle edges rather than projecting the mesh into a 3D texture volume [S6, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by vertices $P$ and normals $N$; a voxel grid resolution $(v_x, v_y, v_z)$; an accuracy multiplier $a$; and a mode flag (occupancy vs. color/normal) [S6, unofficial].
*   **Output:** A voxel buffer containing occupancy bits and/or color/normal data for each voxel coordinate [S6, unofficial].

**Data Structures:**
*   **Voxel Buffer (Occupancy):** A bit-packed array `voxel` [S6, unofficial]. The index for a voxel at integer coordinates $(px, py, pz)$ is calculated as:
    $$ ofs = \lfloor px/8 \rfloor + py \cdot \lfloor v_x/8 \rfloor + pz \cdot \lfloor v_x/8 \rfloor \cdot v_y $$
    The bit within the byte is:
    $$ bit = 1 \ll (px \bmod 8) $$
    [S6, unofficial]
*   **Voxel Buffer (Color/Normal):** A struct containing `pos[3]` (int) for coordinates and `color[4]` (int) for RGBA values, or separate arrays for color bits if using bit-packing [S6, unofficial].

**Parameters:**
*   **accuracy:** A float multiplier for sampling density. Default is 2. It increases to 3 if $v_x < 1024$, and to 4 if $v_x < 512$ [S6, unofficial].
*   **mScale:** Voxel size scaling factor (default 1.0f) [S5, unofficial].
*   **mSize:** Dimensions of the voxel grid $(v_x, v_y, v_z)$ [S5, unofficial].

**Algorithm Steps:**
1.  **Preparation:** Extract vertices $p_1, p_2, p_3$ and normals $n_1, n_2, n_3$ for each triangle face $t$ [S6, unofficial].
2.  **Edge Calculation:** Compute edge vectors $d_1 = p_2 - p_1$, $d_2 = p_3 - p_1$ and normal differences $dn_1 = n_2 - n_1$, $dn_2 = n_3 - n_1$ [S6, unofficial].
3.  **Sampling Count:** Determine the number of samples along the longest edge to ensure uniform density [S6, unofficial]. Let $vm = \max(v_x, v_y, v_z)$ [S6, unofficial]. Calculate:
    $$ c = \text{length}(d_1) \cdot vm \cdot accuracy $$
    $$ c_2 = \text{length}(d_2) \cdot vm \cdot accuracy $$
    If $c_2 > c$, set $c = c_2$ [S6, unofficial].
4.  **Outer Loop (Edge 1 to 2):** Iterate $i$ from 0 to $c$ [S6, unofficial].
    *   Calculate interpolation factor $a = i / c$ [S6, unofficial].
    *   Calculate point $x_1 = p_1 + d_1 \cdot a$ and $x_2 = p_1 + d_2 \cdot a$ [S6, unofficial].
    *   Calculate vector $dx = x_2 - x_1$ [S6, unofficial].
    *   Calculate normal interpolation: $xn_1 = n_1 + dn_1 \cdot a$, $xn_2 = n_1 + dn_2 \cdot a$, $dnx = xn_2 - xn_1$ [S6, unofficial].
5.  **Inner Loop (Across Edge):** Calculate inner loop count $d = \text{length}(dx) \cdot vm \cdot accuracy$ [S6, unofficial]. Iterate $j$ from 0 to $d$ [S6, unofficial].
    *   Calculate factor $b = j / d$ [S6, unofficial].
    *   Calculate point $p = x_1 + dx \cdot b$ [S6, unofficial].
    *   Calculate normal $n = xn_1 + dnx \cdot b$ [S6, unofficial].
6.  **Voxel Write:**
    *   Convert $p$ to integer voxel coordinates $(px, py, pz)$ using $px = \lfloor p.x \cdot v_x \rfloor$, $py = \lfloor p.y \cdot v_y \rfloor$, $pz = \lfloor p.z \cdot v_z \rfloor$ [S6, unofficial].
    *   Perform boundary checks: if $px \ge v_x$ or $py \ge v_y$ or $pz \ge v_z$, skip [S6, unofficial].
    *   **Occupancy Mode:** Set the bit in the `voxel` array: `voxel[ofs] |= bit` [S6, unofficial].
    *   **Color/Normal Mode:** If `colorizemap` is true, call `map_voxel(p, n)`; otherwise call `set_voxel(p)` [S6, unofficial].
    *   **Bit-Packed Color:** If `voxel_col1` is active, clear the corresponding bit and set the color bits: `voxel_col1[ofs] &= (255 - bit); voxel_col1[ofs] |= (color & 1) << bit` [S6, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def length(v):
    return math.sqrt(sum(c**2 for c in v))

def voxel_rasterize(triangles, grid_size, accuracy_base=2.0):
    vx, vy, vz = grid_size
    vm = max(vx, vy, vz)
    
    # Adjust accuracy based on resolution
    accuracy = accuracy_base
    if vx < 1024:
        accuracy = 3.0
    if vx < 512:
        accuracy = 4.0

    # Initialize bit-packed occupancy buffer
    # Size: ceil(vx/8) * vy * vz
    voxel_buffer = [0] * ((vx + 7) // 8 * vy * vz)
    
    for p1, p2, p3 in triangles:
        d1 = p2 - p1
        d2 = p3 - p1
        
        # Calculate sampling counts
        c1 = length(d1) * vm * accuracy
        c2 = length(d2) * vm * accuracy
        c = max(c1, c2)
        
        # Outer loop along the "base" of the triangle span
        for i in range(int(c) + 1):
            a = i / c
            x1 = p1 + d1 * a
            x2 = p1 + d2 * a
            dx = x2 - x1
            
            # Inner loop across the triangle width
            d = length(dx) * vm * accuracy
            for j in range(int(d) + 1):
                b = j / d
                p = x1 + dx * b
                
                # Convert to voxel coordinates
                px = int(p.x * vx)
                py = int(p.y * vy)
                pz = int(p.z * vz)
                
                # Boundary checks
                if px < 0 or px >= vx or py < 0 or py >= vy or pz < 0 or pz >= vz:
                    continue
                
                # Bit packing index
                vx8 = vx // 8
                ofs = (px >> 3) + py * vx8 + pz * vx8 * vy
                bit = 1 << (px & 7)
                
                # Set occupancy bit
                voxel_buffer[ofs] |= bit
                
    return voxel_buffer
```

## Subtopics

### [CUDA Voxelization](3491-cuda-voxelization.md)
A CUDA GPU implementation that launches a 3D grid of thread blocks to process voxels in parallel. It differs from the parent method by utilizing a specific 3D block launch configuration and precomputed face bounds to optimize device-side computation.

### [Hybrid Voxelization Classification](3946-hybrid-voxelization-classification.md)
A GPU-accelerated method for voxelizing 3D CAD models using a nested two-level resolution structure. It distinguishes itself by maintaining a coarse Level1 grid and a finer Level2 grid for voxels containing surface geometry, enabling detailed normal estimation while managing memory. It builds on Voxel Rasterization by extending the output data structures to support hierarchical classification and normal accumulation.

### [CUDA Rasterizer](10772-cuda-rasterizer.md)
A custom implementation of forward and backward CUDA kernels for rasterizing triangle meshes into 3D voxel grids and computing gradients for vertex positions. It supports both full-image rendering and batched pixel-query modes, enabling differentiable rendering. It is distinguished by its explicit use of atomic operations for gradient accumulation and specific grid/block launch configurations optimized for hardware, rather than relying on standard graphics APIs or sparse voxel structures.

### [CUDA Sparse Voxel Rasterization Wrapper](10783-cuda-sparse-voxel-rasterization-wrapper.md)
A differentiable PyTorch interface for rasterizing sparse voxel representations into 2D image space. It distinguishes itself by supporting simultaneous outputs for color, depth, surface normals, and distortion, which facilitates advanced regularization and reconstruction tasks.

### [Direct Voxel Rasterization with Edge Fading](13471-direct-voxel-rasterization-with-edge-fading.md)
Renders 3D spheres into a voxel grid by iterating over axis-aligned bounding boxes, computing Euclidean distance to the sphere center, applying a linear edge fade, and blending colors using maximum blending. It differs from standard binary voxelization by introducing anti-aliased edges via intensity falloff, and from ray-marching volume rendering by using a direct, iterative rasterization approach that is computationally lightweight and suitable for real-time updates.

### [WebGL Voxelization via Triangle Splitting and Instancing](15405-webgl-voxelization-via-triangle-splitting-and-instancing.md)
A method to calculate the distance field of a mesh in real-time on WebGL by converting mesh triangles into small points (particles) that fit within voxels, using instancing to handle adjacent voxels and depth testing to simulate atomic minimum-distance updates. It sets itself apart by avoiding compute shaders and atomic operations, which are unavailable in WebGL, by instead leveraging the rasterization pipeline, instancing, and depth buffering.

### Raster Preprocessing
A preprocessing step that calculates the number of duplicates (likely for binning or occupancy) and generates a geometry buffer for rasterization [S8, unofficial]. It calls `_C.rasterize_preprocess` with image dimensions, camera parameters (tanfovx, tanfovy, cx, cy, w2c_matrix, c2w_matrix, near_plane), and voxel data (octree_paths, vox_centers, vox_lengths) [S8, unofficial]. It can return both the duplicate count and the geometry buffer, or only the count [S8, unofficial].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of triangles multiplied by the sampling density, which scales with grid resolution and the accuracy parameter [S6, unofficial].
*   **Memory Complexity:** Dominated by the voxel buffer size, which is $O(v_x \cdot v_y \cdot v_z)$ for dense grids [S6, unofficial]. Bit-packing reduces occupancy storage by a factor of 8 [S6, unofficial].
*   **Strengths:** Simple to implement; supports direct normal and color assignment; efficient for moderate resolutions [S6, unofficial].
*   **Limitations:** Sampling density must be carefully tuned to avoid gaps or excessive overlap; boundary handling requires explicit checks [S6, unofficial].

## Background
Voxel Rasterization is a fundamental technique in voxel-based rendering and modeling, often used as a preprocessing step for volume rendering or voxel-based physics [S5, unofficial]. The method leverages the simplicity of rasterization principles extended into 3D space [S5, unofficial]. Origin and development history are not given in the sources.

## Sources
[S1] Ruyi-Zha/r2_gaussian, https://github.com/Ruyi-Zha/r2_gaussian, fetched 2026-10-26 (unofficial)
[S2] yuyouxixi/x2-gaussian, https://github.com/yuyouxixi/x2-gaussian, fetched 2026-10-26 (unofficial)
[S3] yulunwu0108/Perturbed-Gaussian-Ensemble, https://github.com/yulunwu0108/Perturbed-Gaussian-Ensemble, fetched 2026-10-26 (unofficial)
[S4] bw2012/UE4VoxelTerrain, https://github.com/bw2012/UE4VoxelTerrain, fetched 2026-10-26 (unofficial)
[S5] nustxujun/AfterHumanDeclined, https://github.com/nustxujun/AfterHumanDeclined, fetched 2026-10-26 (unofficial)
[S6] sp4cerat/OpenGL-Voxel-Splatting, https://github.com/sp4cerat/OpenGL-Voxel-Splatting, fetched 2026-10-27 (unofficial)
[S7] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S8] leonhardrobin/svraster-win10, https://github.com/leonhardrobin/svraster-win10, fetched 2026-10-27 (unofficial)
