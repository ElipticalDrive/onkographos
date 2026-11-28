# Mesh Normalization

Also called: Mesh Normalization and Voxelization, Mesh Scaling and Normalization

## Overview
Mesh Normalization is a preprocessing method that transforms a raw 3D mesh into a canonical coordinate space, typically centering it at the origin and scaling it to fit within a unit cube or a specific target volume. It solves the problem of inconsistent mesh scales and positions, which would otherwise cause incorrect voxelization, Signed Distance Field (SDF) generation, or Gaussian splatting initialization. It differs from [SDF Grid Resampling from File](4878-sdf-grid-resampling-from-file.md) and [SDF Training Data Resampling](13279-sdf-training-data-resampling.md) by operating on the continuous mesh geometry prior to discretization, rather than on discrete grid data. It differs from [Mesh Loading and Normalization](7069-mesh-loading-and-normalization.md) by focusing specifically on the geometric transformation logic, whereas the sibling method includes file I/O and acceleration structure construction.

## How it works
**Inputs:** A 3D mesh defined by a set of vertices $V \in \mathbb{R}^{N \times 3}$ and optionally faces/edges.
**Outputs:** A normalized mesh with vertices $V' \in \mathbb{R}^{N \times 3}$ transformed to a canonical space (typically $[-0.5, 0.5]^3$ or $[-1, 1]^3$).

### Data Structures
*   **Vertices:** Array of shape $(N, 3)$ containing $(x, y, z)$ coordinates.
*   **AABB (Axis-Aligned Bounding Box):** Defined by two 3D vectors: `min_bound` and `max_bound`.

### Steps (Axis-Aligned Bounding Box Method)
This is the most common approach, cited in [S1, S2, S3, unofficial].

1.  **Compute Bounds:** Calculate the minimum and maximum coordinates along each axis for all vertices.
    $$ \text{min\_bound} = \min_{v \in V} v $$
    $$ \text{max\_bound} = \max_{v \in V} v $$
    $$ \text{AABB} = [\text{min\_bound}, \text{max\_bound}] $$

2.  **Compute Center:** Calculate the centroid of the bounding box.
    $$ \text{center} = \frac{\text{min\_bound} + \text{max\_bound}}{2} $$

3.  **Compute Scale Factor:** Determine the maximum extent of the bounding box and invert it to normalize to a unit size.
    $$ \text{extent} = \text{max\_bound} - \text{min\_bound} $$
    $$ \text{max\_extent} = \max(\text{extent}) $$
    $$ \text{scale\_factor} = \frac{1.0}{\text{max\_extent}} $$

4.  **Translate and Scale:** Transform each vertex $v$ to $v'$.
    $$ v' = (v - \text{center}) \times \text{scale\_factor} $$
    This maps the mesh into the range $[-0.5, 0.5]^3$.

5.  **Clip (Optional but Recommended):** To avoid boundary issues during subsequent voxelization or ray tracing, clip the normalized vertices slightly inside the unit cube.
    $$ v' = \text{clip}(v', -0.5 + \epsilon, 0.5 - \epsilon) $$
    Where $\epsilon$ is a small epsilon, e.g., $10^{-6}$ [S1, unofficial].

### Steps (Variance Normalization Method)
An alternative approach cited in [S4, unofficial] that normalizes based on the distribution of points rather than the bounding box.

1.  **Compute Center of Mass (COM):**
    $$ \text{com} = \frac{1}{N} \sum_{i=1}^{N} v_i $$

2.  **Translate to Origin:**
    $$ v_{temp} = v_i - \text{com} $$

3.  **Compute Covariance Matrix:**
    $$ C = \frac{1}{N} \sum_{i=1}^{N} v_{temp, i} v_{temp, i}^T $$

4.  **Compute Scale Factor:**
    $$ s = \frac{1}{\sqrt{\text{tr}(C)}} = \frac{1}{\sqrt{C_{00} + C_{11} + C_{22}}} $$

5.  **Scale and Translate Back:**
    $$ v' = (v_{temp} \times s) + \text{com} $$
    *Note: [S4, unofficial] specifies translating back to the original COM after scaling, preserving the original position but normalizing the spread.*

