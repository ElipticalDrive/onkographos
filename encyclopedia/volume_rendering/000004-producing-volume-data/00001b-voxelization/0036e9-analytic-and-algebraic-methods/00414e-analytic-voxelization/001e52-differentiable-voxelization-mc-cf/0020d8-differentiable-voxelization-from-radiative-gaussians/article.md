# Differentiable Voxelization from Radiative Gaussians

Also called: Density Voxelization, Radiative Gaussian Voxelizer

## Overview
This method provides a differentiable voxelizer $\mathcal{V}$ that efficiently queries a 3D density volume $V \in \mathbb{R}^{X \times Y \times Z}$ from a set of radiative Gaussians $G_3$ [S1]. It solves the computational bottleneck of mapping continuous Gaussian splats to a discrete grid, enabling 3D regularization priors for tomographic reconstruction [S1]. Unlike [Differentiable Voxelization (MC/CF)](1742-differentiable-voxelization-mc-cf.md) which operates on polygonal meshes, this method operates directly on parametric Gaussian distributions, leveraging spatial tiling for acceleration [S1].

## How it works
**Inputs:**
- A set of $N$ radiative Gaussians $G_3$ [S1].
- Target grid dimensions $X, Y, Z$ [S1].
- Tile size $T=8$ (default, based on $8 \times 8 \times 8$ tiles) [S1].

**Outputs:**
- A 3D density volume tensor $V$ of shape $(X, Y, Z)$ [S1].

**Data Structures:**
- **Gaussian List:** Array of Gaussian parameters [S1].
- **Tile Grid:** Logical partitioning of the target space into $8 \times 8 \times 8$ voxel tiles [S1].
- **Tile-AABB:** Axis-Aligned Bounding Box for each tile, used for culling [S1].

**Steps:**
1. **Tiling:** Partition the target 3D space into a grid of $8 \times 8 \times 8$ voxel tiles [S1].
2. **Culling:** For each tile, compute its AABB. Retain only those Gaussians that have a 99% probability of intersecting the tile's AABB [S1].
3. **Parallel Computation:** For each active tile, compute the voxel values in parallel [S1]. For each voxel $(x,y,z)$ within the tile, sum the contributions of all retained Gaussians $g_i$ using the Gaussian kernel density formula [S1].
4. **Backpropagation:** Implement the gradient computation in CUDA to allow gradients to flow from the volume $V$ back to the Gaussian parameters [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_radiative_gaussians(gaussians, grid_shape, tile_size=8):
    volume = [[[0.0 for _ in range(grid_shape[2])] for _ in range(grid_shape[1])] for _ in range(grid_shape[0])]
    
    for tx in range(0, grid_shape[0], tile_size):
        for ty in range(0, grid_shape[1], tile_size):
            for tz in range(0, grid_shape[2], tile_size):
                active = []
                for g in gaussians:
                    if g.mu[0] + g.scale[0] >= tx and g.mu[0] - g.scale[0] <= tx + tile_size and \
                       g.mu[1] + g.scale[1] >= ty and g.mu[1] - g.scale[1] <= ty + tile_size and \
                       g.mu[2] + g.scale[2] >= tz and g.mu[2] - g.scale[2] <= tz + tile_size:
                        active.append(g)
                
                for dx in range(tile_size):
                    for dy in range(tile_size):
                        for dz in range(tile_size):
                            x, y, z = tx + dx, ty + dy, tz + dz
                            if x < grid_shape[0] and y < grid_shape[1] and z < grid_shape[2]:
                                density = 0.0
                                for g in active:
                                    dx_v, dy_v, dz_v = x - g.mu[0], y - g.mu[1], z - g.mu[2]
                                    exponent = -0.5 * (dx_v**2 * g.inv_cov[0][0] + dy_v**2 * g.inv_cov[1][1] + dz_v**2 * g.inv_cov[2][2] + 
                                                       2*dx_v*dy_v*g.inv_cov[0][1] + 2*dx_v*dz_v*g.inv_cov[0][2] + 2*dy_v*dz_v*g.inv_cov[1][2])
                                    density += g.alpha * math.exp(exponent)
                                volume[x][y][z] = density
    return volume
```

**Parameters:**
- Tile size: $8 \times 8 \times 8$ [S1].
- Culling confidence: 99% [S1].

**Edge Cases:**
- **Empty Tiles:** Tiles with no active Gaussians return zero density [G].
- **Precision:** CUDA implementation is used to maintain differentiability and speed [S1].

## Subtopics

### [Fast Volume Reconstruction from Gaussians](2575-fast-volume-reconstruction-from-gaussians.md)
A high-performance kernel that decomposes the Mahalanobis distance into separable 1D operations to accelerate the query of the density volume $V$. It solves the prohibitive memory and time complexity of direct Gaussian-to-voxel mapping by enabling efficient, differentiable volume rendering suitable for joint optimization of Gaussian parameters in tomographic reconstruction. It builds on [Differentiable Voxelization from Radiative Gaussians](2380-differentiable-voxelization-from-radiative-gaussians.md) by providing the specific high-performance kernel used to query the density volume $V$.

## Operating characteristics
- **Performance:** Accelerates the query process to > 100 FPS [S1].
- **Complexity:** Not given in the sources.
- **Memory:** Requires storage for the volume $V$ and temporary tile buffers [G].
- **Strengths:** Enables 3D regularization for radiative Gaussians in tomography [S1]. Differentiable via CUDA [S1].
- **Limitations:** Dependent on grid resolution and tile size configuration [G].

## Background
Originated in the context of Rectifying Radiative Gaussian Splatting for Tomographic Reconstruction [S1]. Inspired by voxelizers used in RGB tasks [S1].

## Sources
[S1] R$^2$-Gaussian: Rectifying Radiative Gaussian Splatting for Tomographic Reconstruction, https://arxiv.org/abs/2405.20693v2, fetched 2026-10-26
[S2] Fast Volume Reconstruction from Gaussians, https://arxiv.org/abs/2405.20693v2, fetched 2026-10-26
