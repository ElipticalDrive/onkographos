# Pre-rendering Frustum Culling

Also called: Coarse Frustum Culling, Explicit Frustum Index Culling, Frustum Culling (Coarse), Frustum Pre-culling, Visibility Pre-culling

## Overview
Pre-rendering Frustum Culling is an engineering optimization for 3D Gaussian Splatting that eliminates unnecessary Gaussians from rasterization input by explicitly computing and storing in-frustum indices before the rendering kernel executes [S3][S1, unofficial]. Unlike standard implementations where frustum culling is fused into rendering kernels and computed implicitly by CUDA threads without explicit storage, this method explicitly stores indices to allow the elimination of non-in-frustum Gaussians from the input tensor, thereby reducing GPU memory usage and computation [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** The full set of 3D Gaussians defined by their parameters (position, covariance, opacity, spherical harmonics coefficients) and the camera frustum definition [S3][S1, unofficial].
*   **Output:** A subset of Gaussian indices $S_i$ representing only those Gaussians that intersect the camera frustum, and the corresponding subset of Gaussian data passed to the rasterization kernel [S3][S1, unofficial].

**Data Structures:**
*   **Index Buffer:** A GPU memory buffer storing the integer indices $S_i$ [S3].
*   **Gaussian Data Tensor:** The input tensor to the rasterization kernel is filtered to contain only the data corresponding to indices in $S_i$, reducing the original input size $N$ to $|S_i|$ [S3].

**Steps:**
1. Launch a CUDA kernel (`checkFrustum`) with one thread per Gaussian ($P$) and a block size of 256 [S1, unofficial].
2. In the kernel, compute the view-space position $p_{view}$ for each Gaussian and call `in_frustum` to determine visibility, storing the boolean result in a `present` array [S1, unofficial].
3. Collect the indices of visible Gaussians into $S_i$ and store them in GPU memory prior to rendering kernel execution [S3].
4. Pass only the Gaussians in $S_i$ to the rasterization kernels, reducing the input size from $N$ to $|S_i|$ [S3].

**Formulas:**
*   Let $N$ be the total number of Gaussians.
*   Let $\rho_i$ be the ratio of in-frustum Gaussians to total Gaussians for a given view, such that $|S_i| = \rho_i N$ [S3].
*   The reduction in input size is proportional to $\rho_i$ [S3].

**Parameters:**
*   **Frustum Planes:** Defined by the camera projection matrix and view volume [G].
*   **Bounding Volume:** The method assumes a bounding volume for each Gaussian to perform the intersection test [G].

**Edge Cases and Pitfalls:**
*   **Precision:** The accuracy of the frustum intersection test affects the value of $\rho_i$ [G].
*   **Dynamic Scenes:** If the camera or Gaussians move rapidly, the pre-computed indices $S_i$ may become stale if not updated every frame [G].
*   **Memory Overhead:** Storing the index buffer $S_i$ adds a small memory overhead, but this is offset by the reduction in Gaussian data passed to the kernel [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pre_rendering_frustum_culling(gaussians, camera_frustum):
    """
    Perform pre-rendering frustum culling for 3D Gaussian Splatting.
    Args:
        gaussians: Tensor of shape (N, D) containing Gaussian parameters.
        camera_frustum: Object defining the six frustum planes and projection matrix.
    Returns:
        in_frustum_indices: Tensor of indices of Gaussians intersecting the frustum.
        filtered_gaussians: Subset of Gaussian data corresponding to in_frustum_indices.
    """
    present = []
    for i, gaussian in enumerate(gaussians):
        # Compute view-space position and check visibility
        p_view = transform_to_view_space(gaussian.position, camera_frustum.projection_matrix)
        if in_frustum(p_view, camera_frustum.planes):
            present.append(i)
    # Store indices in GPU memory and filter data tensor
    filtered_gaussians = gaussians[present]
    return present, filtered_gaussians

def in_frustum(p_view, planes):
    """Check if a view-space point is inside all six frustum planes."""
    for plane in planes:
        if distance_point_to_plane(p_view, plane) > 0:
            return False
    return True

def distance_point_to_plane(point, plane):
    """Compute the signed distance from a point to a plane."""
    return plane.normal.dot(point) - plane.distance
```

## Subtopics

### Enhanced Baseline
A GPU-only training variant that incorporates pre-rendering frustum culling to skip unnecessary kernel computation, serving as a closer baseline for evaluating offloading overhead [S3]. It is used in the Grendel-GS training framework to provide a better comparison for CLM's offloading strategy [S3].

## Operating characteristics
*   **Time Complexity:** Reduces the number of Gaussians processed by the rasterization kernel from $N$ to $|S_i| = \rho_i N$ [S3].
*   **Memory Complexity:** Reduces GPU memory usage for Gaussian data passed to the kernel by a factor of $\rho_i$, while adding a small overhead for storing the index buffer $S_i$ [S3].
*   **Measured Performance:** Achieves a 3.9× rendering speed improvement while maintaining equivalent visual quality for real-world 360° scenes [S2].
*   **Strengths:** Simple engineering method; effective in large scenes with sparse Gaussians; reduces both memory and computation [S3].
*   **Limitations:** Requires an additional step to compute and store indices [S3].
*   **When to Use:** When rendering large scenes with low density of Gaussians within the camera frustum ($\rho_i$ is small) [S3].

## Background
Pre-rendering Frustum Culling was introduced in the CLM paper (2026) as part of the Grendel-GS training framework to remove the GPU memory barrier for 3D Gaussian Splatting [S3]. It was developed to address the inefficiency of standard 3DGS implementations where frustum culling is fused into rendering kernels, leading to wasted computation and memory for non-in-frustum Gaussians [S3]. The method is also applicable to traditional GPU-only training without offloading [S3].

## Sources
[S1] Mercerai/PFGS, https://github.com/Mercerai/PFGS, fetched 2026-10-28 (unofficial)
[S2] HyRF: Hybrid Radiance Fields for Memory-efficient and High-quality Novel View Synthesis, https://doi.org/10.52202/085713-0617, fetched 2026-10-30
[S3] CLM: Removing the GPU Memory Barrier for 3D Gaussian Splatting, https://doi.org/10.1145/3779212.3790140, fetched 2026-10-31
