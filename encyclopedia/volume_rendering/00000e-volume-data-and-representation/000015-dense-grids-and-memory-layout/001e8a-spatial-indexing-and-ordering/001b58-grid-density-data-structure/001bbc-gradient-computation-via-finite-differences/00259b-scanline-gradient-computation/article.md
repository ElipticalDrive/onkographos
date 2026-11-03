# Scanline Gradient Computation
Also called: VolPack Scanline Normals Computation, vpScanlineNormals

## Overview
Scanline Gradient Computation is a memory-efficient method for computing 3D gradient vectors and surface normals for scalar volume data by processing one scanline (a 1D row of voxels) at a time. It solves the problem of high memory bandwidth and storage requirements when computing derivatives for large volumes, which typically require accessing neighboring voxels in all three dimensions. It sets itself apart from batch processing methods like [VolPack Volume Normals Computation](3126-volpack-volume-normals-computation.md) by maintaining only a sliding window of three slices in memory, making it suitable for volumes that exceed available RAM.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume. The method requires access to the current scanline, the scanline immediately preceding it in the Y dimension, the scanline immediately following it in the Y dimension, the scanline immediately preceding it in the Z dimension, and the scanline immediately following it in the Z dimension.
*   **Output:** For each voxel in the current scanline, the method computes and stores:
    1.  The gradient magnitude.
    2.  An encoded surface normal vector.
    3.  (Optional) The original scalar value.

**Data Structures:**
The implementation relies on a `Voxel` structure defined by the VolPack library, which contains fields for scalars, gradients, and normals. The exact layout is determined by `vpSetRawVoxels`, which defines the stride in bytes for X, Y, and Z dimensions.
*   `scalars`: Pointer to the current scanline of input scalar data (8-bit unsigned integers).
*   `scalars_minus_y`: Pointer to the previous Y-slice scanline.
*   `scalars_plus_y`: Pointer to the next Y-slice scanline.
*   `scalars_minus_z`: Pointer to the previous Z-slice scanline.
*   `scalars_plus_z`: Pointer to the next Z-slice scanline.
*   `voxel_scan`: Pointer to the output voxel scanline where results are written.

**Algorithm Steps:**
1.  Initialize the volume context and set the raw voxel layout using `vpSetRawVoxels` to define strides [S1].
2.  Load scalar data into a buffer. If the volume is too large for a single array, load data slice-by-slice [S1].
3.  For each scanline in the volume (typically iterating through Z and Y, then X):
    a.  Ensure the five required scanlines (current, $\pm Y$, $\pm Z$) are loaded into memory. This requires keeping at most three slices of the volume in memory at any time [S1].
    b.  Call the core computation routine `vpScanlineNormals` (or implement the equivalent logic) for the current scanline.
    c.  The routine computes the gradient using immediately adjacent neighbors in the Y and Z dimensions [S1].
    d.  Store the computed gradient magnitude and encoded normal into the `voxel_scan` output buffer at the specified field indices [S1].
4.  Repeat until all scanlines are processed.

**Formulas:**
The sources describe the use of "immediately-adjacent neighbors" to compute the gradient [S1]. While the explicit finite difference formula is not detailed in the provided text, standard central differences for a voxel $(x,y,z)$ with neighbors $S_{x,y,z}$ would typically be:
$$ \frac{\partial S}{\partial y} \approx \frac{S_{x,y+1,z} - S_{x,y-1,z}}{2} $$
$$ \frac{\partial S}{\partial z} \approx \frac{S_{x,y,z+1} - S_{x,y,z-1}}{2} $$
The gradient vector $\mathbf{g}$ is then $\left[ \frac{\partial S}{\partial x}, \frac{\partial S}{\partial y}, \frac{\partial S}{\partial z} \right]$. The normal is derived from the normalized gradient. The specific encoding of the normal vector into the `normal_field` is handled internally by the VolPack library [S1].

