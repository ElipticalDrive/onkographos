# Unbounded TSDF Mesh Extraction

Also called: Command Line Mesh Extraction

## Overview
Extracts a triangle mesh from an unbounded scene by contracting space, performing adaptive TSDF fusion in the contracted space, and applying Marching Cubes [S1, unofficial]. It differs from bounded extraction by handling scenes without a predefined bounding box through spatial contraction, allowing the method to isolate the largest object and fill internal bubbles via preprocessing inherited from [Preprocessed Marching Cubes](3765-preprocessed-marching-cubes.md) [G].

## How it works
**Inputs:** A trained Gaussian model (positions, covariances, SH coefficients, opacities), camera poses, and command-line arguments controlling resolution and truncation [G].
**Outputs:** A `.ply` triangle mesh file containing vertices, normals, and diffuse colors [G].

**Data Structures:**
*   **Gaussian Model:** Stores $N$ Gaussians. Each Gaussian has position $\mathbf{x} \in \mathbb{R}^3$, covariance matrix $\mathbf{\Sigma} \in \mathbb{R}^{3 \times 3}$ (or rotation/scale), SH coefficients $\mathbf{c}_l$, and opacity $\alpha$ [G].
*   **TSDF Volume:** A 3D grid of size $N \times N \times N$. Each voxel stores a weighted average distance and a weight sum [G].
*   **Mesh:** A collection of triangles with vertex attributes (position, normal, color) [G].

**Formulas and Definitions:**
1.  **Contraction Function:** Maps unbounded space $\mathbf{x}$ to a bounded unit sphere [S1, unofficial].
    $$
    \text{contract}(\mathbf{x}) = \begin{cases} 
    \mathbf{x} & \text{if } \|\mathbf{x}\| < 1 \\
    \left(2 - \frac{1}{\|\mathbf{x}\|}\right) \frac{\mathbf{x}}{\|\mathbf{x}\|} & \text{otherwise}
    \end{cases}
    $$
2.  **Uncontraction Function:** Maps the bounded space back to unbounded space [S1, unofficial].
    $$
    \text{uncontract}(\mathbf{y}) = \begin{cases} 
    \mathbf{y} & \text{if } \|\mathbf{y}\| < 1 \\
    \frac{1}{2 - \|\mathbf{y}\|} \frac{\mathbf{y}}{\|\mathbf{y}\|} & \text{otherwise}
    \end{cases}
    $$
3.  **Normalization:**
    $$
    \text{normalize}(\mathbf{x}) = \frac{\mathbf{x} - \text{center}}{\text{radius}}
    $$
4.  **Unnormalization:**
    $$
    \text{unnormalize}(\mathbf{x}) = \mathbf{x} \cdot \text{radius} + \text{center}
    $$
5.  **Inverse Contraction:** Combines uncontraction and unnormalization [S1, unofficial].
    $$
    \text{inv\_contraction}(\mathbf{x}) = \text{unnormalize}(\text{uncontract}(\mathbf{x}))
    $$

**Parameters:**
*   `resolution` / `mesh_res`: Grid resolution for Marching Cubes. Default: 1024 [S1, unofficial].
*   `depth_trunc`: Maximum depth range for TSDF fusion. If negative, auto-calculated as $2 \cdot \text{radius}$ [S1, unofficial].
*   `voxel_size`: Physical size of a voxel. If negative, auto-calculated as $\text{depth\_trunc} / \text{mesh\_res}$ [S1, unofficial].
*   `sdf_trunc`: Truncation distance for the TSDF. If negative, auto-calculated as $5.0 \cdot \text{voxel\_size}$ [S1, unofficial].
*   `num_cluster`: Number of largest connected components to keep during post-processing. Default: 50 [S1, unofficial].

**Algorithm Steps:**

1.  **Determine Bounding Box Radius ($R$):**
    *   Compute the contracted positions of all Gaussians: $\mathbf{p}'_i = \text{contract}(\mathbf{p}_i)$ [S1, unofficial].
    *   Calculate the norm $\|\mathbf{p}'_i\|$ for all $i$ [S1, unofficial].
    *   Find the 95th percentile of these norms [S1, unofficial].
    *   Set $R = \min(1.9, \text{percentile} + 0.01)$ [S1, unofficial].

2.  **Define SDF Function:**
    *   Create a function `sdf_function(x)` that evaluates the signed distance field at point $\mathbf{x}$ by querying the Gaussian splatting model [G].

3.  **Execute Marching Cubes with Contraction:**
    *   Call `marching_cubes_with_contraction` with:
        *   `sdf_function`: The function defined in Step 2.
        *   `bounding_box_min`: $-R$
        *   `bounding_box_max`: $R$
        *   `level`: 0 (isosurface value)
        *   `resolution`: `mesh_res`
        *   `inv_contraction`: The function defined in Formula 5.
    *   This function samples the SDF on a grid in the contracted space, runs Marching Cubes, and applies `inv_contraction` to the resulting vertices to map them back to world space [S1, unofficial].

