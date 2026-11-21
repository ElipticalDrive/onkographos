# Gaussian Preprocessing

Also called: forward preprocessing, preprocessCUDA

## Overview
Gaussian Preprocessing is a CUDA kernel that transforms 3D Gaussian parameters into voxel-space coordinates, conic matrices, and bounding radii to enable efficient rasterization. It serves as the bridge between the learned Gaussian representation and the tile-based rasterizer, converting world-space attributes into the specific data structures required for the subsequent splatting pass. It is distinct from [Conical Frustum to Gaussian](12183-conical-frustum-to-gaussian.md) by operating on pre-computed covariances and focusing on voxel-space transformation rather than frustum approximation.

## How it works
**Inputs and Outputs:**
The kernel takes as input the 3D means (`means3D`), scales (`scales`), rotations (`rotations`), opacities (`opacities`), and pre-computed 3D covariances (`cov3D_precomp`). It outputs normalized means (`means3D_norm`), depths (`depths`), 2D covariances (`cov3Ds`), a packed `conic_opacity` array (7 floats per Gaussian: 6 for the upper triangle of the inverse 2D covariance matrix + 1 for opacity), and a bitmask of `tiles_touched`.

**Data Structures:**
- `means3D`: `float*`, size $P \times 3$.
- `scales`: `glm::vec3*`, size $P \times 3$.
- `rotations`: `glm::vec4*`, size $P \times 4$ (quaternions).
- `opacities`: `float*`, size $P$.
- `cov3D_precomp`: `float*`, size $P \times 9$ (symmetric 3x3 matrix).
- `means3D_norm`: `float3*`, size $P \times 3$.
- `depths`: `float*`, size $P$.
- `cov3Ds`: `float*`, size $P \times 6$ (upper triangle of 2x2 2D covariance).
- `conic_opacity`: `float*`, size $P \times 7$.
- `tiles_touched`: `uint32*`, size equal to the number of tiles.

**Parameters:**
- $P$: Number of Gaussians.
- `BLOCK3D_SIZE`: 256 (threads per block).
- `NUM_CHANNELS`: Template parameter for output channels.
- `nVoxel_x/y/z`: Voxel grid dimensions.
- `sVoxel_x/y/z`: Voxel sizes (scaling factors).
- `center_x/y/z`: Center offset.
- `radii_x/y/z`: Bounding box radii.

**Steps:**
1. Launch the kernel with one thread per Gaussian. The grid dimension is calculated as $\lceil P / 256 \rceil$ blocks, with a block size of 256 threads.
2. Each thread $i$ processes the $i$-th Gaussian:
   a. Transform the 3D mean $\mathbf{m}$ to voxel space using the voxel size and center offset:
      $$ \mathbf{m}_{voxel} = (\mathbf{m} - \mathbf{c}) \odot \mathbf{s}_{voxel} $$
      where $\mathbf{c} = (center_x, center_y, center_z)$ and $\mathbf{s}_{voxel} = (sVoxel_x, sVoxel_y, sVoxel_z)$.
   b. Store the normalized mean in `means3D_norm[i]`.
   c. Compute the depth $z$ of the transformed mean and store it in `depths[i]`.
   d. Retrieve the pre-computed 3D covariance $\Sigma_{3D}$ from `cov3D_precomp`.
   e. Compute the 2D covariance $\Sigma_{2D}$ by projecting $\Sigma_{3D}$ onto the image plane. This involves the view matrix $J$ and the Jacobian of the affine approximation of the projection. The resulting 2D covariance is then inverted to form the conic matrix $C = \Sigma_{2D}^{-1}$.
   f. Pack the upper triangle of $C$ (6 values) and the opacity $\alpha$ into `conic_opacity[i]`.
   g. Compute the bounding box radii for the Gaussian in voxel space.
   h. Determine which tiles are intersected by the bounding box and set the corresponding bits in `tiles_touched`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for a single thread processing Gaussian i
def preprocess_gaussian(i, means3D, scales, rotations, opacities, cov3D_precomp, 
                        nVoxel, sVoxel, center, tiles_touched, num_tiles):
    # 1. Transform mean to voxel space
    m = means3D[i]
    m_voxel = (m - center) * sVoxel
    
    # 2. Store normalized mean and depth
    means3D_norm[i] = m_voxel
    depths[i] = m_voxel.z
    
    # 3. Get pre-computed 3D covariance
    cov3D = cov3D_precomp[i] # 9 floats
    
    # 4. Compute 2D covariance and conic matrix
    # Note: The exact projection formula depends on the view matrix and Jacobian,
    # which are typically passed as additional parameters or computed from camera intrinsics.
    # Here we assume a helper function exists that performs the projection.
    cov2D = project_covariance_3d_to_2d(cov3D) # Returns 6 floats (upper triangle)
    conic = invert_2x2_matrix(cov2D) # Returns 6 floats (upper triangle of inverse)
    
    # 5. Pack conic and opacity
    conic_opacity[i * 7 : (i + 1) * 7] = conic + [opacities[i]]
    
    # 6. Compute bounding box radii
    # Radii are derived from the eigenvalues of the 2D covariance or pre-computed
    # based on the 3D scale and rotation.
    radii = compute_gaussian_radii(cov2D)
    
    # 7. Mark tiles touched
    tile_bounds = compute_tile_bounds_from_radii(m_voxel.xy, radii)
    for tile in tile_bounds:
        if 0 <= tile < num_tiles:
            tiles_touched[tile] |= (1 << i) # Or use atomic OR if concurrent
```

**Edge Cases and Pitfalls:**
- Precision: Floating-point precision in the inversion of the 2D covariance matrix can lead to instability if the determinant is near zero. A small epsilon should be added to the diagonal before inversion [G].
- Boundaries: Ensure that tile indices are clamped to valid ranges to avoid out-of-bounds memory access.
- Degenerate Input: If a Gaussian has zero scale or opacity, the covariance may be singular. Handle this by skipping or setting a default conic matrix [G].

## Operating characteristics
- **Time Complexity:** $O(P)$, where $P$ is the number of Gaussians. Each Gaussian is processed by a single thread.
- **Memory Complexity:** $O(P)$ for the output arrays. The `tiles_touched` array size depends on the number of tiles.
- **Performance:** The kernel is highly parallel and efficient on GPUs due to the one-thread-per-Gaussian model. Performance scales linearly with the number of Gaussians.
- **Strengths:** Fast preprocessing, efficient tile culling, and minimal memory overhead.
- **Limitations:** Requires pre-computed 3D covariances. The quality of the rasterization depends on the accuracy of the 2D covariance projection.

## Background
Gaussian Preprocessing is part of the Gaussian Splatting pipeline, specifically designed for efficient rasterization of 3D Gaussians. It was introduced in the context of differentiable rendering and 3D Gaussian Splatting [S1, unofficial]. The method leverages CUDA for parallel processing to transform 3D Gaussian parameters into a format suitable for tile-based rasterization.

## Sources
[S1] Ruyi-Zha/r2_gaussian, https://github.com/Ruyi-Zha/r2_gaussian, fetched 2026-10-26 (unofficial)
