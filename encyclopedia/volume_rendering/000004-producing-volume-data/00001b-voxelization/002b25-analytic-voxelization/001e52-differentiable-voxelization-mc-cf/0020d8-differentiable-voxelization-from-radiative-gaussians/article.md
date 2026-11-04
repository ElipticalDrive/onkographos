# Differentiable Voxelization from Radiative Gaussians

Also called: Density Voxelization, Radiative Gaussian Voxelizer

## Overview
This method provides a differentiable voxelizer $\mathcal{V}$ that efficiently queries a 3D density volume $V \in \mathbb{R}^{X \times Y \times Z}$ from a set of radiative Gaussians $G_3$ [S1]. It solves the computational bottleneck of mapping continuous Gaussian splats to a discrete grid, enabling 3D regularization priors for tomographic reconstruction [S1]. Unlike [Differentiable Voxelization (MC/CF)](1742-differentiable-voxelization-mc-cf.md) which operates on polygonal meshes, this method operates directly on parametric Gaussian distributions, leveraging spatial tiling for acceleration [S1].

## How it works
**Inputs:**
- A set of $N$ radiative Gaussians $G_3$. Each Gaussian is defined by its mean $\mu$, covariance matrix $\Sigma$, and opacity/density parameters.
- Target grid dimensions $X, Y, Z$.
- Tile size $T=8$ (default, based on $8 \times 8 \times 8$ tiles) [S1].

**Outputs:**
- A 3D density volume tensor $V$ of shape $(X, Y, Z)$ [S1].

**Data Structures:**
- **Gaussian List:** Array of Gaussian parameters.
- **Tile Grid:** Logical partitioning of the bounding box of $G_3$ into $8 \times 8 \times 8$ voxel tiles.
- **Tile-AABB:** Axis-Aligned Bounding Box for each tile, used for culling.

**Steps:**
1. **Tiling:** Partition the target 3D space into a grid of $8 \times 8 \times 8$ voxel tiles [S1].
2. **Culling:** For each tile, compute its AABB. Retain only those Gaussians that have a 99% probability of intersecting the tile's AABB. This is determined by evaluating the Mahalanobis distance or bounding ellipsoid intersection against the tile bounds [S1].
3. **Parallel Computation:** For each active tile, compute the voxel values in parallel. For each voxel $(x,y,z)$ within the tile, sum the contributions of all retained Gaussians $g_i$ using the Gaussian kernel density estimation formula (Eq. 4 in [S1]):
   $$ V(x,y,z) = \sum_{i \in \text{active}} \alpha_i \exp\left(-\frac{1}{2} (\mathbf{p} - \mu_i)^T \Sigma_i^{-1} (\mathbf{p} - \mu_i)\right) $$
   where $\mathbf{p} = [x, y, z]^T$ is the voxel center, $\alpha_i$ is the opacity/scaling factor of Gaussian $i$, and $\Sigma_i$ is its covariance matrix [S1].
4. **Backpropagation:** Implement the gradient computation in CUDA to allow gradients to flow from the volume $V$ back to the Gaussian parameters ($\mu, \Sigma, \alpha$) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_radiative_gaussians(gaussians, grid_shape, tile_size=8):
    """
    Differentiable voxelization of radiative Gaussians.
    gaussians: List of Gaussian objects with attributes .mu, .cov, .alpha
    grid_shape: Tuple (X, Y, Z)
    tile_size: Integer, default 8
    """
    # 1. Determine global bounds and tile grid
    bounds = compute_bounds(gaussians)
    tiles = create_tile_grid(bounds, grid_shape, tile_size)
    
    # Initialize output volume
    volume = torch.zeros(grid_shape, requires_grad=True)
    
    # 2. Process each tile in parallel (conceptually)
    for tile in tiles:
        # 2a. Culling: Find Gaussians with 99% confidence of intersecting tile AABB
        active_gaussians = []
        for g in gaussians:
            if is_gaussian_in_tile(g, tile.aabb, confidence=0.99):
                active_gaussians.append(g)
        
        # 2b. Compute voxel values for this tile
        tile_coords = tile.get_voxel_indices(grid_shape)
        for idx in tile_coords:
            voxel_pos = index_to_world_coord(idx, bounds)
            density = 0.0
            for g in active_gaussians:
                # Eq. (4) from [S1]: Gaussian contribution
                diff = voxel_pos - g.mu
                # Compute quadratic form: diff^T * Sigma^-1 * diff
                # Note: In CUDA implementation, this is optimized [S1]
                exponent = -0.5 * quadratic_form(diff, g.cov)
                density += g.alpha * torch.exp(exponent)
            volume[idx] = density
            
    return volume
```

**Parameters:**
- Tile size: $8 \times 8 \times 8$ [S1].
- Culling confidence: 99% [S1].

**Edge Cases:**
- **Empty Tiles:** Tiles with no active Gaussians return zero density.
- **Precision:** CUDA implementation is used to maintain differentiability and speed [S1].

## Subtopics

### [Fast Volume Reconstruction from Gaussians](2575-fast-volume-reconstruction-from-gaussians.md)
A high-performance kernel that decomposes the Mahalanobis distance into separable 1D operations to accelerate the query of the density volume $V$.

## Operating characteristics
- **Performance:** Accelerates the query process to > 100 FPS [S1].
- **Complexity:** Reduced from naive $O(N \cdot V)$ by spatial culling and tiling. Exact complexity not given, but depends on the number of active Gaussians per tile [S1].
- **Memory:** Requires storage for the volume $V$ and temporary tile buffers.
- **Strengths:** Enables 3D regularization for radiative Gaussians in tomography [S1]. Differentiable via CUDA [S1].
- **Limitations:** Dependent on grid resolution and tile size configuration.

## Background
Originated in the context of Rectifying Radiative Gaussian Splatting for Tomographic Reconstruction [S1]. Inspired by voxelizers used in RGB tasks [S1].

## Sources
[S1] R$^2$-Gaussian: Rectifying Radiative Gaussian Splatting for Tomographic Reconstruction, https://arxiv.org/abs/2405.20693v2, fetched 2026-10-26