4.  **Color the Mesh:**
    *   For each vertex $\mathbf{v}$ in the extracted mesh, query the TSDF function or the Gaussian model to retrieve the RGB color value [S1, unofficial].
    *   Set `active_sh_degree` to 0 to export only the diffuse texture component, ignoring higher-order spherical harmonics [S1, unofficial].

5.  **Post-Processing:**
    *   Identify connected components (clusters) in the mesh [S1, unofficial].
    *   Keep the largest `num_cluster` clusters [S1, unofficial].
    *   Discard all other vertices and faces [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def contract(x):
    """Contraction function mapping unbounded space to unit sphere [S1, unofficial]."""
    norm = np.linalg.norm(x, axis=-1, keepdims=True)
    norm = np.maximum(norm, 1e-8)
    unit_x = x / norm
    inside = norm < 1.0
    factor = np.where(inside, 1.0, 2.0 - 1.0 / norm)
    return np.where(inside[..., None], x, factor * unit_x)

def uncontract(y):
    """Uncontraction function mapping unit sphere to unbounded space [S1, unofficial]."""
    norm = np.linalg.norm(y, axis=-1, keepdims=True)
    norm = np.maximum(norm, 1e-8)
    unit_y = y / norm
    inside = norm < 1.0
    factor = np.where(inside, 1.0, 1.0 / (2.0 - norm))
    return np.where(inside[..., None], y, factor * unit_y)

def unnormalize(x, center, radius):
    return x * radius + center

def normalize(x, center, radius):
    return (x - center) / radius

def inv_contraction(x, center, radius):
    return unnormalize(uncontract(x), center, radius)

def extract_mesh_unbounded(gaussians, args):
    """
    Extracts mesh from unbounded scene following [S1, unofficial].
    gaussians: Object containing xyz (N, 3), covariances, sh_coeffs, opacities.
    args: Namespace with mesh_res, num_cluster, etc.
    """
    # 1. Determine Bounding Box Radius R
    xyz = gaussians.xyz
    contracted_xyz = contract(xyz)
    norms = np.linalg.norm(contracted_xyz, axis=1)
    percentile_95 = np.percentile(norms, 95)
    R = min(1.9, percentile_95 + 0.01)

    resolution = args.mesh_res if hasattr(args, 'mesh_res') else 1024
    center = np.array([0.0, 0.0, 0.0])
    radius = 1.0

    # 2. Define SDF Function
    def sdf_function(x_contracted):
        # [G] Query TSDF volume or Gaussian model for signed distance at contracted coordinates.
        # Returns a scalar float representing the SDF value.
        return query_sdf_at_contracted_point(x_contracted)

    # 3. Execute Marching Cubes with Contraction
    # marching_cubes_with_contraction samples grid in [-R, R]^3,
    # evaluates sdf_function, runs Marching Cubes, and applies inv_contraction to vertices.
    vertices, faces = marching_cubes_with_contraction(
        sdf_function=sdf_function,
        bounding_box_min=-R,
        bounding_box_max=R,
        level=0.0,
        resolution=resolution,
        inv_contraction=lambda x: inv_contraction(x, center, radius)
    )

    # 4. Color the Mesh
    # Query TSDF function for RGB values at mesh vertices [S1, unofficial]
    colors = query_rgb_at_vertices(vertices)
    active_sh_degree = 0  # Export only diffuse texture [S1, unofficial]

    # 5. Post-Processing
    mesh = post_process_mesh(vertices, faces, colors, cluster_to_keep=args.num_cluster)
    return mesh
```

## Operating characteristics
*   **Time Complexity:** Dominated by the TSDF fusion or SDF evaluation at each grid point. With resolution $N$, there are $N^3$ points. Each point requires querying the Gaussian model, which is $O(M)$ where $M$ is the number of Gaussians, or $O(\log M)$ with spatial indexing. Total: $O(N^3 \cdot M)$ or $O(N^3 \cdot \log M)$ [G].
*   **Memory Complexity:** $O(N^3)$ for the TSDF volume and $O(M)$ for the Gaussian model [G].
*   **Performance:** The contraction allows handling unbounded scenes without arbitrary clipping. The 95th percentile radius $R$ adapts to the scene's extent [S1, unofficial].
*   **Strengths:** Handles unbounded scenes naturally; isolates the main object via clustering; fills internal bubbles via preprocessing [G].
*   **Limitations:** High memory usage for high resolutions; SDF evaluation from Gaussians can be noisy or inaccurate if not properly fused; relies on the quality of the Gaussian model [G].

## Background
Used in 3D Gaussian Splatting implementations for mesh extraction from unbounded scenes [S1, unofficial]. It addresses the limitation of standard Marching Cubes which requires a bounded volume, by introducing a space contraction technique. This allows for the extraction of meshes from scenes that extend to infinity or have no clear bounding box [G].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
