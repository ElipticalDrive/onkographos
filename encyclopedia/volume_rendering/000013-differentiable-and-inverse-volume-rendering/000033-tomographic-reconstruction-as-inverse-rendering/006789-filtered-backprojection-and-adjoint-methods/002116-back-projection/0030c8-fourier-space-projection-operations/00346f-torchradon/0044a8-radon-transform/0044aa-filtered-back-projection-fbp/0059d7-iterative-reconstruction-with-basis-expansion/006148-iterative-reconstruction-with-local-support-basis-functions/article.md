# Iterative Reconstruction with Local Support Basis Functions

Also called: Algebraic Reconstruction with Visual Hull and Local Basis, Sparse-View Tomographic Reconstruction of Flames

## Overview
A tomographic method for reconstructing a volumetric density field from a small number of camera images by modeling the scene as a linear combination of basis functions with local support [S1]. It differs from Fourier-space methods like [Filtered Back-Projection (FBP)](16502-iterative-reconstruction-with-basis-expansion.md) by using basis functions with finite support to create a sparse system matrix, which allows the solution space to be restricted by a visual hull to discard zero-valued variables [S1].

## How it works
**Inputs:**
- A set of $N$ images from different camera viewpoints.
- Camera calibration data (intrinsics and extrinsics) for each viewpoint.
- A 3D voxel grid defining the spatial domain.
- A visual hull mask indicating regions where density is known to be zero.

**Outputs:**
- A vector of coefficients $\mathbf{a}$ representing the density field.
- The reconstructed 3D density field $\phi$.

**Data Structures:**
- **Voxel Grid:** A regular 3D grid of resolution $R_x \times R_y \times R_z$.
- **Basis Functions:** Defined on the voxel grid. Two types are described:
    1.  **Box Basis:** Defined on a single voxel $i$.
    2.  **Trilinear Basis:** Defined on a $2 \times 2 \times 2$ block of voxels, centered at the voxel corner, with one-voxel overlap in every dimension to ensure smooth blending [S1].
- **System Matrix $S$:** A sparse matrix of size $N_{pixels} \times N_{basis}$. Rows correspond to pixels, columns to basis functions. Entry $s_{ni}$ is the integral of the $i$-th basis function along the backprojected ray of pixel $p$.

**Steps:**

1.  **Define the Image Formation Model:**
    Model the image intensity $I_p$ for pixel $p$ as the line integral of the density field $\phi$ along the backprojected ray $c$ [S1]:
    $$ I_p = \int_c \phi \, ds \quad (1) $$
    Assumptions: negligible absorption/scattering and proportional self-emission [S1].

2.  **Basis Expansion:**
    Assume $\phi$ is a linear combination of basis functions $\phi_i$ with coefficients $a_i$ [S1]:
    $$ \phi(\mathbf{x}) = \sum_i a_i \phi_i(\mathbf{x}) $$
    Substitute into (1) and swap sum and integral [S1]:
    $$ I_p = \sum_i a_i \left( \int_c \phi_i \, ds \right) \quad (2) $$
    This forms the linear system $\mathbf{p} = S\mathbf{a}$, where $\mathbf{p}$ is the vector of pixel intensities, $S_{pi} = \int_c \phi_i \, ds$, and $\mathbf{a}$ is the vector of coefficients [S1].

3.  **Construct Matrix $S$ (Sparse Setup):**
    Compute entries $s_{ni}$ by intersecting backprojected rays with basis function supports [S1].
    -   Use **voxel-aligned** basis functions to reduce intersection computation from $O(n^3)$ to $O(3n)$ by intersecting $3n$ planes and performing a lookup [S1].
    -   **Box Basis:** For a box basis function $\phi_i^{Box}$ defined on voxel $i$ with bounds $[x_{min}^j, x_{max}^j]$, $[y_{min}^j, y_{max}^j]$, $[z_{min}^j, z_{max}^j]$ [S1]:
        $$ \phi_i^{Box}(x,y,z) = \begin{cases} 1 & \text{if } x_{min}^j < x \leq x_{max}^j \land y_{min}^j < y \leq y_{max}^j \land z_{min}^j < z \leq z_{max}^j \\ 0 & \text{otherwise} \end{cases} \quad (3) $$
        Let $x_1$ and $x_2$ be the intersection points of ray $c_n$ with voxel $i$. The entry is the distance between these points [S1]:
        $$ s_{ni} = \|x_2 - x_1\| \quad (4) $$
    -   **Trilinear Basis:** Arranged on the voxel grid with one-voxel overlap in every dimension [S1]. The intersection of the ray with the support is a cubic polynomial. This polynomial is integrated analytically [S1].

