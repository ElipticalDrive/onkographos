# Point Sampling from Mesh

Also called: Barycentric Triangle Sampling, Triangle Sampling SDF

## Overview
Point Sampling from Mesh generates a point cloud by randomly distributing points across the surface of a triangle mesh [S2, unofficial]. It solves the problem of converting explicit geometric representations into discrete, unstructured data suitable for downstream voxelization or neural rendering pipelines [S1, unofficial]. It differs from [Area-Based Mesh Sampling](14310-area-based-mesh-sampling.md) by not explicitly weighting triangles by area in the sampling logic, and from [KD-Tree Guided Point Cloud Sampling](4715-kd-tree-guided-point-cloud-sampling.md) by generating the initial raw samples rather than perturbing existing points based on local density [S1, 4715-kd-tree-guided-point-cloud-sampling.md, unofficial].

## How it works
**Inputs:**
- `vertices`: Array of shape $(N, 3)$ containing mesh vertex coordinates [G].
- `faces`: Array of shape $(M, 3)$ containing triangle indices referencing `vertices` [G].
- `num_points`: Total number of points to sample (default 100,000,000) [S1, unofficial].
- `batch_size`: Number of points to sample per iteration (default 10,000,000) [S1, unofficial].
- `device`: Execution device, either `'cpu'` or a GPU device identifier [S1, unofficial].

**Outputs:**
- `points`: Array of shape $(K, 3)$ containing the sampled 3D coordinates, where $K \le num\_points$ [G].

**Data Structures:**
- The mesh is represented as two arrays: `verts` (float32, $N \times 3$) and `faces` (uint32 or int32, $M \times 3$) [G].
- On GPU, these are converted to tensors and processed in parallel batches [S1, unofficial].
- On CPU, they are passed to a mesh object for sequential sampling [S1, unofficial].

**Algorithm:**
The method supports two execution paths based on the `device` parameter [S1, unofficial].

1.  **GPU Path:**
    -   Convert `vertices` and `faces` to device tensors [S1, unofficial].
    -   Iterate in batches of `batch_size`.
    -   Generate random face indices and barycentric coordinates in parallel.
    -   Compute points using the barycentric formula and transfer results to CPU [S1, unofficial].

2.  **CPU Path:**
    -   Construct a mesh object from `vertices` and `faces` [S1, unofficial].
    -   Iterate in batches of `batch_size`.
    -   Sample surface points sequentially [S1, unofficial].
    -   Append results to a list [S1, unofficial].

3.  **Finalization:**
    -   Concatenate all sampled batches.
    -   Truncate the result to exactly `num_points` if the total exceeds the limit [S1, unofficial].

**Mathematical Basis (Barycentric Sampling):**
When sampling a single triangle with vertices $A$, $B$, and $C$, two independent random values $r_1, r_2 \sim U[0, 1]$ are generated [S3, unofficial]. The point $P$ on the triangle surface is computed using barycentric coordinates [S3, unofficial]:

$$
P = (1 - \sqrt{r_1})A + \sqrt{r_1}(1 - r_2)B + r_2 C
$$

