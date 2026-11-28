# Voxel-based Radiance Fields

Also called: Voxel NeRF, Voxel Radiance Fields

## Overview
Voxel-based Radiance Fields replace the multi-layer perceptron (MLP) decoder of standard Neural Radiance Fields with structured voxel grids to store volume density and spherical harmonic (SH) coefficients, enabling real-time volume rendering by eliminating heavy feed-forward neural computations [S1]. Unlike MLP-based NeRF which requires evaluating fully-connected layers for every sample point, this method computes color directly from interpolated grid values, offering a memory-efficient framework that limits learnable parameters to the voxel grid values [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D spatial location $\mathbf{X}_k$ and a viewing direction $\mathbf{d}$ for the $k$-th sample point on a ray.
*   **Output:** Volume density $\sigma_k$ and view-dependent color $\mathbf{c}_k$.

**Data Structures:**
The representation consists of two distinct voxel grids:
1.  **Density Grid ($G_\sigma$):** Maps $\mathbb{R}^3 \rightarrow \mathbb{R}$. Stores scalar volume density values.
2.  **SH Coefficient Grid ($G_{sh}$):** Maps $\mathbb{R}^3 \rightarrow \mathbb{R}^{9 \times 3}$. Stores degree 2 spherical harmonic coefficients for RGB color. Each spatial location holds a vector of 9 coefficients per color channel (R, G, B), totaling 27 values per voxel [S1].

**Method Steps:**
1.  **Density Retrieval:** For a query point $\mathbf{X}_k$, perform trilinear interpolation using the values from the nearest 8 voxels in $G_\sigma$ to obtain the volume density $\sigma_k$ [S1].
2.  **SH Coefficient Retrieval:** For the same query point $\mathbf{X}_k$, perform trilinear interpolation using the values from the nearest 8 voxels in $G_{sh}$ to obtain a vector of SH coefficients $\mathbf{v}_k$ [S1].
3.  **Direction Mapping:** Map the viewing direction $\mathbf{d}$ to a 9-dimensional vector $\mathbf{s}(\mathbf{d})$ using the degree 2 spherical harmonic basis functions [S1].
4.  **Color Computation:** Compute the color $\mathbf{c}_k$ by taking the dot product of the SH direction vector and the interpolated coefficients [S1].

**Formulas:**
The volume density $\sigma_k$ at sample point $\mathbf{X}_k$ is defined as [S1]:
$$
\sigma_k = \mathcal{G}_\sigma(\mathbf{X}_k)
$$
where $\mathcal{G}_\sigma$ denotes trilinear interpolation from the nearest 8 voxels in $G_\sigma$.

The color $\mathbf{c}_k$ is defined as [S1]:
$$
\mathbf{c}_k = S(\mathbf{d})^\intercal \cdot \mathcal{G}_{sh}(\mathbf{X}_k)
$$
where:
*   $\mathbf{c}_k$ is the resulting RGB color vector.
*   $\mathbf{d}$ is the viewing direction.
*   $S(\mathbf{d})$ is the SH function mapping direction $\mathbf{d}$ to $\mathbb{R}^9$.
*   $\mathcal{G}_{sh}(\mathbf{X}_k)$ is the trilinearly interpolated vector of SH coefficients from the nearest 8 voxels in $G_{sh}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_radiance_field_query(X_k, d, G_sigma, G_sh):
    """
    Query voxel-based radiance field for density and color.
    
    Args:
        X_k: 3D spatial coordinate (float32[3])
        d: Viewing direction vector (float32[3])
        G_sigma: Density grid, shape [D, D, D], stores scalar density
        G_sh: SH coefficient grid, shape [D, D, D, 27], stores degree-2 SH coeffs for RGB
        
    Returns:
        sigma: Volume density (float32)
        c: RGB color (float32[3])
    """
    # 1. Retrieve density via trilinear interpolation
    # G_sigma is a 3D grid. Trilinear interpolation uses the 8 nearest neighbors.
    sigma = trilinear_interpolate(G_sigma, X_k)
    
    # 2. Retrieve SH coefficients via trilinear interpolation
    # G_sh is a 4D grid (3D space + 27 channels).
    sh_coeffs = trilinear_interpolate(G_sh, X_k) # Returns vector of size 27
    
    # 3. Map direction to SH basis vector
    # S(d) maps direction to R^9. For degree 2, there are 9 basis functions.
    # The coefficients are stored as 9 per channel (R, G, B).
    # We assume a standard degree-2 SH basis evaluation.
    sh_basis = evaluate_sh_basis(d, degree=2) # Returns vector of size 9
    
    # 4. Compute color
    # sh_coeffs layout: [R0..R8, G0..G8, B0..B8]
    # We split coefficients into R, G, B components
    sh_coeffs_r = sh_coeffs[0:9]
    sh_coeffs_g = sh_coeffs[9:18]
    sh_coeffs_b = sh_coeffs[18:27]
    
    # Dot product of SH basis and coefficients for each channel
    r = dot_product(sh_basis, sh_coeffs_r)
    g = dot_product(sh_basis, sh_coeffs_g)
    b = dot_product(sh_basis, sh_coeffs_b)
    
    c = np.array([r, g, b])
    
    return sigma, c

def trilinear_interpolate(grid, point):
    """
    Perform trilinear interpolation on a 3D grid.
    grid: 3D array (or 4D for SH grid)
    point: 3D coordinate in continuous space
    """
    # Normalize point to grid coordinates if necessary, assuming grid covers [-1, 1]^3 or similar
    # Find integer indices of the corner voxel
    x, y, z = point
    ix = floor(x)
    iy = floor(y)
    iz = floor(z)
    
    # Fractional parts
    fx = x - ix
    fy = y - iy
    fz = z - iz
    
    # Get values from 8 corners
    # Handle boundary checks [G]
    v000 = grid[ix, iy, iz]
    v100 = grid[ix+1, iy, iz]
    v010 = grid[ix, iy+1, iz]
    v110 = grid[ix+1, iy+1, iz]
    v001 = grid[ix, iy, iz+1]
    v101 = grid[ix+1, iy, iz+1]
    v011 = grid[ix, iy+1, iz+1]
    v111 = grid[ix+1, iy+1, iz+1]
    
    # Interpolate along x
    v00 = lerp(v000, v100, fx)
    v01 = lerp(v010, v110, fx)
    v10 = lerp(v001, v101, fx)
    v11 = lerp(v011, v111, fx)
    
    # Interpolate along y
    v0 = lerp(v00, v01, fy)
    v1 = lerp(v10, v11, fy)
    
    # Interpolate along z
    result = lerp(v0, v1, fz)
    
    return result

def lerp(a, b, t):
    return a + t * (b - a)

def evaluate_sh_basis(d, degree):
    """
    Evaluate spherical harmonic basis functions for a direction d.
    Returns a vector of size (degree+1)^2.
    For degree 2, size is 9.
    """
    # Normalize direction
    d = d / np.linalg.norm(d)
    x, y, z = d
    
    # Degree 0
    y0 = 0.282096
    
    # Degree 1
    y1 = 0.488603 * y
    y2 = 0.488603 * z
    y3 = 0.488603 * x
    
    # Degree 2
    y4 = 1.092548 * x * y
    y5 = 1.092548 * y * z
    y6 = 0.315392 * (3 * z**2 - 1)
    y7 = 1.092548 * x * z
    y8 = 0.546274 * (x**2 - y**2)
    
    return np.array([y0, y1, y2, y3, y4, y5, y6, y7, y8])
```

**Parameters:**
*   **Grid Resolution:** Not given in the sources.
*   **SH Degree:** 2 [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Trilinear interpolation requires handling points outside the grid bounds. Standard practice clamps coordinates or uses padding, but specific boundary handling is not detailed in the source [G].
*   **Precision:** Trilinear interpolation is differentiable, allowing gradient flow through the grid for training [S1].

## Subtopics

### Sparse Voxel Variants
Uses coarse-to-fine sparse voxel reconstruction combined with pruning strategies to further enhance memory efficiency during training [S1].

### Hash Encoding (Sibling)
[Neural Multi-View Image Reconstruction with Hash Encoding](13013-neural-multi-view-image-reconstruction-with-hash-encoding.md) uses hash tables instead of structured grids for spatial indexing, offering a trade-off between memory locality and storage density.

### G-NeLF (Sibling)
[G-NeLF](13480-g-nelf.md) incorporates geometric constraints into the radiance field optimization, differing in its regularization strategy rather than its spatial representation.

### SCALAR-NeRF (Sibling)
[SCALAR-NeRF](14267-scalar-nerf.md) focuses on scalar field representations for specific optimization properties, distinct from the explicit voxel grid storage of Voxel NeRF.

### TriDF (Sibling)
[TriDF](14761-tridf.md) utilizes tri-plane representations, which are a compressed form of 3D feature storage, differing from the full 3D voxel grid.

### Tile and Slide (Sibling)
[Tile and Slide Framework](15219-tile-and-slide-framework.md) processes the scene in tiles to manage memory, whereas Voxel NeRF typically loads the entire grid or sparse structure into memory.

## Operating characteristics
*   **Time Complexity:** Rendering is significantly faster than MLP-based NeRF because it eliminates the feed-forward neural network computation. Color is computed via direct interpolation and dot product [S1].
*   **Memory Complexity:** Memory usage is proportional to the number of voxels. The density grid requires 1 value per voxel, and the SH grid requires 27 values per voxel (degree 2, 3 channels) [S1].
*   **Performance:** Achieves real-time volume rendering [S1].
*   **Strengths:** Memory-efficient framework; limits learnable parameters to grid values; enables real-time rendering by avoiding heavy MLP computations [S1].
*   **Limitations:** Resolution is limited by grid size; aliasing may occur at high frequencies if grid resolution is insufficient; not given in the sources.
*   **When to use:** When real-time rendering is required and memory efficiency is critical, such as in dynamic scenes or mobile applications [S1].
*   **When not to use:** When high-frequency detail is required beyond the grid resolution, or when the scene geometry is highly complex and sparse, requiring adaptive resolution not provided by a uniform grid.

## Background
Voxel-based Radiance Fields were introduced as an optimization of NeRF [29] to address the computational bottleneck of MLP-based decoders. The method was notably applied in ExBluRF for efficient radiance field optimization in the presence of extreme motion blur [S1]. By replacing the MLP with voxel grids, the approach leverages differentiable volume rendering while achieving real-time performance through direct grid lookups and interpolation [S1].

## Sources
[S1] ExBluRF: Efficient Radiance Fields for Extreme Motion Blurred Images, https://arxiv.org/abs/2309.08957v3, fetched 2026-10-27
