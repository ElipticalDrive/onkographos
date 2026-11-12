# Gaussian Opacity Fields (GOF) Mesh Extraction

Also called: GOF-style 3-D mesh extraction, mesh extraction from standard 3-D Gaussian Splatting

## Overview
A method to extract a 3-D triangulated mesh from a trained Gaussian model (3-D Gaussian Splatting) without retraining, by building a volumetric density field and extracting an isosurface [S1, unofficial]. It differs from [Distance-based Gaussian Splatting](7411-distance-based-gaussian-splatting.md) by relying on opacity-based density accumulation rather than signed distance functions, and from [2D-Lift-3D Scene Segmentation](11513-2d-lift-3d-scene-segmentation.md) by operating directly on the 3D primitive structure rather than lifting 2D features.

## How it works
**Inputs:** A trained 3D Gaussian Splatting model containing $N$ Gaussians, each with center $\boldsymbol{\mu}_i$, covariance $\boldsymbol{\Sigma}_i$, opacity $\alpha_i$ (derived from $o_i$ and the Gaussian value at the query point), and spherical harmonics coefficients for color.
**Outputs:** A PLY file containing a triangulated mesh with per-vertex positions and colors.

**Data Structures:**
1.  **Voxel Grid:** A 3D array of size $R \times R \times R$ (where $R$ is the resolution) storing accumulated density values.
2.  **KD-Tree:** A spatial index built on the Gaussian centers $\{\boldsymbol{\mu}_i\}$ for nearest-neighbor color lookup.

**Steps:**
1.  **Initialize Voxel Grid:** Create a 3D grid of zeros with resolution $R$.
2.  **Build Density Field:** For each voxel center $\mathbf{x}$ in the grid, compute the density $f(\mathbf{x})$ by summing contributions from all Gaussians. The contribution of the $i$-th Gaussian is truncated at $3\sigma$.
    $$ f(\mathbf{x}) = \sum_{i=1}^{N} \alpha_i \cdot \exp\left(-\frac{1}{2} (\mathbf{x}-\boldsymbol{\mu}_i)^T \boldsymbol{\Sigma}_i^{-1} (\mathbf{x}-\boldsymbol{\mu}_i)\right) $$
    Where $\alpha_i$ is the opacity parameter of the Gaussian [S1, unofficial].
3.  **Extract Isosurface:** Apply the marching cubes algorithm to the voxel grid using a density threshold $T$.
    $$ \text{Mesh} = \text{marching\_cubes}(f, \text{level}=T) $$
    Specifically, `skimage.measure.marching_cubes` is used [S1, unofficial].
4.  **Color Assignment:** For each vertex $\mathbf{v}$ in the extracted mesh, find the nearest Gaussian center $\boldsymbol{\mu}_k$ using the KD-Tree. Assign the color of $\mathbf{v}$ to be the DC (zeroth-order) spherical harmonics component of the $k$-th Gaussian [S1, unofficial].

**Parameters:**
*   `resolution` ($R$): Voxel grid resolution. Default: 256 [S1, unofficial].
*   `density_threshold` ($T$): Marching cubes isosurface level. Default: 1.0 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial import KDTree
from skimage import measure