**Parameters:**
*   `size`: The length in bytes of one scanline of scalar data (equal to the X dimension of the volume) [S1].
*   `scalar_field`, `gradient_field`, `normal_field`: Integer indices specifying which fields within the `Voxel` structure to populate. Use `VP_SKIP_FIELD` if a specific field should not be stored [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The method requires neighbors in $\pm Y$ and $\pm Z$. At the boundaries of the volume, these neighbors do not exist. The VolPack documentation does not specify the boundary handling strategy (e.g., clamping, reflection, or zero-padding) in the provided text [S1]. Implementers must check the VolPack source or assume a specific behavior (often clamping or treating out-of-bounds as the boundary value) [G].
*   **Memory Management:** The method requires careful management of the sliding window of slices. Failing to update the pointers for `scalars_minus_y`, etc., correctly will result in incorrect gradients [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_scanline_normals(volume_dims, scalar_data, voxel_fields):
    """
    Simulates the logic of vpScanlineNormals for a 3D volume.
    Assumes scalar_data is a 3D array [Z][Y][X].
    """
    Z, Y, X = volume_dims
    # Output buffers
    gradients = np.zeros((Z, Y, X, 3))
    normals = np.zeros((Z, Y, X, 3))
    
    # Iterate through each voxel. 
    # Note: The actual VolPack routine processes one scanline (1D) at a time.
    # We simulate the access pattern here for clarity.
    for z in range(Z):
        for y in range(Y):
            for x in range(X):
                # Get neighbors. Handle boundaries by clamping to edge values [G]
                s_curr = scalar_data[z, y, x]
                s_y_minus = scalar_data[z, max(0, y-1), x]
                s_y_plus = scalar_data[z, min(Y-1, y+1), x]
                s_z_minus = scalar_data[max(0, z-1), y, x]
                s_z_plus = scalar_data[min(Z-1, z+1), y, x]
                
                # Compute central differences [G]
                # Note: VolPack uses 8-bit scalars, so precision loss is expected.
                dy = (s_y_plus - s_y_minus) / 2.0
                dz = (s_z_plus - s_z_minus) / 2.0
                # dx is typically 0 or computed from X-neighbors if available.
                # VolPack's vpScanlineNormals only takes Y and Z neighbors as inputs [S1].
                # This implies dx might be assumed 0 or computed differently.
                # Based on "immediately-adjacent neighbors in the y and z dimensions" [S1],
                # we assume dx is not computed via this specific scanline routine or is 0.
                dx = 0.0 
                
                grad = np.array([dx, dy, dz])
                grad_mag = np.linalg.norm(grad)
                
                if grad_mag > 0:
                    normal = grad / grad_mag
                else:
                    normal = np.array([0.0, 0.0, 0.0])
                    
                gradients[z, y, x] = grad
                normals[z, y, x] = normal
                
    return gradients, normals
```

## Subtopics

### VolPack Volume Normals Computation
A utility routine (`vpVolumeNormals`) that computes gradient-magnitude data and encoded normal vectors from a three-dimensional array of 8-bit scalar values in one pass. It differs from the scanline version by requiring the entire volume to be loaded into memory at once. It is chosen when memory is sufficient for the full volume, simplifying the data loading logic. It accepts a `VP_SKIP_FIELD` constant to omit storing scalars, gradients, or normals [S1].

## Operating characteristics
*   **Memory Complexity:** $O(V)$ for the volume storage, but only $O(S \cdot W)$ auxiliary memory for the scalar data, where $V$ is total voxels, $S$ is slice size, and $W$ is the window of 3 slices. This is significantly lower than loading the entire scalar array if it exceeds RAM [S1].
*   **Time Complexity:** Linear with respect to the number of voxels, $O(V)$, as each voxel is processed once.
*   **Strengths:** Enables processing of volumes larger than available RAM. Efficient for 8-bit scalar data.
*   **Limitations:** Requires careful buffer management. Limited to 8-bit input scalars in the `vpVolumeNormals` variant. Boundary conditions are not explicitly defined in the high-level guide [S1].

## Background
This method is part of the VolPack library, a set of routines for volume rendering. The specific routines `vpScanlineNormals` and `vpVolumeNormals` are documented in the VolPack User's Guide [S1]. The library was developed at Stanford University.

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
