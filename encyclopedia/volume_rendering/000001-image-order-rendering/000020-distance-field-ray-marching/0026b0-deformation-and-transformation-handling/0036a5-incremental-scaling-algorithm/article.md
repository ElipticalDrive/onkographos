# Incremental Scaling Algorithm

Also called: Back Scaling

## Overview
The Incremental Scaling Algorithm is a method for applying uniform or non-uniform scaling transformations to voxel primitives by computing the new position of each voxel based on the already-transformed position of its immediate predecessor, using a maximum of three additions per coordinate instead of full matrix multiplication [S1]. It sets itself apart from direct matrix multiplication by exploiting spatial coherence between neighboring voxels, significantly reducing arithmetic operations at the cost of requiring all volume elements (including empty ones) to be processed, making it particularly suitable for parallel architectures [S1].

## How it works
**Inputs and outputs:**
*   Input: Source voxel grid defined by bounds $(X_{src}, Y_{src}, Z_{src})$ to $(X_{src}', Y_{src}', Z_{src}')$ and scaling factors $S_x, S_y, S_z$.
*   Output: Scaled voxel grid defined by bounds $(X_{dest}, Y_{dest}, Z_{dest})$ to $(X_{dest}', Y_{dest}', Z_{dest}')$.

**Data structures:**
*   Source and destination voxel arrays.
*   Floating-point or fixed-point accumulators for coordinates to maintain precision during incremental steps.

**Steps:**
1.  **Determine Bounds:** Calculate the destination bounds based on the source bounds and scaling factors. For forward scaling:
    $$ X_{dest} = X_{src} S_x, \quad Y_{dest} = Y_{src} S_y, \quad Z_{dest} = Z_{src} S_z $$
    $$ X_{dest}' = X_{src}' S_x, \quad Y_{dest}' = Y_{src}' S_y, \quad Z_{dest}' = Z_{src}' S_z $$
2.  **Select Strategy:** If any scaling factor $S_i > 1$, use **Back Scaling** to prevent holes. If all $S_i \le 1$, use **Forward Scaling**.
3.  **Forward Scaling (Downscaling/Equal Scaling):**
    *   Initialize destination coordinates $(X_d, Y_d, Z_d)$ to $(X_{dest}, Y_{dest}, Z_{dest})$.
    *   Iterate through the source grid. For each voxel at $(X_s, Y_s, Z_s)$:
        *   Map to destination using incremental updates. The coordinate of the next transformed voxel is obtained by adding the scaling factor to the predecessor's coordinate:
            $$ X_{next} = X_{prev} + S_x $$
            $$ Y_{next} = Y_{prev} + S_y $$
            $$ Z_{next} = Z_{prev} + S_z $$
        *   Round the resulting coordinates to integer grid positions.
        *   Write the source voxel value to the calculated destination position.
    *   Increment source coordinates by 1.
4.  **Back Scaling (Upscaling):**
    *   Transform two opposite corners of the source room to find two opposite corners of the destination room to define the destination bounds.
    *   Initialize destination coordinates $(X_d, Y_d, Z_d)$ to the starting corner of the destination room.
    *   Iterate through the destination grid. For each voxel at $(X_d, Y_d, Z_d)$:
        *   Reverse-transform the coordinates to determine the corresponding source voxel. This is done by multiplying the destination coordinates by the inverse scaling factors $(1/S_x, 1/S_y, 1/S_z)$.
        *   Alternatively, step through the source room in parallel with the destination room. The increments used in stepping through the source room are:
            $$ \Delta X_{src} = \frac{X_{incr}}{S_x}, \quad \Delta Y_{src} = \frac{Y_{incr}}{S_y}, \quad \Delta Z_{src} = \frac{Z_{incr}}{S_z} $$
            where $X_{incr}, Y_{incr}, Z_{incr}$ are the increments (typically 1) used in stepping through the destination room.
        *   Sample the source voxel at the computed (possibly fractional) source coordinates.
        *   Write the sampled value to the current destination position.
    *   Increment destination coordinates by 1.
5.  **Prevent Overwriting:** When scaling in-place or when source and destination overlap, choose the starting corner of the room based on the values of $S_x, S_y, S_z$ to ensure source voxels are not overwritten before they are read [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def incremental_scale(source_grid, dest_grid, src_bounds, dest_bounds, Sx, Sy, Sz):
    """
    source_grid: 3D array of voxel values
    dest_grid: 3D array for output
    src_bounds: (x0, y0, z0, x1, y1, z1)
    dest_bounds: (x0, y0, z0, x1, y1, z1)
    Sx, Sy, Sz: scaling factors
    """
    # Determine if back scaling is needed
    use_back_scaling = (Sx > 1) or (Sy > 1) or (Sz > 1)

    if use_back_scaling:
        # Back Scaling: Iterate destination, sample source
        # Initialize destination iterator
        xd, yd, zd = dest_bounds[0], dest_bounds[1], dest_bounds[2]
        
        # Calculate source start point corresponding to dest start
        # Note: Exact mapping depends on corner definitions. 
        # Assuming dest_bounds[0] corresponds to src_bounds[0] scaled.
        xs = xd / Sx
        ys = yd / Sy
        zs = zd / Sz
        
        # Increments for source grid
        dx_s = 1.0 / Sx
        dy_s = 1.0 / Sy
        dz_s = 1.0 / Sz
        
        while yd <= dest_bounds[5]:
            while xd <= dest_bounds[4]:
                while zd <= dest_bounds[3]:
                    # Sample source grid at (xs, ys, zs)
                    # Interpolation or nearest neighbor is required here [G]
                    val = sample_source(source_grid, xs, ys, zs)
                    dest_grid[int(xd), int(yd), int(zd)] = val
                    
                    zd += 1
                    xs += dx_s
                xd += 1
                ys += dy_s
                zs = zd_start # Reset Z source coordinate for new Y row? 
                              # Logic depends on memory layout. 
                              # Standard row-major: reset Z, increment Y.
                              # Here we assume simple iteration.
            yd += 1
            xs = xd_start / Sx # Reset X source coordinate
            ys += dy_s
            zs = zd_start / Sz
    else:
        # Forward Scaling: Iterate source, write destination
        xd, yd, zd = dest_bounds[0], dest_bounds[1], dest_bounds[2]
        
        xs, ys, zs = src_bounds[0], src_bounds[1], src_bounds[2]
        
        while xs <= src_bounds[3]:
            while ys <= src_bounds[4]:
                while zs <= src_bounds[5]:
                    dest_grid[int(xd), int(yd), int(zd)] = source_grid[int(xs), int(ys), int(zs)]
                    
                    zd += 1
                    xd += Sx
                ys += 1
                yd += Sy
                xd = dest_bounds[0] # Reset X for new Y row
            xs += 1
            xd = dest_bounds[0] # Reset X for new Y plane
            yd = dest_bounds[1] # Reset Y for new Z plane

    return dest_grid
```

**Parameters:**
*   $S_x, S_y, S_z$: Scaling factors. Typical values depend on the desired resolution change.
*   $X_{incr}, Y_{incr}, Z_{incr}$: Step increments, typically 1.

**Edge cases and pitfalls:**
*   **Holes in Upscaling:** If scaling factors are greater than 1, forward scaling may produce holes due to volume increase without a proportional increase in voxel count. This is averted by using back scaling [S1].
*   **Overwriting:** Precautions must be taken to avoid overwriting source voxels before they have been scaled. This is accomplished by properly choosing the corner of the room to be used as a starting point for the transformation based on the values of $S_x, S_y, S_z$ [S1].
*   **Precision:** Rounding results to integer coordinates is necessary for grid alignment. Floating-point accumulation errors may occur over large grids; fixed-point arithmetic or periodic re-initialization may be required [G].

## Subtopics

### [Incremental Geometric Transformation](10583-incremental-geometric-transformation.md)
Incremental Geometric Transformation is a method for applying affine transformations (translation, scaling, rotation) to voxel primitives by computing the new position of each voxel based on the already-transformed position of its immediate predecessor, using a maximum of three additions per coordinate instead of full matrix multiplication. It sets itself apart from direct matrix multiplication by exploiting spatial coherence between neighboring voxels, significantly reducing arithmetic operations at the cost of requiring all volume elements (including empty ones) to be processed, making it particularly suitable for parallel architectures.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the output grid. Each voxel requires a constant number of additions and a rounding operation, avoiding the 12-15 multiplications and additions required for full matrix-vector multiplication per voxel.
*   **Memory Complexity:** $O(N)$ for the output grid. No additional significant memory is required beyond coordinate accumulators.
*   **Strengths:** High computational efficiency due to reduced arithmetic operations. Well-suited for parallel execution as each voxel update is independent once the coordinate mapping is established.
*   **Limitations:** Requires processing all voxels, including empty ones, which can be inefficient for sparse volumes. Precision issues may arise with large scaling factors or large grids if not managed carefully.
*   **When to use:** Ideal for real-time voxel rendering on hardware with limited arithmetic throughput or when scaling is a frequent operation.
*   **When not to use:** For sparse volumes where traversal of empty space is expensive, or when non-uniform scaling with complex metric adjustments is required without post-processing.

## Background
The Incremental Scaling Algorithm was introduced in the context of the VoxBLIT Engine, a voxel frame buffer processor [S1]. It was developed to optimize the performance of voxel transformations by reducing the computational cost of scaling operations, which are common in voxel-based rendering pipelines. The method leverages the linear nature of scaling to update coordinates incrementally, a technique analogous to incremental translation algorithms used in the same engine [S1].

## Sources
[S1] The voxblt Engine: A Voxel Frame Buffer Processor, https://doi.org/10.2312/eggh/eggh88/085-102, fetched 2026-10-26
