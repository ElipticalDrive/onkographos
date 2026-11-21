# Pre-rendering Frustum Culling

Also called: Explicit Frustum Index Culling

## Overview
Pre-rendering Frustum Culling is an engineering optimization for 3D Gaussian Splatting that eliminates unnecessary Gaussians from rasterization input by explicitly computing and storing in-frustum indices before the rendering kernel executes [S1]. Unlike standard implementations where frustum culling is fused into rendering kernels and computed implicitly by CUDA threads without explicit storage, this method explicitly stores indices to allow the elimination of non-in-frustum Gaussians from the input tensor, thereby reducing GPU memory usage and computation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The full set of 3D Gaussians defined by their parameters (position, covariance, opacity, spherical harmonics coefficients) and the camera frustum definition.
*   **Output:** A subset of Gaussian indices $S_i$ representing only those Gaussians that intersect the camera frustum, and the corresponding subset of Gaussian data passed to the rasterization kernel.

**Data Structures:**
*   **Index Buffer:** A GPU memory buffer storing the integer indices $S_i$. The size of this buffer is $|S_i|$, the count of in-frustum Gaussians [S1].
*   **Gaussian Data Tensor:** The input tensor to the rasterization kernel is resized or sliced to contain only the data corresponding to indices in $S_i$. The original input size $N$ is reduced to $|S_i|$ [S1].

**Steps:**
1.  **Frustum Intersection Test:** For each Gaussian in the scene, determine if it intersects the camera's viewing frustum. This is typically done by checking if the Gaussian's bounding volume (often derived from its covariance matrix) overlaps with the six planes of the frustum.
2.  **Index Collection:** Collect the indices of all Gaussians that pass the intersection test into a list $S_i$.
3.  **Storage:** Store the list $S_i$ in GPU memory prior to the execution of the rendering kernel [S1].
4.  **Data Filtering:** Filter the full Gaussian data tensor to include only the entries corresponding to indices in $S_i$. This reduces the input size from $N$ to $|S_i|$ [S1].
5.  **Kernel Execution:** Pass the filtered Gaussian data and the index list $S_i$ to the rasterization kernels. The kernels process only the in-frustum Gaussians [S1].

**Formulas:**
*   Let $N$ be the total number of Gaussians.
*   Let $\rho_i$ be the ratio of in-frustum Gaussians to total Gaussians for a given view, such that $|S_i| = \rho_i N$ [S1].
*   The reduction in input size is proportional to $\rho_i$. In large scenes with low $\rho_i$, the majority of input Gaussians are not in-frustum, resulting in substantial wasted computation in standard implementations [S1].

**Parameters:**
*   **Frustum Planes:** Defined by the camera projection matrix and view volume. Not given in the sources.
*   **Bounding Volume:** The method assumes a bounding volume for each Gaussian to perform the intersection test. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The accuracy of the frustum intersection test affects the value of $\rho_i$. Inaccurate tests may include Gaussians that do not significantly contribute to the image or exclude those that do, affecting visual quality and performance [S1].
*   **Dynamic Scenes:** If the camera or Gaussians move rapidly, the pre-computed indices $S_i$ may become stale if not updated every frame. The method is applied per-rendering step [S1].
*   **Memory Overhead:** Storing the index buffer $S_i$ adds a small memory overhead, but this is offset by the reduction in Gaussian data passed to the kernel [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pre_rendering_frustum_culling(gaussians, camera_frustum):
    """
    Perform pre-rendering frustum culling for 3D Gaussian Splatting.
    
    Args:
        gaussians: List or tensor of all Gaussian parameters (position, covariance, etc.)
        camera_frustum: Definition of the camera viewing frustum (e.g., plane equations)
        
    Returns:
        in_frustum_indices: List of indices of Gaussians intersecting the frustum
        filtered_gaussians: Subset of Gaussian data corresponding to in_frustum_indices
    """
    in_frustum_indices = []
    
    # Step 1 & 2: Frustum Intersection Test and Index Collection
    for i, gaussian in enumerate(gaussians):
        # Check if Gaussian i intersects the camera frustum
        # This involves checking the Gaussian's bounding volume against the frustum planes
        if intersects_frustum(gaussian, camera_frustum):
            in_frustum_indices.append(i)
            
    # Step 3: Storage (Implicit in the return of indices for GPU memory)
    # The indices are stored in GPU memory before kernel execution
    
    # Step 4: Data Filtering
    # Filter the Gaussian data to include only in-frustum Gaussians
    filtered_gaussians = [gaussians[i] for i in in_frustum_indices]
    
    # Step 5: Kernel Execution (Called by the renderer)
    # rasterization_kernel(filtered_gaussians, camera_frustum)
    
    return in_frustum_indices, filtered_gaussians

def intersects_frustum(gaussian, frustum):
    """
    Check if a Gaussian intersects the camera frustum.
    This is a placeholder for the specific intersection test logic.
    """
    # [G] Typically involves checking if the Gaussian's bounding sphere or ellipsoid
    # intersects with any of the six frustum planes.
    # For simplicity, we assume a bounding sphere check here.
    center = gaussian.position
    radius = gaussian.radius # Derived from covariance
    for plane in frustum.planes:
        if distance_point_to_plane(center, plane) > radius:
            return False
    return True
```

## Subtopics

### Enhanced Baseline
A GPU-only training variant that incorporates pre-rendering frustum culling to skip unnecessary kernel computation, serving as a closer baseline for evaluating offloading overhead [S1]. It is used in the Grendel-GS training framework to provide a better comparison for CLM's offloading strategy [S1].

## Operating characteristics
*   **Time Complexity:** Reduces the number of Gaussians processed by the rasterization kernel from $N$ to $|S_i| = \rho_i N$. This reduces computation time proportional to $\rho_i$ [S1].
*   **Memory Complexity:** Reduces GPU memory usage for Gaussian data passed to the kernel by a factor of $\rho_i$. However, it adds memory overhead for storing the index buffer $S_i$ [S1].
*   **Measured Performance:** In large scenes with low $\rho_i$, the method results in substantial reduction in wasted GPU computation and memory usage compared to standard implementations where all Gaussians are processed [S1].
*   **Strengths:** Simple engineering method; effective in large scenes with sparse Gaussians; reduces both memory and computation [S1].
*   **Limitations:** Requires an additional step to compute and store indices; may not provide significant benefits in scenes with high $\rho_i$ [S1].
*   **When to Use:** When rendering large scenes with low density of Gaussians within the camera frustum ($\rho_i$ is small) [S1].
*   **When Not to Use:** When the overhead of computing and storing indices outweighs the savings in kernel computation, or in scenes with high $\rho_i$ [S1].

## Background
Pre-rendering Frustum Culling was introduced in the CLM paper (2026) as part of the Grendel-GS training framework to remove the GPU memory barrier for 3D Gaussian Splatting [S1]. It was developed to address the inefficiency of standard 3DGS implementations where frustum culling is fused into rendering kernels, leading to wasted computation and memory for non-in-frustum Gaussians [S1]. The method is also applicable to traditional GPU-only training without offloading [S1].

## Sources
[S1] CLM: Removing the GPU Memory Barrier for 3D Gaussian Splatting, https://doi.org/10.1145/3779212.3790140, fetched 2026-10-31
