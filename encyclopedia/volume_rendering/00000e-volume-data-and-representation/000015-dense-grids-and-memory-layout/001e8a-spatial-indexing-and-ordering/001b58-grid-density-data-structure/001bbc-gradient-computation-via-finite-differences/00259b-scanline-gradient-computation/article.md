# Scanline Gradient Computation
Also called: VolPack Scanline Normals Computation, vpScanlineNormals

## Overview
Scanline Gradient Computation is a memory-efficient method for computing 3D gradient vectors and surface normals for scalar volume data by processing one scanline at a time [S1]. It solves the problem of high memory bandwidth and storage requirements when computing derivatives for large volumes by maintaining only a sliding window of three slices in memory, making it suitable for volumes that exceed available RAM [S1]. It sets itself apart from batch processing methods like [VolPack Volume Normals Computation](3126-volpack-volume-normals-computation.md) by streaming data slice-by-slice rather than requiring the entire scalar array in memory [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume. The method requires the current scanline, the scanline immediately preceding it in the Y dimension, the scanline immediately following it in the Y dimension, the scanline immediately preceding it in the Z dimension, and the scanline immediately following it in the Z dimension [S1].
*   **Output:** For each voxel in the current scanline, the method computes and stores the gradient magnitude, an encoded surface normal vector, and optionally the original scalar value if the corresponding field index is provided [S1].

**Data Structures:**
The implementation relies on a `Voxel` structure defined by the VolPack library, which contains fields for scalars, gradients, and normals [S1]. The exact layout and byte strides for X, Y, and Z dimensions are defined by `vpSetRawVoxels` [S1].
*   `scalars`: Pointer to the current scanline of input scalar data (8-bit unsigned integers) [S1].
*   `scalars_minus_y`: Pointer to the previous Y-slice scanline [S1].
*   `scalars_plus_y`: Pointer to the next Y-slice scanline [S1].
*   `scalars_minus_z`: Pointer to the previous Z-slice scanline [S1].
*   `scalars_plus_z`: Pointer to the next Z-slice scanline [S1].
*   `voxel_scan`: Pointer to the output voxel scanline where results are written [S1].

**Algorithm Steps:**
1.  Initialize the volume context and set the raw voxel layout using `vpSetRawVoxels` to define strides [S1].
2.  Load scalar data into a buffer. If the volume is too large for a single array, load data slice-by-slice [S1].
3.  For each scanline in the volume (typically iterating through Z and Y, then X):
    a.  Ensure the five required scanlines (current, $\pm Y$, $\pm Z$) are loaded into memory [S1]. This requires keeping at most three slices of the volume in memory at any time [S1].
    b.  Call the core computation routine `vpScanlineNormals` for the current scanline [S1].
    c.  The routine computes the gradient using immediately adjacent neighbors in the Y and Z dimensions [S1].
    d.  Store the computed gradient magnitude and encoded normal into the `voxel_scan` output buffer at the specified field indices [S1].
4.  Repeat until all scanlines are processed [S1].

**Formulas:**
The sources describe the use of "immediately-adjacent neighbors" in the Y and Z dimensions to compute the gradient [S1]. For a voxel at $(x,y,z)$, the gradient components are computed from the Y and Z neighbors using standard central differences [G]. The gradient vector $\mathbf{g}$ is then $\left[ \frac{\partial S}{\partial x}, \frac{\partial S}{\partial y}, \frac{\partial S}{\partial z} \right]$ [G]. The normal is derived from the normalized gradient [G]. The specific encoding of the normal vector into the `normal_field` is handled internally by the VolPack library [S1].

**Parameters:**
*   `size`: The length in bytes of one scanline of scalar data (equal to the X dimension of the volume) [S1].
*   `scalar_field`, `gradient_field`, `normal_field`: Integer indices specifying which fields within the `Voxel` structure to populate [S1]. Use `VP_SKIP_FIELD` if a specific field should not be stored [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The method requires neighbors in $\pm Y$ and $\pm Z$ [S1]. At the boundaries of the volume, these neighbors do not exist [S1]. The VolPack documentation does not specify the boundary handling strategy (e.g., clamping, reflection, or zero-padding) in the provided text [S1]. Implementers must check the VolPack source or assume a specific behavior (often clamping or treating out-of-bounds as the boundary value) [G].
*   **Memory Management:** The method requires careful management of the sliding window of slices [S1]. Failing to update the pointers for `scalars_minus_y`, etc., correctly will result in incorrect gradients [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_scanline_normals(volume_dims, scalar_slices, voxel_fields):
    """
    Processes a 3D volume slice-by-slice using the scanline approach.
    volume_dims: (X, Y, Z)
    scalar_slices: list of 2D arrays [Z][Y][X]
    voxel_fields: (scalar_field, gradient_field, normal_field)
    """
    X, Y, Z = volume_dims
    gradients = [[[0.0]*X for _ in range(Y)] for _ in range(Z)]
    normals = [[[0.0]*X for _ in range(Y)] for _ in range(Z)]

    # Maintain a sliding window of 3 Z-slices: prev, curr, next
    prev_slice = scalar_slices[0]
    curr_slice = scalar_slices[1]
    next_slice = scalar_slices[2]

    for z in range(Z):
        for y in range(Y):
            # Extract 1D scanlines (X dimension)
            scanline_curr = curr_slice[y]
            scanline_y_minus = curr_slice[max(0, y - 1)]
            scanline_y_plus = curr_slice[min(Y - 1, y + 1)]
            scanline_z_minus = prev_slice[y]
            scanline_z_plus = next_slice[y]

            # Call core routine for this scanline
            # vpScanlineNormals(vpc, X, scanline_curr, scanline_y_minus,
            #                   scanline_y_plus, scanline_z_minus,
            #                   scanline_z_plus, voxel_scan, ...)
            for x in range(X):
                s_curr = scanline_curr[x]
                s_y_minus = scanline_y_minus[x]
                s_y_plus = scanline_y_plus[x]
                s_z_minus = scanline_z_minus[x]
                s_z_plus = scanline_z_plus[x]

                # Compute gradient components from Y and Z neighbors [S1]
                dy = (s_y_plus - s_y_minus) / 2.0
                dz = (s_z_plus - s_z_minus) / 2.0
                dx = 0.0  # X-derivative not provided by scanline inputs [S1]

                grad_mag = (dx**2 + dy**2 + dz**2)**0.5
                if grad_mag > 0:
                    nx, ny, nz = dx / grad_mag, dy / grad_mag, dz / grad_mag
                else:
                    nx, ny, nz = 0.0, 0.0, 0.0

                gradients[z][y][x] = [dx, dy, dz]
                normals[z][y][x] = [nx, ny, nz]

        # Slide Z-window forward
        prev_slice = curr_slice
        curr_slice = next_slice
        next_idx = z + 3
        next_slice = scalar_slices[next_idx] if next_idx < Z else curr_slice

    return gradients, normals
```

## Subtopics

### VolPack Volume Normals Computation
A utility routine (`vpVolumeNormals`) that computes gradient-magnitude data and encoded normal vectors from a three-dimensional array of 8-bit scalar values in one pass [S1]. It differs from the scanline version by requiring the entire volume to be loaded into memory at once [S1]. It is chosen when memory is sufficient for the full volume, simplifying the data loading logic [S1]. It accepts a `VP_SKIP_FIELD` constant to omit storing scalars, gradients, or normals [S1].

## Operating characteristics
*   **Memory Complexity:** $O(V)$ for the volume storage, but only $O(S)$ auxiliary memory for the scalar data, where $V$ is total voxels and $S$ is slice size, as it keeps at most three slices in memory at a time [S1]. This is significantly lower than loading the entire scalar array if it exceeds RAM [S1].
*   **Time Complexity:** Linear with respect to the number of voxels, $O(V)$, as each voxel is processed once [S1].
*   **Strengths:** Enables processing of volumes larger than available RAM. Efficient for 8-bit scalar data [S1].
*   **Limitations:** Requires careful buffer management. Limited to 8-bit input scalars in the `vpVolumeNormals` variant [S1]. Boundary conditions are not explicitly defined in the high-level guide [S1].

## Background
This method is part of the VolPack library, a set of routines for volume rendering [S1]. The specific routines `vpScanlineNormals` and `vpVolumeNormals` are documented in the VolPack User's Guide [S1]. The library was developed at Stanford University [S1].

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
