# Screened Poisson Surface Reconstruction

Also called: Poisson Surface Reconstruction, SPSR, Surface Extraction via Screened Poisson Surface Reconstruction

## Overview
Screened Poisson Surface Reconstruction (SPSR) is an optimization-based method that fits a manifold surface directly to a single oriented point cloud without prior training [S2]. It solves a screened Poisson equation to interpolate a scalar field from oriented samples, extracting the zero-level set as a watertight mesh [S1, unofficial]. Unlike marching cubes, which evaluates a volumetric grid and often produces non-watertight meshes or requires searching for an ideal level-set threshold, SPSR operates directly on the oriented point samples to construct a continuous implicit surface [S1, unofficial].

## How it works
**Inputs:** An oriented point cloud consisting of positions $\mathbf{c}_i$ and unit normals $\mathbf{n}_i$ for $i=1 \dots N$ [S1, unofficial].
**Outputs:** A watertight triangle mesh [S1, unofficial].

**Data Structures:**
The input is a list of surfels. Each surfel $i$ contains:
- $\mathbf{c}_i \in \mathbb{R}^3$: Position.
- $\mathbf{n}_i \in \mathbb{R}^3$: Oriented unit normal.
- $w_i \in \mathbb{R}$: Optional weight (default 1.0).

**Method:**
1.  **Field Definition:** Define a scalar field $f(\mathbf{x})$ over domain $\Omega$ such that its gradient $\nabla f(\mathbf{x})$ approximates a target vector field $\mathbf{V}(\mathbf{x})$ constructed from the point cloud normals [S4, G].
2.  **Poisson Equation:** The reconstruction minimizes the energy functional $E(f) = \int_{\Omega} |\nabla f(\mathbf{x}) - \mathbf{V}(\mathbf{x})|^2 d\mathbf{x} + \lambda \int_{\Omega} |f(\mathbf{x})|^2 d\mathbf{x}$, which yields the screened Poisson equation $\Delta f - \lambda f = \nabla \cdot \mathbf{V}$ [S4, G]. The parameter $\lambda > 0$ stabilizes the solution and controls the trade-off between interpolation accuracy and smoothness [G].
3.  **Discretization:** Discretize $\Omega$ using a uniform grid or octree [G]. The vector field $\mathbf{V}$ is estimated at grid nodes using Shepard interpolation or kernel weighting of the input normals [G].
4.  **Linear System:** The discretized equation forms a sparse linear system $A f = b$, where $A$ is the screened Laplacian matrix and $b$ contains the divergence of the estimated vector field [G].
5.  **Solve:** Solve the linear system for $f$ using a sparse linear solver (e.g., Conjugate Gradient) [G].
6.  **Extraction:** Extract the isosurface $f(\mathbf{x}) = 0$ using a marching cubes algorithm [G].

**Parameters:**
-   `pointWeight`: Interpolation weight. Reducing this parameter relaxes the interpolation constraint, which is recommended to mitigate under-sampling artifacts [S1, unofficial].
-   $\lambda$ (screening parameter): Controls stability and smoothness. Not given in the sources.

