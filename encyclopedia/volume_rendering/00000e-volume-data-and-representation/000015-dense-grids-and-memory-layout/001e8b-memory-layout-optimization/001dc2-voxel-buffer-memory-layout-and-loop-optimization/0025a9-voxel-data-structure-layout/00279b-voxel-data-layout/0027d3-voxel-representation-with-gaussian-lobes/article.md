# Voxel Representation with Gaussian Lobes

Also called: Isotropic Gaussian Voxel Data

## Overview
Voxel Representation with Gaussian Lobes is a method for compactly encoding directional scattering properties and occlusion within dense voxel grids, replacing single-value averages with isotropic Gaussian distributions to improve rendering accuracy [S1]. It solves the problem of directional aliasing and loss of anisotropic detail in standard voxel cone tracing by storing the mean direction and spread of light transport or surface normals [S1]. Unlike [Voxel Data Volume Storage and Access](4453-voxel-data-volume-storage-and-access.md) which typically stores scalar density or color, this method stores vector-valued directional statistics, and unlike [Voxel Renderer with Top-Level Grid and Bricks](8114-voxel-renderer-with-top-level-grid-and-bricks.md) which focuses on hierarchical traversal, this method defines the fundamental data layout for the leaf-level directional information [S1].

## How it works
**Inputs:** Raw voxel data containing directional information (e.g., normals, light directions) and scalar properties (e.g., color, occlusion).
**Outputs:** A compact voxel buffer where each voxel stores a Gaussian lobe representation.

**Data Structures:**
Each voxel stores the following fields in a fixed layout:
1.  **Average Vector $D$**: A 3D vector representing the mean direction.
2.  **Standard Deviation $\sigma$**: Encoded implicitly via the norm of $D$ to ease interpolation.
3.  **Visibility/Occlusion**: A single scalar value representing the average percentage of blocked rays.
4.  **Material Color**: Opacity-weighted, alpha-premultiplied color.
5.  **Normal**: Pre-multiplied by opacity.

**Formulas:**
The variance $\sigma^2$ is derived from the norm of the average vector $|D|$ using the encoding defined in [Tok05] [S1]:
$$ \sigma^2 = \frac{1 - |D|}{|D|} $$
This encoding ensures that as the direction becomes more uniform ($|D| \to 1$), the variance approaches 0, and as the direction becomes uniform in all directions ($|D| \to 0$), the variance increases [S1].

**Steps:**
1.  **Accumulate Directions:** For each voxel, accumulate all relevant directional vectors (e.g., surface normals or incoming light directions) from the underlying geometry or simulation.
2.  **Compute Average Vector $D$:** Calculate the arithmetic mean of the accumulated directional vectors.
3.  **Encode Variance:** Compute $|D|$ and derive $\sigma$ using the formula above. [G] Note: If $|D| = 0$, the direction is undefined; a default isotropic distribution or zero vector should be used.
4.  **Compute Occlusion:** Calculate the average visibility (fraction of unblocked rays) for the voxel.
5.  **Compute Color:** Average the material colors, weighted by opacity.
6.  **Compute Normal:** Average the normals, weighted by opacity.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_voxel_gaussian_lobe(directions, colors, opacities, visibility_counts, total_rays):
    """
    Encodes a single voxel's data into Gaussian lobe representation.
    
    Args:
        directions: List of 3D vectors (normals or light directions).
        colors: List of 3D color vectors.
        opacities: List of scalar opacity values.
        visibility_counts: List of scalar visibility counts (unblocked rays).
        total_rays: Total number of rays cast for this voxel.
        
    Returns:
        dict: Encoded voxel data.
    """
    if not directions:
        return {
            "D": [0.0, 0.0, 0.0],
            "sigma_squared": 1.0, # Default isotropic
            "visibility": 0.0,
            "color": [0.0, 0.0, 0.0],
            "normal": [0.0, 0.0, 0.0]
        }

    # 1. Compute Average Vector D
    sum_D = [0.0, 0.0, 0.0]
    for d in directions:
        sum_D[0] += d[0]
        sum_D[1] += d[1]
        sum_D[2] += d[2]
    
    n = len(directions)
    D = [sum_D[i] / n for i in range(3)]
    
    # 2. Encode Variance
    norm_D = math.sqrt(D[0]**2 + D[1]**2 + D[2]**2)
    if norm_D < 1e-6:
        sigma_squared = 1.0 # Fallback for undefined direction
    else:
        sigma_squared = (1.0 - norm_D) / norm_D
        
    # 3. Compute Visibility
    sum_visibility = sum(visibility_counts)
    visibility = sum_visibility / total_rays if total_rays > 0 else 0.0
    
    # 4. Compute Opacity-Weighted Color
    sum_color = [0.0, 0.0, 0.0]
    total_opacity = 0.0
    for i in range(n):
        alpha = opacities[i]
        sum_color[0] += colors[i][0] * alpha
        sum_color[1] += colors[i][1] * alpha
        sum_color[2] += colors[i][2] * alpha
        total_opacity += alpha
        
    if total_opacity > 0:
        color = [sum_color[i] / total_opacity for i in range(3)]
    else:
        color = [0.0, 0.0, 0.0]
        
    # 5. Compute Normal (Pre-multiplied by opacity)
    # Note: The source states "Normal (pre-multiplied by opacity)".
    # This implies the stored normal is the average normal weighted by opacity.
    sum_normal = [0.0, 0.0, 0.0]
    for i in range(n):
        alpha = opacities[i]
        # Assuming directions here are normals for this step
        sum_normal[0] += directions[i][0] * alpha
        sum_normal[1] += directions[i][1] * alpha
        sum_normal[2] += directions[i][2] * alpha
        
    if total_opacity > 0:
        normal = [sum_normal[i] / total_opacity for i in range(3)]
    else:
        normal = [0.0, 0.0, 0.0]
        
    return {
        "D": D,
        "sigma_squared": sigma_squared,
        "visibility": visibility,
        "color": color,
        "normal": normal
    }