def extract_mesh(gaussians, resolution=256, density_threshold=1.0):
    """
    Extract mesh from 3D Gaussian Splatting using GOF method.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
                   .center (np.array [3]), .covariance (np.array [3,3]),
                   .opacity (float), .sh_coeffs (np.array for SH)
        resolution: int, voxel grid size
        density_threshold: float, isosurface level
        
    Returns:
        vertices: np.array [V, 3]
        faces: np.array [F, 3]
        colors: np.array [V, 3]
    """
    # 1. Determine bounding box from Gaussian centers
    centers = np.array([g.center for g in gaussians])
    min_bound = np.min(centers, axis=0)
    max_bound = np.max(centers, axis=0)
    # Add small padding to ensure Gaussians are fully contained
    padding = 0.1 * (max_bound - min_bound)
    min_bound -= padding
    max_bound += padding
    
    # 2. Initialize voxel grid
    grid_size = resolution
    voxel_grid = np.zeros((grid_size, grid_size, grid_size))
    
    # 3. Compute voxel centers
    x = np.linspace(min_bound[0], max_bound[0], grid_size)
    y = np.linspace(min_bound[1], max_bound[1], grid_size)
    z = np.linspace(min_bound[2], max_bound[2], grid_size)
    xx, yy, zz = np.meshgrid(x, y, z, indexing='ij')
    voxel_centers = np.stack([xx.ravel(), yy.ravel(), zz.ravel()], axis=1)
    
    # 4. Accumulate density
    # Note: In practice, this loop is optimized or parallelized.
    # We iterate over voxels and sum Gaussian contributions.
    # For implementation efficiency, one might iterate over Gaussians and splat to voxels,
    # but the formula defines the density at a point x.
    
    # Precompute inverse covariances and determinants if needed for normalization,
    # though the formula provided [S1, unofficial] uses the raw exponential term scaled by opacity.
    
    for idx, voxel_pos in enumerate(voxel_centers):
        density = 0.0
        for g in gaussians:
            # Compute Mahalanobis distance squared
            diff = voxel_pos - g.center
            # Check if within 3 sigma for truncation (optimization)
            # dist_sq = diff @ np.linalg.inv(g.covariance) @ diff
            # if dist_sq > 9.0: continue 
            
            # Compute Gaussian value
            # exp(-0.5 * diff^T Sigma^-1 diff)
            # Note: The source formula [S1, unofficial] does not include the normalization factor 
            # (2*pi)^(3/2) |Sigma|^0.5. We follow the source formula exactly.
            exponent = -0.5 * diff @ np.linalg.inv(g.covariance) @ diff
            gaussian_val = np.exp(exponent)
            
            density += g.opacity * gaussian_val
            
        voxel_grid.ravel()[idx] = density
        
    # 5. Marching Cubes
    # skimage.measure.marching_cubes returns (verts, faces, normals, values)
    verts, faces, normals, values = measure.marching_cubes(voxel_grid, level=density_threshold)
    
    # 6. Map vertices back to world coordinates
    # The voxel indices need to be converted to world coordinates
    # verts are in voxel index space (0 to resolution-1)
    # We need to map them to the bounding box defined in step 1
    
    # Create mapping from voxel index to world coordinate
    # x_world = min_x + (idx_x / (resolution-1)) * (max_x - min_x)
    # However, meshgrid and linspace behavior must match.
    # skimage returns indices. We reconstruct world coords.
    
    # Re-create the coordinate arrays to map indices to world space
    # Note: meshgrid with indexing='ij' means xx[i,j,k] corresponds to x[i], y[j], z[k]
    # The raveled index corresponds to (i, j, k)
    
    # Efficient mapping:
    # idx = i * (grid_size*grid_size) + j * grid_size + k
    # i = idx // (grid_size*grid_size)
    # j = (idx // grid_size) % grid_size
    # k = idx % grid_size
    
    # But simpler: use the linspace arrays directly if we know the mapping.
    # Let's assume standard mapping:
    # world_x = min_bound[0] + (verts[:,0] / (resolution - 1)) * (max_bound[0] - min_bound[0])
    # This assumes the voxel centers are evenly spaced between min and max.
    
    # Correct mapping for linspace(min, max, N):
    # The points are min + i * (max-min)/(N-1)
    
    step_x = (max_bound[0] - min_bound[0]) / (resolution - 1)
    step_y = (max_bound[1] - min_bound[1]) / (resolution - 1)
    step_z = (max_bound[2] - min_bound[2]) / (resolution - 1)
    
    world_verts = np.zeros_like(verts)
    world_verts[:, 0] = min_bound[0] + verts[:, 0] * step_x
    world_verts[:, 1] = min_bound[1] + verts[:, 1] * step_y
    world_verts[:, 2] = min_bound[2] + verts[:, 2] * step_z
    
    # 7. Color Assignment via KD-Tree
    tree = KDTree(centers)
    colors = np.zeros((len(world_verts), 3))
    
    for i, v in enumerate(world_verts):
        # Find nearest Gaussian
        dist, idx = tree.query(v)
        g = gaussians[idx]
        # Get DC SH coefficient (usually the first 3 values if SH degree 0, or first 3 of degree 1)
        # Assuming standard SH representation where DC is the first component
        # The source says "DC spherical-harmonics component".
        # If SH is stored as a vector, the DC term is typically the first 3 floats (for R, G, B).
        colors[i] = g.sh_coeffs[:3] 
        
    return world_verts, faces, colors
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating point errors in the exponential calculation for large Mahalanobis distances can lead to underflow. Truncation at $3\sigma$ mitigates this.
*   **Boundaries:** The voxel grid must fully encompass all Gaussians. If a Gaussian is partially outside the grid, its contribution is lost, potentially creating holes in the mesh. Padding is required [S1, unofficial].
*   **Degenerate Input:** If all opacities are zero, the density field is zero, and marching cubes may fail or return an empty mesh.