This formula ensures uniform distribution over the triangle area, avoiding clustering near vertices that occurs with linear interpolation [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from typing import List

def sample_points_from_mesh(
    vertices: np.ndarray,
    faces: np.ndarray,
    num_points: int = 100_000_000,
    batch_size: int = 10_000_000,
    device: str = 'cpu'
) -> np.ndarray:
    """
    Sample points from a mesh using barycentric coordinates.
    
    Args:
        vertices: (N, 3) array of vertex coordinates.
        faces: (M, 3) array of face indices.
        num_points: Total number of points to sample.
        batch_size: Number of points per batch.
        device: 'cpu' or GPU device string.
        
    Returns:
        (K, 3) array of sampled points.
    """
    samples_list: List[np.ndarray] = []
    num_batches = (num_points + batch_size - 1) // batch_size
    
    for i in range(num_batches):
        current_batch_size = min(batch_size, num_points - i * batch_size)
        
        # 1. Select random faces
        face_indices = np.random.randint(0, faces.shape[0], size=current_batch_size)
        
        # 2. Gather triangle vertices A, B, C
        A = vertices[faces[face_indices, 0]]
        B = vertices[faces[face_indices, 1]]
        C = vertices[faces[face_indices, 2]]
        
        # 3. Generate barycentric coordinates
        r1 = np.random.uniform(0.0, 1.0, size=(current_batch_size, 1))
        r2 = np.random.uniform(0.0, 1.0, size=(current_batch_size, 1))
        sqrt_r1 = np.sqrt(r1)
        
        # 4. Compute points: P = (1 - sqrt(r1)) * A + sqrt(r1) * (1 - r2) * B + r2 * C
        points = (1.0 - sqrt_r1) * A + sqrt_r1 * (1.0 - r2) * B + r2 * C
        
        samples_list.append(points)
        
    all_samples = np.concatenate(samples_list, axis=0)
    
    # Truncate to exact num_points if oversampled
    if len(all_samples) > num_points:
        all_samples = all_samples[:num_points]
        
    return all_samples
```

**Edge Cases and Pitfalls:**
-   **Precision:** Floating-point precision in barycentric coordinate calculation can cause slight non-uniformity on very small triangles [G].
-   **Degenerate Input:** If the mesh contains degenerate triangles (zero area), the sampling distribution may be skewed [G]. Standard libraries may filter these internally, but the output distribution might not be perfectly uniform if degenerate faces are included in the sampling pool [G].

## Subtopics

### [KD-Tree Guided Point Cloud Sampling](4715-kd-tree-guided-point-cloud-sampling.md)
This method augments ground truth point clouds by generating samples near the original points, where the noise magnitude is scaled by local point density estimated via KD-tree nearest neighbor distances. It prepares ground truth data for downstream voxelization or neural rendering pipelines by creating a denser, noise-perturbed dataset. It differs from basic point sampling by relying on spatial distribution constraints derived from existing points rather than raw mesh geometry [4715-kd-tree-guided-point-cloud-sampling.md].

### [Area-Based Mesh Sampling](14310-area-based-mesh-sampling.md)
Area-Based Mesh Sampling is a point cloud generation method where the number of sampled points per triangle is proportional to the triangle's area, determined by a surface density parameter. It differs from uniform sampling by respecting geometric scale and from KD-Tree guided sampling by relying solely on area metrics rather than spatial distribution constraints.

### Latent Vector to Point Cloud Generation
This variant involves generating a PLY mesh from a latent vector or parameter set, then sampling points from that mesh [S4, unofficial]. Steps include: determining a temporary filename for the mesh, calling a mesh creation function (`create_mesh` or `create_mesh_speedup`) to generate the PLY file, sampling points from the PLY mesh using `trimesh.sample.sample_surface`, and cleaning up the temporary file [S4, unofficial]. It is used when the mesh is not pre-existing but generated dynamically from a model [S4, unofficial].

### Gaussian Kernel Sampling for Simulation
This variant distinguishes between points sampled from a NeuS mesh surface for simulation and trained Gaussian kernels for rendering [S5]. It initializes GMLS kernels on simulation points $S_s$ and finds $k$ nearest neighbors from $S_s$ for each rendering point $p_{r,i}$ in $S_r$ [S5]. As simulation proceeds, rendering points are updated by interpolating from evolved simulation positions using the pre-built GMLS kernel [S5]. It is chosen when uniform distribution is critical for simulation accuracy, as sampling directly from trained Gaussian kernels results in uneven distribution and sparsely sampled regions (especially thin parts) [S5].

### Surface Point Cloud Generation
This variant returns an intermediate data structure containing a surface point cloud, scans, and a kd-tree of the point cloud, used for calculating SDFs multiple times for the same mesh or for debugging [S2, unofficial]. Steps include: if the mesh is a trimesh.Scene, dump and sum it to get a single mesh; if `surface_point_method` is 'scan', call `surface_point_cloud.create_from_scans`; if 'sample', call `surface_point_cloud.sample_from_mesh` [S2, unofficial]. A key parameter is `bounding_radius`, calculated as `np.max(np.linalg.norm(mesh.vertices, axis=1)) * 1.1` if not provided [S2, unofficial].

### Farthest-Point Sampling for Kernel Initialization
This variant uses a uniform sampling strategy based on mesh area to first oversample a large number of points on the mesh surface, and then uses farthest-point sampling to obtain kernel locations and face indices [S6]. Kernel colours are sampled from a uniform distribution between -1 and 1, with the mean subtracted and set to a global base colour [S6]. Angles are sampled from a uniform distribution between 0 and $\pi$, and anisotropies from 1 to 100 [S6]. Sharpness is sampled from beta distribution $B(2, 1)$ and scaled/shifted to 10-50, while threshold is sampled from $B(10/7, 1)$ and scaled to 0.9-1.0 (or 0.7-1.0 for multi-view rendering) [S6].

## Operating characteristics
-   **Time Complexity:** $O(K)$ where $K$ is the number of sampled points, as each point requires constant-time barycentric computation [G].
-   **Memory Complexity:** $O(K)$ for the output point cloud, plus $O(V+F)$ for the mesh storage [G].
-   **Performance:** GPU-based sampling is significantly faster for large `num_points` (e.g., 100M+) compared to CPU-based sampling due to parallel tensor operations [S1, unofficial].
-   **Strengths:** Leverages optimized libraries for batched execution; straightforward mathematical implementation [S1, unofficial].
-   **Limitations:** Does not preserve edge connectivity; requires sufficient memory for large point clouds [S3, unofficial].
-   **When to use:** When a uniform point cloud representation of a mesh surface is needed for voxelization or neural rendering initialization [S1, unofficial].
-   **When not to use:** When edge connectivity is required (use mesh directly) or when non-uniform density based on curvature/area is needed (use Area-Based Sampling) [S2, 14310-area-based-mesh-sampling.md, unofficial].

## Background
The method originates from the need to convert triangle meshes into point clouds for applications such as Signed Distance Field (SDF) generation and neural rendering [S2, unofficial]. The barycentric coordinate sampling technique described is a standard approach in computer graphics for uniform triangle sampling [S3, unofficial]. Implementations vary between CPU libraries like Trimesh and GPU-accelerated libraries like PyTorch3D [S1, unofficial].

## Sources
[S1] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S2] mesh_to_sdf, https://github.com/marian42/mesh_to_sdf, fetched 2026-10-28 (unofficial)
[S3] PDF Generating Signed Distance Fields from Triangle Meshes using Vulkan ..., https://williamwhitehouse.dev/assets/projects/2023/sdf-gen/gen-sdf-from-meshes-using-vulkan-compute-report.pdf, fetched 2026-10-30 (unofficial)
[S4] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S5] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S6] Heat Kernel Textures the Geodesic Gaussians That Do Not Splat, https://doi.org/10.1007/978-3-032-37595-7_17, fetched 2026-10-29