```

**Edge Cases and Pitfalls:**
*   **Zero Norm:** If the average vector $D$ has a norm of zero, the variance formula divides by zero. A fallback isotropic distribution ($\sigma^2 = 1$) or a zero vector must be handled [G].
*   **View-Dependency:** Storing a single average visibility value lacks view-dependency, which is a disadvantage for large thin objects [S1].
*   **Precision:** The encoding $\sigma^2 = (1 - |D|) / |D|$ is sensitive to small changes in $|D|$ near 1. High precision in $D$ is required to maintain accurate spread information [S1].

## Subtopics

### [Leaf Node Gaussian Approximation](14713-leaf-node-gaussian-approximation.md)
Leaf Node Gaussian Approximation is a method to approximate VDB leaf nodes (typically 512 voxels) with a single Gaussian primitive to reduce rendering complexity while maintaining spatial coherence. It serves as the lowest Level of Detail (LOD) in a hierarchical Gaussian representation, leveraging the inherent spatial clustering of VDB leaf nodes. Unlike higher LOD variants that subdivide leaf nodes into smaller blocks, this method treats the entire leaf node as a single cluster, making it distinct from [Medium LOD Approximation](4128-voxel-representation-with-gaussian-lobes.md#medium-lod-approximation) and [High LOD Approximation](4128-voxel-representation-with-gaussian-lobes.md#high-lod-approximation).

### Lobe Reordering
Lobe Reordering is a step to reorder the SGGX lobes within each voxel so that lobes with the same index across different voxels have similar shapes, enabling shared weight factors [S2]. This is critical when using voxel clustering where weight factors are shared among multiple voxels [S2]. The process involves: 1. Picking an arbitrary voxel $i$ as a reference and keeping its lobe ordering fixed. 2. For each other voxel $i'$, finding a permutation $\pi^*_{i'}$ of $\{1..m\}$ that minimizes the L2 difference between corresponding SGGX normal distributions $D$. 3. Applying $\pi^*_{i'}$ to reorder the lobes of voxel $i'$ [S2]. The optimal permutation is found by minimizing:
$$ \pi^*_{i'} = \arg \min_{\pi} \sum_{j=1}^{m} \int (D_{i,j}(m) - D_{i',\pi(j)}(m))^2 dm $$
Since the number of lobes $m$ is usually very small, $\pi^*$ can be computed via brute-force search [S2].

## Operating characteristics
*   **Time Complexity:** Encoding is linear in the number of directions per voxel. Lobe reordering is $O(K \cdot m!)$ where $K$ is the number of voxels and $m$ is the number of lobes, but $m$ is small enough for brute force [S2].
*   **Memory Complexity:** Each voxel stores a fixed-size structure (3D vector $D$, scalar $\sigma$, scalar visibility, 3D color, 3D normal). This is more memory-intensive than scalar voxel storage but less than storing full directional distributions [S1].
*   **Performance:** Improves rendering accuracy for anisotropic media compared to scalar averaging [S1, S2].
*   **Strengths:** Compact representation of directional data; enables efficient interpolation of Gaussian lobes [S1].
*   **Limitations:** Lacks view-dependency in occlusion; isotropic assumption may not capture all anisotropic effects perfectly [S1].
*   **When to use:** When directional information (normals, light directions) is important for rendering quality and memory is constrained.
*   **When not to use:** For highly view-dependent thin structures where single-value visibility is insufficient [S1].

## Background
The method originates from the work on Interactive Indirect Illumination Using Voxel Cone Tracing [S1]. It builds on the idea that storing arbitrary directional distributions is too memory-intensive, so isotropic Gaussian lobes are used as a compact approximation [S1]. The variance encoding $\sigma^2 = (1 - |D|) / |D|$ follows [Tok05] [S1]. The concept of lobe reordering for shared weight factors in clustered voxel representations was introduced in the context of downsampling scattering parameters for rendering anisotropic media [S2].

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
[S2] Downsampling scattering parameters for rendering anisotropic media, https://doi.org/10.1145/2980179.2980228, fetched 2026-10-28