## Subtopics

### Mesh-3DGS Hybrid Representation
A representation where one flat Gaussian is allocated to each triangle of an extracted mesh, combining mesh geometry with Gaussian splatting [S2, unofficial]. It differs from standard GOF by adding a second training stage to refine the hybrid representation. Choose this when physically-based rendering (PBR) parameters are needed or when the initial mesh extraction quality is insufficient due to large triangles [S2, unofficial].
*   **Stage 1:** Train vanilla 3DGS for 40k steps (30k vanilla + 10k SDF regularization). Apply opacity culling. Apply SDF-based regularization to flatten Gaussians. Apply masked opacity regularization. Extract level set from trained 3DGS. Apply Poisson reconstruction to obtain surface mesh. Apply mesh cleanup (duplicate removal) [S2, unofficial].
*   **Stage 2:** Allocate one flat Gaussian to each triangle of the extracted mesh. Train non-PBR parameters (mesh geometry/location/shape, opacity, radiance SH coefficients, scale) for 10k steps [S2, unofficial].
*   **Parameters:** Stage 1: 40k steps. Stage 2: 10k steps [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(V \cdot N)$ for density field construction, where $V$ is the number of voxels ($R^3$) and $N$ is the number of Gaussians. Marching cubes is $O(V)$. Color lookup is $O(V \log N)$.
*   **Memory Complexity:** $O(R^3)$ for the voxel grid and $O(N)$ for the KD-tree.
*   **Performance:** Default resolution 256 provides a balance between detail and speed. Higher resolutions increase memory usage cubically.
*   **Strengths:** No retraining required. Simple to implement. Works with standard 3DGS outputs.
*   **Limitations:** Mesh quality depends on the density threshold. Large Gaussians can produce large triangles. Does not inherently produce watertight meshes without post-processing (e.g., Poisson reconstruction, though GOF itself uses marching cubes).
*   **When to use:** When a quick mesh extraction from an existing 3DGS model is needed without additional training overhead.
*   **When not to use:** When high-fidelity PBR materials are required (use Mesh-3DGS Hybrid instead) or when the scene has very large, anisotropic Gaussians that may not be well-represented by a voxel grid.

## Background
Originated from the Gaussian Opacity Fields (GOF) method by Yu et al., SIGGRAPH Asia 2024 [S1, unofficial]. It was implemented as an addition to the EOGS (Efficient Open Gaussian Splatting) repository to provide state-of-the-art mesh extraction from standard 3-D Gaussian Splatting without retraining [S1, unofficial].

## Sources
[S1] Add GOF-style 3-D mesh extraction to EOGS, https://github.com/Jay2002-whu/EOGS/pull/1, fetched 2026-10-27 (unofficial)
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
