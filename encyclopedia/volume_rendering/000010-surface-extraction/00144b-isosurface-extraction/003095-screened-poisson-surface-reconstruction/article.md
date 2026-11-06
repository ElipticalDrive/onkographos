# Screened Poisson Surface Reconstruction

Also called: Poisson Surface Reconstruction, SPSR, Surface Extraction via Screened Poisson Surface Reconstruction

## Overview
Screened Poisson Surface Reconstruction (SPSR) is an optimization-based method that fits a watertight surface directly to a single oriented point cloud without prior training [S2]. It solves a screened Poisson equation to reconstruct a scalar field whose zero-level set approximates the input geometry [S1, unofficial]. Unlike sibling methods such as [Marching Cubes](30-isosurface-extraction.md) which require a dense volumetric grid, SPSR operates on sparse point samples, making it distinct from [Advancing Front Framework for Implicit Surfaces](819-advancing-front-framework-for-implicit-surfaces.md) which builds meshes via local triangulation. It is particularly suited for generating explicit meshes from 3D Gaussian splats or unstructured point clouds where volumetric storage is prohibitive [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An oriented point cloud $P = \{(c_i, n_i)\}_{i=1}^N$, where $c_i \in \mathbb{R}^3$ are positions and $n_i \in \mathbb{R}^3$ are unit normals.
*   **Output:** A watertight triangle mesh representing the surface [S1, unofficial].

**Data Structures:**
The method discretizes the domain into a regular grid (typically Cartesian) or an octree. The core data structure is a sparse linear system matrix $A$ and a right-hand side vector $b$, derived from the discretized screened Poisson operator. The scalar field values are stored at grid nodes.

**Steps:**
1.  **Grid Construction:** Define a bounding box around the point cloud and construct a uniform grid with resolution $N_x \times N_y \times N_z$. The cell size $h$ is chosen based on the average nearest-neighbor distance of the input points to ensure sufficient sampling density [G].
2.  **Weighted Field Construction:** For each point $i$ with position $c_i$ and normal $n_i$, compute a contribution to the grid. The standard approach defines a vector field $V(x)$ such that $\nabla \cdot V(x) \approx \sum_i \delta(x - c_i) n_i \cdot \nabla \delta(x - c_i)$? No, the standard Poisson formulation seeks a scalar function $f$ such that $\nabla f \approx n$ on the surface. The screened Poisson equation is:
    $$ \nabla \cdot (\nabla f - \mathbf{v}) = \sigma^2 f $$
    where $\mathbf{v}$ is a vector field derived from the normals, and $\sigma$ is a screening parameter. Specifically, $\mathbf{v}(x) = \sum_i w_i(x) n_i$, where $w_i(x)$ are interpolation weights (e.g., Shepard weights or Gaussian kernels) that sum to 1 in dense regions [S1, unofficial].
3.  **Discretization:** Discretize the operator $\nabla \cdot (\nabla \cdot - \mathbf{v}) - \sigma^2 I$ using finite differences on the grid. This results in a sparse symmetric positive definite matrix $A$ and a vector $b$ representing the divergence of the weighted normal field.
4.  **Linear Solve:** Solve the linear system $A f = b$ for the scalar field $f$ at grid nodes. This is typically done using a conjugate gradient solver or multigrid method.
5.  **Mesh Extraction:** Extract the zero-level set of $f$ using [Marching Cubes](30-isosurface-extraction.md) or a similar isosurface extraction algorithm. The resulting mesh vertices are merged to ensure watertightness [30-isosurface-extraction.md].

**Parameters:**
*   **Screening Parameter ($\sigma$):** Controls the trade-off between interpolation and smoothing. A higher $\sigma$ enforces stricter interpolation of the input normals, while a lower value allows for more smoothing. Typical values depend on the grid resolution and point density.
*   **Interpolation Weight ($pointWeight$):** In the context of surfel-based inputs, this parameter relaxes the interpolation constraint. Reducing this weight mitigates artifacts in under-sampled regions [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Under-sampled Regions:** In areas where the partition-of-unity condition is not met (e.g., surface boundaries or sparse regions), the normalized accumulation acts as a kernel regression (Nadaraya–Watson estimator) rather than an interpolant [S1, unofficial]. This causes surfel centers to deviate from the true surface. SPSR with high interpolation weight may produce artifacts here [S1, unofficial].
*   **Sparse/Corrupted Inputs:** The method struggles with sparse or heavily corrupted inputs [S2].
*   **Precision:** The linear solve requires careful handling of numerical precision, especially for large grids.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def screened_poisson_surface_reconstruction(points, normals, grid_resolution, screening_param, interpolation_weight):
    """
    Reconstructs a mesh from an oriented point cloud using Screened Poisson Surface Reconstruction.
    
    Args:
        points: List of 3D positions (N, 3).
        normals: List of unit normals (N, 3).
        grid_resolution: Tuple (Nx, Ny, Nz) for the output grid.
        screening_param: Sigma for the screened Poisson equation.
        interpolation_weight: Weight to relax interpolation constraint.
        
    Returns:
        Vertices and Faces of the watertight mesh.
    """
    # 1. Define grid and bounding box
    bounds = compute_bounds(points)
    grid = create_grid(bounds, grid_resolution)
    
    # 2. Compute weighted vector field on grid
    # Initialize grid vectors to zero
    grid_vectors = zeros_like(grid, shape=(Nx, Ny, Nz, 3))
    
    for i in range(len(points)):
        c = points[i]
        n = normals[i]
        # Compute interpolation weights for this point to all grid nodes
        # Using a kernel function (e.g., Gaussian) centered at c
        weights = compute_kernel_weights(c, grid.nodes, bandwidth=estimate_bandwidth(points))
        
        # Accumulate weighted normal contribution
        # v(x) = sum(w_i(x) * n_i)
        for node in grid.nodes:
            idx = grid.node_index(node)
            grid_vectors[idx] += weights[idx] * n
            
    # Apply interpolation weight relaxation if needed
    # This effectively scales the contribution of the normals
    grid_vectors *= interpolation_weight
    
    # 3. Discretize the screened Poisson operator
    # Equation: div(grad(f) - v) = sigma^2 * f
    # Rearranged: div(grad(f)) - sigma^2 * f = div(v)
    A = build_laplacian_matrix(grid)
    A -= screening_param**2 * identity_matrix(grid.num_nodes)
    
    b = compute_divergence(grid_vectors, grid)
    
    # 4. Solve the linear system
    # A * f = b
    f = solve_linear_system(A, b, method='conjugate_gradient')
    
    # 5. Extract mesh from scalar field f
    # Use Marching Cubes on the grid with isovalue 0
    vertices, faces = marching_cubes(grid, f, isovalue=0.0)
    
    return vertices, faces
```

## Subtopics

### [Poisson Surface Reconstruction for Mesh Export](6401-poisson-surface-reconstruction-for-mesh-export.md)
A variant specifically designed to export watertight, colored triangle meshes from 3D Gaussians by solving the Poisson equation on an oriented point cloud derived from Gaussian splats. It focuses on transferring material attributes to the mesh vertices.

## Operating characteristics
*   **Time Complexity:** Dominated by the linear solve of the sparse system, which is $O(N)$ to $O(N \log N)$ depending on the solver and grid size, where $N$ is the number of grid nodes.
*   **Memory Complexity:** $O(N)$ for storing the sparse matrix and grid data.
*   **Strengths:** Produces watertight, smooth meshes from sparse point clouds; robust to noise; does not require volumetric storage of the entire scene.
*   **Limitations:** Struggles with sparse or heavily corrupted inputs; sensitive to the choice of screening parameter and interpolation weight; computationally expensive for very high-resolution grids.
*   **When to Use:** Ideal for reconstructing surfaces from 3D Gaussian splats or unstructured point clouds where a watertight mesh is required for downstream tasks like physics simulation or rendering [S1, unofficial].
*   **When Not to Use:** Not suitable for real-time applications due to the cost of the linear solve; not ideal for highly dynamic scenes where frequent reconstruction is needed.

## Background
SPSR was introduced by Kazhdan and Hoppe in 2013 [S1, unofficial]. It builds upon the original Poisson Surface Reconstruction method by Hoppe et al. (1992) [S1, unofficial], adding a screening term to improve stability and handle noisy data better. It has become a standard method for surface reconstruction from oriented point clouds, particularly in the context of 3D Gaussian Splatting and inverse rendering [S1, unofficial].

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
