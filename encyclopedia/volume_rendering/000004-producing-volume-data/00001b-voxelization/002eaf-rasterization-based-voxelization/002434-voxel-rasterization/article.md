# Voxel Rasterization

Also called: CUDA Voxel Rasterization

## Overview
Voxel Rasterization converts a triangle mesh into a voxelized representation by rendering the geometry into a 3D voxel buffer, supporting both simple occupancy and normal/color mapping [S2, S3, unofficial]. It differs from its sibling [3D Grid Rasterization](903-3d-grid-rasterization.md) by explicitly iterating over triangle surfaces to calculate interpolated points and normals rather than relying on a pure grid traversal, and from [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by utilizing a Direct3D 11-based rendering pipeline or CUDA kernels for the rasterization step [S2, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Triangle mesh (vertices, normals, indices), voxel grid dimensions ($V_x, V_y, V_z$), voxel size scaling factor ($mScale$), and accuracy multiplier ($accuracy$) [S2, S3, unofficial].
*   **Output:** A voxel buffer containing occupied voxel coordinates and optional color/normal data [S2, S3, unofficial].

**Data Structures:**
1.  **Voxel Buffer:** A bit-packed array for occupancy or an array of structs for color/normal data [S2, S3, unofficial].
    *   **Occupancy Layout:** A 1D integer array `voxel`. Index calculation for voxel $(px, py, pz)$:
        $$
        \begin{aligned}
        vx8 &= V_x / 8 \\
        ofs &= (px \gg 3) + py \cdot vx8 + pz \cdot vx8 \cdot V_y \\
        bit &= 1 \ll (px \& 7) \\
        voxel[ofs] \ |= \ bit
        \end{aligned}
        $$
        Boundary check: if $px > V_x - 1$, return [S3, unofficial].
    *   **Color/Normal Layout:** Struct containing `pos[3]` (int) for coordinates and `color[4]` (int) for RGBA values [S2, unofficial].
2.  **EffectParameter:** Passes world, view matrices (3), projection matrices, width, height, depth (floats), and bcount (int) to shaders [S2, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Prepare resource buffers and invoke the associated effect's rendering pass [S2, unofficial]. If using CUDA, create a block structure `TVdGenBlockCuda` with `TotalZoneCount` and populate `zoneData` with zone positions [S1, unofficial].
2.  **Triangle Processing:** For each triangle face $t$ with vertices $p_1, p_2, p_3$ and normals $n_1, n_2, n_3$:
    a.  Calculate edge vectors $d_1 = p_2 - p_1$, $d_2 = p_3 - p_1$ and normal differences $dn_1 = n_2 - n_1$, $dn_2 = n_3 - n_1$ [S3, unofficial].
    b.  Determine sampling count $c$ along the longest edge. Let $vm = \max(V_x, V_y, V_z)$.
        $$
        c = \max(\|d_1\| \cdot vm \cdot accuracy, \|d_2\| \cdot vm \cdot accuracy)
        $$
        Default $accuracy$ is 2, increased to 3 if $V_x < 1024$, and 4 if $V_x < 512$ [S3, unofficial].
    c.  **Outer Loop:** Iterate $i$ from 0 to $c$.
        i.  Calculate interpolation factor $a = i / c$.
        ii. Calculate points $x_1 = p_1 + d_1 \cdot a$ and $x_2 = p_1 + d_2 \cdot a$.
        iii. Calculate vector $dx = x_2 - x_1$.
        iv. Calculate inner loop count $d = \|dx\| \cdot vm \cdot accuracy$.
        v.  **Inner Loop:** Iterate $j$ from 0 to $d$.
            1.  Calculate factor $b = j / d$.
            2.  Calculate point $p = x_1 + dx \cdot b$.
            3.  Calculate interpolated normal $n = (n_1 + dn_1 \cdot a) + ( (n_1 + dn_2 \cdot a) - (n_1 + dn_1 \cdot a) ) \cdot b$.
            4.  **Voxel Write:**
                *   If `octree_render` is false:
                    *   If `colorizemap` is true, call `map_voxel(p, n)` [S3, unofficial].
                    *   Else, call `set_voxel(p)` [S3, unofficial].
                *   In `set_voxel`, if `voxel_col1` is active, clear the corresponding bit and set color bits:
                    $$
                    \begin{aligned}
                    voxel\_col1[ofs] \ &= \ 255 - bit \\
                    voxel\_col1[ofs] \ |= \ (color \& 1) \ll bit
                    \end{aligned}
                    $$
                    [S3, unofficial].
3.  **Finalization:** Copy results from device memory to host memory. For CUDA, copy `voxelData` and `materialData` using `copyDataUnsafe` with size $s = ZoneVoxelResolution^3$, and copy cache data using `copyCacheUnsafe` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_rasterize(triangles, grid_size, accuracy_multiplier=2.0):
    """
    Rasterize triangles into a voxel grid.
    :param triangles: List of tuples (p1, p2, p3, n1, n2, n3)
    :param grid_size: Tuple (Vx, Vy, Vz)
    :param accuracy_multiplier: Float for sampling density
    :return: Occupancy array (bit-packed) and optional color/normal data
    """
    Vx, Vy, Vz = grid_size
    vm = max(Vx, Vy, Vz)
    
    # Adjust accuracy based on grid size
    accuracy = accuracy_multiplier
    if Vx < 512:
        accuracy = 4.0
    elif Vx < 1024:
        accuracy = 3.0
        
    # Initialize occupancy buffer (bit-packed)
    # Size: (Vx/8) * Vy * Vz integers
    vx8 = Vx // 8
    voxel_count = vx8 * Vy * Vz
    voxel = [0] * voxel_count
    
    # Initialize color/normal buffers if needed (simplified as lists of structs)
    # For this implementation, we focus on occupancy as per the core bit-packing logic
    # In a full implementation, parallel structures for color/normal would be updated
    
    for p1, p2, p3, n1, n2, n3 in triangles:
        # Edge vectors and normal differences
        d1 = p2 - p1
        d2 = p3 - p1
        dn1 = n2 - n1
        dn2 = n3 - n1
        
        # Calculate sampling count along longest edge
        c1 = length(d1) * vm * accuracy
        c2 = length(d2) * vm * accuracy
        c = max(c1, c2)
        
        # Outer loop along the triangle surface
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
                n = (n1 + dn1 * a) + ((n1 + dn2 * a) - (n1 + dn1 * a)) * b
                
                # Map point to voxel coordinates
                px = int(p.x * Vx)
                py = int(p.y * Vy)
                pz = int(p.z * Vz)
                
                # Boundary checks
                if px < 0 or px >= Vx or py < 0 or py >= Vy or pz < 0 or pz >= Vz:
                    continue
                
                # Bit-packing for occupancy
                ofs = (px >> 3) + py * vx8 + pz * vx8 * Vy
                bit = 1 << (px & 7)
                voxel[ofs] |= bit
                
                # Note: Color/normal mapping logic (map_voxel) would update
                # additional buffers here if colorizemap is enabled.
                
    return voxel
```

## Subtopics

### [CUDA Voxelization](3491-cuda-voxelization.md)
A CUDA GPU implementation that launches a 3D grid of thread blocks to process voxels in parallel. It differs from the parent by utilizing a specific 3D block launch configuration and precomputed face bounds to optimize device-side computation.

### [Hybrid Voxelization Classification](3946-hybrid-voxelization-classification.md)
A GPU-accelerated method using a nested, two-level resolution structure (coarse Level1 and fine Level2 grids) for detailed normal estimation. It builds on the parent by extending output data structures to support hierarchical classification.

## Operating characteristics
*   **Time Complexity:** Proportional to the number of triangles multiplied by the sampling density along their edges, which scales with grid resolution ($V_x, V_y, V_z$) and accuracy [S3, unofficial].
*   **Memory Complexity:** $O(V_x V_y V_z / 8)$ for bit-packed occupancy, or $O(V_x V_y V_z)$ for full color/normal storage [S2, S3, unofficial].
*   **Performance:** GPU-accelerated versions (Direct3D 11, CUDA) offer significant speedups over CPU rasterization [S2, S4, unofficial].
*   **Strengths:** Supports direct integration with graphics pipelines; allows for normal and color interpolation [S3, unofficial].
*   **Limitations:** Sampling density must be carefully tuned to avoid gaps or excessive computation; bit-packing assumes $V_x$ is divisible by 8 or requires handling [S3, unofficial].

## Background
Voxel Rasterization is a technique used in voxel terrain generation and 3D modeling tools to convert mesh data into voxel grids. It leverages hardware-accelerated rendering (Direct3D 11) or parallel computing (CUDA) to efficiently populate voxel buffers [S2, S4, unofficial]. The method has been implemented in various open-source projects for voxel terrain editing and CAD model voxelization [S1, S2, S3, unofficial].

## Sources
[S1] bw2012/UE4VoxelTerrain, https://github.com/bw2012/UE4VoxelTerrain, fetched 2026-10-26 (unofficial)
[S2] nustxujun/AfterHumanDeclined, https://github.com/nustxujun/AfterHumanDeclined, fetched 2026-10-26 (unofficial)
[S3] sp4cerat/OpenGL-Voxel-Splatting, https://github.com/sp4cerat/OpenGL-Voxel-Splatting, fetched 2026-10-27 (unofficial)
[S4] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