### Pseudocode (AABB Method)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def normalize_mesh_aabb(vertices: np.ndarray, epsilon: float = 1e-6) -> np.ndarray:
    """
    Normalize mesh vertices to [-0.5 + epsilon, 0.5 - epsilon]^3.
    
    Args:
        vertices: Array of shape (N, 3) containing vertex coordinates.
        epsilon: Small value to clip boundaries.
        
    Returns:
        Normalized vertices array of shape (N, 3).
    """
    if vertices.size == 0:
        return vertices

    # 1. Compute AABB
    min_bound = np.min(vertices, axis=0)
    max_bound = np.max(vertices, axis=0)
    
    # 2. Compute Center
    center = (min_bound + max_bound) / 2.0
    
    # 3. Compute Scale Factor
    extent = max_bound - min_bound
    max_extent = np.max(extent)
    
    # Avoid division by zero for degenerate meshes (points or lines)
    if max_extent == 0:
        return vertices
        
    scale_factor = 1.0 / max_extent
    
    # 4. Translate and Scale
    normalized_vertices = (vertices - center) * scale_factor
    
    # 5. Clip to avoid boundary issues
    normalized_vertices = np.clip(normalized_vertices, -0.5 + epsilon, 0.5 - epsilon)
    
    return normalized_vertices
```

### Parameters
*   **Target Range:** Typically $[-0.5, 0.5]^3$ [S1, S3, unofficial] or $[-1, 1]^3$ [S2, unofficial].
*   **Epsilon:** $10^{-6}$ is used in [S1, unofficial] to prevent vertices from lying exactly on the voxel boundaries, which can cause ambiguity in voxelization [S1, unofficial].
*   **Voxel Size:** When converting to voxels immediately after normalization, a specific `voxel_size` is used to create the grid within the bounds $(-0.5, -0.5, -0.5)$ to $(0.5, 0.5, 0.5)$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Degenerate Meshes:** If the mesh consists of a single point or a line, `max_extent` is 0, leading to division by zero. The implementation must check for this [G].
*   **Coordinate System:** Ensure the input mesh coordinates are in a consistent system (e.g., right-handed) before normalization.
*   **Reversibility:** If the original coordinates are needed later (e.g., for color interpolation or inverse transformation), the `center` and `scale_factor` must be stored [S1, unofficial]. The inverse transformation is $v = \frac{v'}{\text{scale\_factor}} + \text{center}$.

## Subtopics

### [Mesh Loading and Normalization](7069-mesh-loading-and-normalization.md)
This method extends basic normalization by including file I/O and the construction of an acceleration structure (e.g., BVH) for ray tracing. It solves the problem of converting raw asset files into a format suitable for analytic voxelization or SDF generation. It differs from [Mesh Normalization](3796-mesh-normalization.md) by handling the entire pipeline from disk to ready-to-trace geometry, and from [SDF Generation from Scanned Surfaces](3795-sdf-generation-from-scanned-surfaces.md) by stopping at the mesh representation rather than computing the distance field. It typically involves loading the mesh, applying the AABB normalization, and then building the acceleration structure.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of vertices. Computing min/max and mean is linear.
*   **Memory Complexity:** $O(N)$ to store the vertex array and temporary bounds.
*   **Strengths:** Simple, fast, and robust for most organic and synthetic meshes. Ensures consistent scale for downstream algorithms like voxelization or neural rendering.
*   **Limitations:** The AABB method can leave significant empty space for non-convex or elongated objects if the bounding box is large but the object is thin. Variance normalization [S4, unofficial] handles elongated shapes better but is more computationally expensive ($O(N)$ for covariance) and may not fit the object into a tight cube.
*   **When to use:** Always before voxelization or SDF generation to ensure the grid resolution is meaningful relative to the object size.
*   **When not to use:** If the absolute scale of the mesh is physically significant and must be preserved (e.g., in CAD or scientific visualization where real-world units matter).

## Background
Mesh normalization is a standard preprocessing step in computer graphics pipelines, particularly in the context of 3D deep learning and voxel-based rendering. It ensures that neural networks or rasterization/voxelization algorithms receive inputs of consistent scale, which aids in convergence and numerical stability. The AABB method is widely adopted in modern 3D Gaussian Splatting and NeRF implementations [S1, S2, unofficial]. The variance normalization method originates from earlier geometric processing literature dealing with shape matching and sparse voxel DAGs [S4, unofficial].

## Sources
[S1] GAP-LAB-CUHK-SZ/gaustudio, https://github.com/GAP-LAB-CUHK-SZ/gaustudio, fetched 2026-10-26 (unofficial)
[S2] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S3] mesh-voxelization/README.md at master, https://github.com/davidstutz/mesh-voxelization/blob/master/README.md, fetched 2026-10-26 (unofficial)
[S4] prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, https://github.com/prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, fetched 2026-10-26 (unofficial)
