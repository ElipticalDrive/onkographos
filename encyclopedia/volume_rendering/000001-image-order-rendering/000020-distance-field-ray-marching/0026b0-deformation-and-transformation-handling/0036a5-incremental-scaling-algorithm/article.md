# Incremental Scaling Algorithm

Also called: Back Scaling

## Overview
The Incremental Scaling Algorithm applies uniform or non-uniform scaling transformations to voxel grids by computing destination coordinates through iterative addition rather than full matrix multiplication [S1]. It solves the computational bottleneck of per-voxel scaling in large volumes by exploiting spatial coherence between neighboring voxels, reducing the arithmetic cost of coordinate transformation to simple additions [S1].

## How it works
**Inputs and Outputs:**
- Input: Source voxel grid defined by bounds $(X_{src}, Y_{src}, Z_{src})$ to $(X_{src\_end}, Y_{src\_end}, Z_{src\_end})$ [S1].
- Input: Scaling factors $S_x, S_y, S_z$ [S1].
- Input: Destination grid bounds $(X_s, Y_s, Z_s)$ to $(X_E, Y_E, Z_E)$ [S1].
- Output: Destination voxel grid with scaled geometry [S1].

**Data Structures:**
- The algorithm operates on integer voxel coordinates [S1].
- Intermediate floating-point coordinates are used for the incremental stepping to maintain precision before rounding to the nearest integer voxel index [S1].

**Method:**
The standard scaling relationship maps a source coordinate to a destination coordinate:
$$ X_{dest} = X_{src} S_x $$
$$ Y_{dest} = Y_{src} S_y $$
$$ Z_{dest} = Z_{src} S_z $$
Direct computation requires a multiplication for every voxel [S1]. The incremental algorithm derives the next coordinate from the previous one:
$$ X_{dest} + S_x = (X_{src} + 1) S_x = X_{src} S_x + S_x $$
$$ Y_{dest} + S_y = (Y_{src} + 1) S_y = Y_{src} S_y + S_y $$
$$ Z_{dest} + S_z = (Z_{src} + 1) S_z = Z_{src} S_z + S_z $$
Thus, the next transformed coordinate is obtained by adding the respective scaling factor to the predecessor's coordinate and rounding the results to integer coordinates [S1].

**Pitfalls and Edge Cases:**
1. **Holes in Forward Scaling:** If any scaling factor is greater than 1, the destination volume expands without a proportional increase in voxels, producing holes in the scaled primitive [S1].
2. **Back Scaling:** To avert holes, the algorithm reverses the transformation process [S1]. It transforms two opposite corners of the source room to find two opposite corners in the destination room, then steps through the destination room [S1]. At each destination point, coordinates are reverse-transformed by multiplying by $(1/S_x, 1/S_y, 1/S_z)$ to determine the source voxel value [S1].
3. **Parallel Stepping:** A more efficient back-scaling variant steps through the source and destination rooms in parallel [S1]. The increments for stepping through the source room are $(X_{incr}/S_x, Y_{incr}/S_y, Z_{incr}/S_z)$, where $X_{incr}, Y_{incr}, Z_{incr}$ are the steps used in the destination room [S1].
4. **Overwriting:** If the source and destination grids share memory, voxels may be overwritten before they are read [S1]. Precautions are taken by properly choosing the starting corner of the transformation loop based on the signs of $S_x, S_y, S_z$ [S1].

**Pseudocode:**
The following implements back-scaling to prevent holes when $S > 1$. It assumes the destination bounds are known and the source bounds are derived or known. The direct back-mapping is mathematically equivalent to the parallel stepping variant where the destination steps by 1 and the source steps by $1/S$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def incremental_scaling(source_grid, dest_bounds, scale_factors):
    """
    source_grid: 3D array of voxel values indexed as [z][y][x]
    dest_bounds: tuple (Xs, Ys, Zs, XE, YE, ZE) defining destination voxel range
    scale_factors: tuple (Sx, Sy, Sz)
    """
    Sx, Sy, Sz = scale_factors
    Xs, Ys, Zs, XE, YE, ZE = dest_bounds
    inv_Sx, inv_Sy, inv_Sz = 1.0 / Sx, 1.0 / Sy, 1.0 / Sz
    
    # Initialize destination grid with background value 0
    dest_width = XE - Xs + 1
    dest_depth = ZE - Zs + 1
    dest_height = YE - Ys + 1
    dest_grid = [[[0 for _ in range(dest_width)] for _ in range(dest_depth)] for _ in range(dest_height)]
    
    src_width = len(source_grid[0][0])
    src_depth = len(source_grid[0])
    src_height = len(source_grid)
    
    # Choose starting corner based on scale factors to avoid overwriting
    # (Exact traversal direction logic depends on Sx, Sy, Sz signs [S1])
    
    for y in range(Ys, YE + 1):
        for z in range(Zs, ZE + 1):
            for x in range(Xs, XE + 1):
                # Reverse transform: map destination integer coordinate to source space
                src_x = x * inv_Sx
                src_y = y * inv_Sy
                src_z = z * inv_Sz
                
                # Round to nearest integer voxel index in source
                src_ix = int(src_x + 0.5) if src_x >= 0 else int(src_x - 0.5)
                src_iy = int(src_y + 0.5) if src_y >= 0 else int(src_y - 0.5)
                src_iz = int(src_z + 0.5) if src_z >= 0 else int(src_z - 0.5)
                
                # Check source bounds to avoid out-of-bounds access
                if 0 <= src_ix < src_width and 0 <= src_iy < src_depth and 0 <= src_iz < src_height:
                    dest_grid[y - Ys][z - Zs][x - Xs] = source_grid[src_iz][src_iy][src_ix]
                # Else leaves as 0 (background)
    return dest_grid
```

## Subtopics

### [Incremental Geometric Transformation](10583-incremental-geometric-transformation.md)
Applies affine transformations (translation, scaling, rotation) by computing new positions from the immediate predecessor's position using a maximum of three additions per coordinate. It reduces arithmetic operations compared to matrix multiplication but requires processing all voxels, including empty ones, making it suitable for parallel architectures.

### [Volume Deformation via Scattered Data Interpolation](16707-volume-deformation-via-scattered-data-interpolation.md)
Deforms volumes using 2D user-guided control points and Radial Basis Functions (RBFs) to generate a 3D displacement map. It differs by relying on global smooth interpolation from sparse handles rather than local voxel-wise transformations.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the output grid [S1]. The cost per voxel is reduced to a few additions and a rounding operation, compared to multiplications in standard scaling [S1].
- **Memory Complexity:** $O(N)$ for the output grid [S1]. In-place modification is possible if traversal order is managed correctly to prevent overwriting [S1].
- **Strengths:** Significantly reduces computational complexity for large rooms/volumes by eliminating multiplications [S1].
- **Limitations:** Requires careful handling of scaling factors $>1$ (back scaling) to avoid holes [S1]. Requires processing all voxels in the destination volume, even if empty, unlike sparse voxel approaches [S1].
- **When to use:** When scaling voxel primitives in a voxel frame buffer processor and computational efficiency is critical [S1].

## Background
The algorithm was developed to handle large rooms efficiently by reducing the complexity of geometric transformations [S1].

## Sources
[S1] The voxblt Engine: A Voxel Frame Buffer Processor, https://doi.org/10.2312/eggh/eggh88/085-102, fetched 2026-10-26