4.  **Restrict Solution Space (Visual Hull):**
    Discard variables (coefficients $a_i$) corresponding to basis functions entirely outside the visual hull (known to be zero) [S1]. This reduces the size of $\mathbf{a}$ and $S$ [S1].

5.  **Solve Linear System:**
    Solve for $\mathbf{a}$ using a least-squares approach [S1]:
    $$ \mathbf{a} = (S^T S)^{-1} S^T \mathbf{p} \quad (5) $$
    Note: The matrix $S$ is often ill-conditioned with a large condition number (quotient of largest to smallest singular value), making direct inversion difficult [S1]. Iterative solvers or regularization may be required in practice, though the source specifies the least-squares formulation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reconstruct_density(pixels, cameras, voxel_grid, visual_hull_mask, basis_type='box'):
    """
    Reconstructs density field using iterative reconstruction with local support basis functions.
    
    Args:
        pixels: List of pixel intensities for all cameras.
        cameras: List of camera objects with projection matrices.
        voxel_grid: 3D grid defining voxel boundaries.
        visual_hull_mask: 3D boolean array, True where density may exist.
        basis_type: 'box' or 'trilinear'.
        
    Returns:
        a: Vector of basis coefficients.
    """
    # 1. Identify active basis functions based on visual hull
    active_basis_indices = []
    for i, basis in enumerate(voxel_grid.basis_functions(basis_type)):
        if basis.intersects(visual_hull_mask):
            active_basis_indices.append(i)
            
    num_pixels = len(pixels)
    num_active_basis = len(active_basis_indices)
    
    # 2. Construct sparse matrix S
    # S is num_pixels x num_active_basis
    S = sparse_matrix(num_pixels, num_active_basis)
    
    for p_idx, pixel in enumerate(pixels):
        # Get backprojected ray for this pixel
        ray = cameras[pixel.camera_id].backproject_ray(pixel.u, pixel.v)
        
        for basis_col, basis_idx in enumerate(active_basis_indices):
            basis = voxel_grid.basis_functions(basis_type)[basis_idx]
            
            # Compute integral of basis function along ray
            # For box: distance between intersection points
            # For trilinear: analytical integral of cubic polynomial
            if basis_type == 'box':
                x1, x2 = ray.intersect_voxel(basis.voxel_bounds)
                s_val = norm(x2 - x1)
            elif basis_type == 'trilinear':
                # Intersection is cubic polynomial; integrate analytically
                poly = ray.intersect_trilinear_support(basis.grid_position)
                s_val = integrate_analytically(poly)
            else:
                raise ValueError("Unknown basis type")
                
            S[p_idx, basis_col] = s_val
            
    # 3. Solve least squares system: a = (S^T S)^-1 S^T p
    # Note: S^T S may be ill-conditioned [S1]
    StS = S.transpose() @ S
    StP = S.transpose() @ pixels
    
    # Use a stable solver (e.g., conjugate gradient or SVD) due to ill-conditioning [S1]
    # Direct inversion is discouraged due to large condition number [S1]
    a = solve_linear_system(StS, StP)
    
    return a
```

## Operating characteristics
- **Time Complexity:** Matrix construction is $O(3n)$ per ray/basis pair due to voxel-aligned intersection optimization, where $n$ is the number of planes intersected [S1]. Solving the linear system depends on the solver; direct inversion is $O(N^3)$ but unstable due to ill-conditioning [S1].
- **Memory Complexity:** The matrix $S$ is sparse, requiring memory proportional to the number of non-zero entries, which is significantly less than the full matrix required by Fourier basis functions [S1].
- **Strengths:** Allows high-quality reconstruction from a small number of camera images (sparse-view) by restricting the solution space with a visual hull [S1]. Handles perspective projection correctly [S1]. Trilinear basis functions allow smooth blending and hardware-accelerated visualization [S1].
- **Limitations:** The system matrix $S$ is ill-conditioned, making inversion difficult [S1]. Requires a static camera setup for dynamic flames [S1]. Assumes negligible absorption/scattering and proportional self-emission [S1].

## Background
Originated in the paper "Image-based tomographic reconstruction of flames" [S1]. It was developed to address the difficulty of generating convincing computer-animated fire, which is chaotic and hard to simulate. The method adapts algebraic reconstruction techniques (ART) to tomography, differing from Radon transform methods by using local support basis functions and a visual hull constraint [S1]. It is applicable to static camera setups observing dynamic flames, enabling fast processing of multi-video sequences for time-varying models [S1].

## Sources
[S1] Image-based tomographic reconstruction of flames, https://doi.org/10.1145/1028523.1028572, fetched 2026-10-26
