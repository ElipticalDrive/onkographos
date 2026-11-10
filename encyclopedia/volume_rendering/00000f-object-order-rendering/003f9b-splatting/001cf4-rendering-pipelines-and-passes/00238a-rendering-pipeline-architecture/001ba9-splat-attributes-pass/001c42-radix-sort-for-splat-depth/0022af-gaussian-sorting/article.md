# Gaussian Sorting

Also called: Depth Sorting for Alpha Blending, Gaussian Depth Sorting

## Overview
Gaussian Sorting orders 3D Gaussians by their distance from the camera to ensure correct alpha blending during rasterization [S1]. It solves the transparency accumulation problem by establishing a front-to-back draw order, which is critical because Gaussians with non-zero opacity must be rendered in depth order [S1][S2, unofficial]. It builds on [Radix Sort for Splat Depth](1207-radix-sort-for-splat-depth.md) by applying the radix sort to a global index buffer of Gaussian attributes rather than just projected splat depths [S1].

## How it works
**Inputs:**
- A list of $N$ Gaussians, each defined by a position $\mathbf{p}_i$, opacity $\alpha_i$, and other attributes (color, covariance, etc.) [S1].
- A view matrix $\mathbf{V}$ transforming world space to camera space [S2, unofficial].
- A regular buffer (not sparse residency) storing Gaussian attributes [S1].

**Outputs:**
- A sorted index buffer $\mathcal{I}$ of length $N$, where $\mathcal{I}[k]$ is the index of the $k$-th closest Gaussian to the camera [S1].

**Data Structures:**
- **Gaussian Attribute Buffer:** A contiguous GPU `StorageBuffer` containing one entry per Gaussian [S1].
- **Index Buffer:** An array of `uint32` of size $N$, initialized to $[0, 1, \dots, N-1]$ [G].
- **Depth Buffer:** An array of `float` or `uint32` of size $N$ to store computed depths for sorting [G].