**Edge Cases and Pitfalls:**
-   **Under-sampling:** In regions where the partition-of-unity condition is not met (e.g., surface boundaries or rapid geometric variation), Shepard normalization acts as a kernel regression (Nadaraya–Watson estimator) rather than an interpolant [S1, unofficial]. This causes surfel centers to deviate from the true surface.
-   **Artifacts:** Using SPSR with high interpolation weight in under-sampled regions may produce artifacts because the method attempts to interpolate through smoothed approximations [S1, unofficial].
-   **Sparse Inputs:** The method struggles with sparse or heavily corrupted inputs [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def screened_poisson_surface_reconstruction(points, normals, lambda_screen=1.0, point_weight=1.0, grid_size=64, grid_extent=1.0):
    """
    Reconstructs a mesh from oriented points using Screened Poisson Surface Reconstruction.
    
    Args:
        points: (N, 3) array of positions.
        normals: (N, 3) array of unit normals.
        lambda_screen: Screening parameter for stability.
        point_weight: Interpolation weight to relax constraints.
        grid_size: Number of cells per dimension.
        grid_extent: Half-width of the bounding box.
        
    Returns:
        vertices: (M, 3) array of mesh vertices.
        faces: (K, 3) array of mesh face indices.
    """
    import numpy as np

    # 1. Discretize domain into a uniform grid
    x = np.linspace(-grid_extent, grid_extent, grid_size + 1)
    y = np.linspace(-grid_extent, grid_extent, grid_size + 1)
    z = np.linspace(-grid_extent, grid_extent, grid_size + 1)
    X, Y, Z = np.meshgrid(x, y, z, indexing='ij')
    grid_nodes = np.stack([X.ravel(), Y.ravel(), Z.ravel()], axis=1)
    num_nodes = len(grid_nodes)

    # 2. Estimate vector field V at grid nodes using Shepard interpolation
    h = grid_extent / grid_size
    V = np.zeros((num_nodes, 3))
    for j in range(num_nodes):
        x_node = grid_nodes[j]
        dists = np.linalg.norm(points - x_node, axis=1)
        weights = point_weight * np.exp(-(dists ** 2) / (h ** 2))
        denom = np.sum(weights)
        if denom > 1e-8:
            V[j] = np.sum(weights[:, np.newaxis] * normals, axis=0) / denom

    # 3. Assemble sparse linear system A f = b
    # A corresponds to -Delta + lambda_screen * I
    # b corresponds to divergence of V
    rows, cols, vals = [], [], []
    b = np.zeros(num_nodes)
    for i in range(num_nodes):
        x_i = grid_nodes[i]
        # Compute divergence of V at node i using central differences
        div_V = 0.0
        for axis in range(3):
            # Approximate partial derivative using neighboring nodes
            # (Implementation details omitted for brevity; standard finite difference stencil)
            pass
        b[i] = div_V
        # Add Laplacian stencil to A
        # (Implementation details omitted for brevity; standard 7-point stencil)
        pass

    # 4. Solve linear system for f
    # f = solve(A, b) using a sparse linear solver
    # (Standard sparse solver implementation)
    pass

    # 5. Extract isosurface f(x) = 0
    # vertices, faces = marching_cubes(f, level=0)
    # (Standard marching cubes implementation)
    pass

    return vertices, faces
```

## Subtopics

### [Poisson Surface Reconstruction for Mesh Export](6401-poisson-surface-reconstruction-for-mesh-export.md)
A variant used to generate watertight, colored triangle meshes from implicit surface representations like 3D Gaussians. It differs by leveraging differentiable renderer outputs to create oriented point sets and transferring material attributes via interpolation.

### RGB-D to Mesh via Poisson Reconstruction
A pipeline that converts a monocular depth map and RGB image into a 3D mesh for an IB object [S3, unofficial]. It combines the depth map and RGB image to generate an RGB-D image, then creates a point cloud using the Open3D library [S3, unofficial]. Outliers are removed by discarding points whose distance from the five surrounding points exceeds one standard deviation [S3, unofficial]. The normal vector for each vertex is estimated from the cleaned point cloud, and a mesh is generated via Poisson surface reconstruction with a depth value of 10 [S3, unofficial]. Vertices with a density below the 0.1 quantile threshold are subsequently removed, based on the assumption that the object of interest exhibits relatively high density [S3, unofficial].

### Geometric INR for Oriented Point Clouds
This variant parameterizes the implicit function $f$ with a neural network instead of solving the PDE directly [S4]. It takes an oriented point cloud $\{p_i, N_i\}_{i=1}^n$ sampled from $S$ as input, where $p_i \in S$ and $N_i$ are normals [S4]. It enforces constraints on the network output: $f(p_i) = 0$ and $\nabla f(p_i) = N_i$ [S4]. To prevent spurious zero-level set components, it samples additional points $\{q_k\}$ outside the surface points $\{p_i\}$ to regularize training [S4]. Unlike classical Poisson reconstruction which solves $\Delta f = \nabla \cdot V$, this method solves via loss minimization [S4].

## Operating characteristics
-   **Time Complexity:** Depends on the solver and grid resolution. Solving the sparse linear system is typically $O(N)$ to $O(N \log N)$ for well-conditioned systems, where $N$ is the number of grid cells or points [G].
-   **Memory Complexity:** Proportional to the number of grid cells and non-zeros in the Laplacian matrix [G].
-   **Strengths:** Produces watertight, manifold meshes suitable for physics simulation and fabrication [S1, unofficial]. Robust to noise compared to simple interpolation methods [G].
-   **Limitations:** Struggles with sparse or heavily corrupted inputs [S2]. Requires careful tuning of the `pointWeight` parameter to handle under-sampled regions [S1, unofficial].
-   **When to use:** When a watertight mesh is required from an oriented point cloud, especially in inverse rendering pipelines where material attributes need to be transferred to the mesh [S1, unofficial].
-   **When not to use:** For sparse inputs or when the input point cloud is heavily corrupted, as the method may produce artifacts or fail to converge to a meaningful surface [S2].

## Background
Screened Poisson Surface Reconstruction was introduced by Kazhdan and Hoppe in 2013 [S1, unofficial]. It builds on the original Poisson Surface Reconstruction by Hoppe et al. (1992), adding a screening term to stabilize the solution [S1, G, unofficial]. It is widely used in computer graphics for surface reconstruction from point clouds, particularly in the context of inverse rendering and 3D reconstruction from multi-view images or neural radiance fields [S1, unofficial].

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S3] DITTO-NeRF: Diffusion-based Iterative Text To Omni-directional 3D Model, https://arxiv.org/abs/2304.02827v1, fetched 2026-10-26 (unofficial)
[S4] Geometric implicit neural representations for signed distance functions, https://doi.org/10.1016/j.cag.2024.104085, fetched 2026-10-26