**Steps:**
1. **Transform to Camera Space:** For each Gaussian $i$, compute $\mathbf{p}'_i = \mathbf{V} \mathbf{p}_i$. The depth is the z-component $z_i = \mathbf{p}'_i.z$ [S2, unofficial].
2. **Compute Distances:** Use $z_i$ as the sort key for each Gaussian [S1][S2, unofficial].
3. **Radix Sort Indices:** Perform a radix sort on the index buffer $\mathcal{I}$ based on the computed depths $z_{\mathcal{I}[j]}$ using compute shaders [S1].
4. **Render:** Pass the sorted index buffer to the vertex shader to fetch attributes from the storage buffer in front-to-back order [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_sort(gaussians, view_matrix):
    """
    Sorts Gaussians by depth for alpha blending.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
                   - pos: (x, y, z) in world space
                   - opacity: float
        view_matrix: 4x4 matrix transforming world to camera space.
        
    Returns:
        sorted_indices: List of integers, indices into gaussians list, sorted front-to-back.
    """
    n = len(gaussians)
    indices = list(range(n))
    depths = []
    
    # Step 1 & 2: Transform to camera space and extract z-depth [S2, unofficial]
    for i in range(n):
        p = gaussians[i].pos
        # Homogeneous transformation: p_cam = V * [p.x, p.y, p.z, 1]^T
        w = view_matrix[3][0]*p[0] + view_matrix[3][1]*p[1] + view_matrix[3][2]*p[2] + view_matrix[3][3]
        z = (view_matrix[2][0]*p[0] + view_matrix[2][1]*p[1] + view_matrix[2][2]*p[2] + view_matrix[2][3]) / w
        depths.append(z)
        
    # Step 3: Stable radix sort indices by depths [S1][G]
    sorted_indices = stable_sort_by_key(indices, depths)
    
    return sorted_indices

def stable_sort_by_key(keys, values):
    """
    Stable sort that returns indices sorted by values.
    Python's built-in sort is stable; GPU implementations use 
    a 2-pass radix sort on uint32 reinterpreted floats [G].
    """
    return sorted(range(len(keys)), key=lambda i: values[i])
```

**Parameters:**
- **Buffer Type:** Regular buffer is used instead of sparse residency buffers [S1].
- **Sorting Algorithm:** Radix sort [S1].

**Edge Cases and Pitfalls:**
- **Sparse Residency Buffers:** Avoid using sparse residency buffers. They require indirection texture lookups for UV coordinates in virtual texturing, adding lookup overhead [S1]. Furthermore, support is limited: less than a third of desktop GPUs and just over 10% of Android devices support `sparseResidencyBuffer` [S1]. MoltenVK (Apple) generally lacks this feature [S1]. Even on hardware that supports it, performance may be unsatisfactory [S1].
- **Precision:** The depth values must be compared accurately. Using the z-coordinate in camera space is standard [S2, unofficial].
- **Degenerate Input:** If multiple Gaussians share identical depths, a stable sort or a secondary key must be used to impose a weak ordering, such as comparing original indices or base vertices [S3, unofficial][G].

## Subtopics

### [Temporal Coherence Heuristic for Gaussian Sorting](7339-temporal-coherence-heuristic-for-gaussian-sorting.md)
A method that introduces a temporal coherence heuristic into the tile sorting phase of 3D Gaussian Splatting rasterization, avoiding the computational cost of re-sorting Gaussians whose depth order has not changed between consecutive frames. It sets itself apart from standard per-frame sorting by leveraging the high degree of similarity between adjacent video frames to skip redundant sorting operations.

### Front-to-Back Rendering for Alpha Blending
An ordering technique that sorts draw calls front-to-back based on the dot product of the vector between chunk positions and the camera position, reducing z-buffer burden and increasing speed for alpha blending [S3, unofficial]. It is applied by calling `vertpool.order` with a lambda comparing two DAICs $a$ and $b$: if $\text{dot}(b.\text{pos} - a.\text{pos}, \text{cam}.\text{pos}) < 0$ return true; if $> 0$ return false; otherwise return $(a.\text{baseVert} < b.\text{baseVert})$ to impose a weak ordering [S3, unofficial].

### Z-order Sorting for Training Optimization
Sorts Gaussians in z-order at regular intervals during training to resolve cache contention in the alpha blending backward pass [S4, unofficial]. It significantly slows down training when the number of Gaussians is small due to massive atomic contention and false sharing, where threads in different tiles write to the same cache line [S4, unofficial]. When the per-Gaussian backward pass is enabled, omitting repeated z-ordering during densification reduces this false sharing issue [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ for radix sort on fixed-width keys [S1].
- **Memory Complexity:** $O(N)$ for the index buffer and depth arrays [S1].
- **Performance:** Using regular buffers avoids the indirection texture lookup overhead required by sparse residency buffers, leading to better performance on supported hardware [S1].
- **Strengths:** Ensures correct alpha blending for transparent Gaussians [S1]. Works on a wide range of hardware due to the avoidance of sparse residency features [S1].
- **Limitations:** Sorting is required for every frame to maintain correct alpha blending order [S1]. Sparse residency buffers are not recommended due to poor support and performance [S1].
- **When to use:** When rendering 3D Gaussians with opacity that require correct alpha blending [S1].
- **When not to use:** When Gaussians are opaque and depth sorting is not required for correctness [G].

## Background
Gaussian Sorting originated with the introduction of 3D Gaussian Splatting by Kerbl et al., where Gaussians with opacity require alpha blending and must be sorted by distance from the camera [S1]. The method was further detailed in implementations such as "Virtual memory for 3D Gaussian Splatting" [S1], which highlighted the use of compute shaders for depth calculation and radix sorting of indices. The approach is also referenced in unofficial implementations like `thomasantony/splat` [S2, unofficial] and adapted in voxel engine contexts for front-to-back rendering of chunks [S3, unofficial].

## Sources
[S1] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
[S2] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S3] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
[S4] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
[S5] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
