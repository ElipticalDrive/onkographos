# Analytic and Algebraic Methods


## Overview
Analytic and Algebraic Methods convert continuous parametric objects into discrete volumetric representations by evaluating exact mathematical properties (such as volume overlap or distance) rather than relying on rasterization or sampling [G]. This family solves the problem of voxelizing analytical implicit surfaces for direct volume rendering [S1, unofficial]. It differs from [Rasterization-based Voxelization](11836-rasterization-and-projection.md) by computing exact geometric metrics instead of projected coverage, and from [Point Cloud Voxelization](1000-point-cloud-voxelization.md) by operating on continuous parametric definitions rather than discrete point sets [G].

## Shared principles
All methods in this family map a continuous parametric object $Q_c$ into a digital representation $D_p$ defined on a 3D grid [G]. The core mechanism involves:
1. **Parametric Definition**: The input is defined by analytical equations (e.g., implicit functions, parametric surfaces) rather than discrete samples [G].
2. **Geometric Evaluation**: For each voxel, the method computes a scalar value based on the geometric relationship between the voxel volume and the object $Q_c$ [G].
3. **Value Mapping**: The output is a scalar field where values represent either the signed Euclidean distance to the surface or the fractional volume of the voxel occupied by the object [G].

## Subtopics

### [Signed Distance Field Voxelization](3849-signed-distance-field-voxelization.md)
Computes the signed Euclidean distance from each voxel center to the nearest point on the input surface. It differs from other methods by providing a continuous, differentiable distance metric suitable for marching cubes extraction, retaining sign information via winding numbers.

### [Surface-Aware Discretization](6820-surface-aware-discretization.md)
Estimates the effective volume of each voxel by performing Monte Carlo ray tracing against the mesh to determine the fraction of the voxel inside the object. It differs from analytic methods by using stochastic sampling to approximate partial occupancy, mitigating resolution artifacts in additive manufacturing.

### [Analytic Voxelization](10493-analytic-voxelization.md)
Assigns each voxel a membership value $m(v) \in [0, 1]$ representing the exact fraction of the voxel's volume occupied by the object. It differs from SDF methods by providing volumetric overlap fractions rather than distance metrics, naturally retaining boundary ambiguities for soft segmentation.

### Recursive Voxelization with Interval Arithmetic
Rendering of recursively voxelized object space with interval arithmetic. This approach evaluates recursive geometric functions within bounded voxel intervals to maintain numerical stability during rendering [S1, unofficial].

## Background
In rendering general-form implicit surfaces, sampling an analytical implicit into a regular grid by voxelization followed by direct volume rendering is a standard approach [S1, unofficial]. The technique of rendering recursively voxelized object space with interval arithmetic was first proposed by [136] [S1, unofficial]. Discrete ray tracing and GPU-based volume ray casting remain valid alternatives for rendering voxellized datasets [S1, unofficial].

## Sources
[S1] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)

```python
import numpy as np

def surface_aware_discretization(mesh, resolution, sample_count=2**14):
    """
    Estimates the effective volume of each voxel by performing Monte Carlo ray tracing.
    
    Args:
        mesh: Target triangular mesh object supporting ray intersection.
        resolution: Tuple (nx, ny, nz) defining grid dimensions.
        sample_count: Number of Monte Carlo samples per voxel. Default 2^14.
        
    Returns:
        3D numpy array of shape (nz, ny, nx) with estimated occupied volumes.
    """
    nx, ny, nz = resolution
    voxel_size = 1.0  # Derived from bounding box in practice
    voxel_volume = voxel_size ** 3
    count_in = np.zeros((nz, ny, nx), dtype=np.uint32)
    
    # 1. Initialization
    if not is_surface_aware():
        return np.full((nz, ny, nx), voxel_volume)
    
    # 2. Monte Carlo Sampling Loop
    for k in range(nz):
        for j in range(ny):
            for i in range(nx):
                hits = 0
                for _ in range(sample_count):
                    # Generate random offset within voxel
                    u, v, w = np.random.random(3)
                    origin = np.array([i + u, j + v, k + w]) * voxel_size
                    
                    # Generate random direction uniformly over sphere
                    theta = np.random.uniform(0, 2 * np.pi)
                    phi = np.arccos(np.random.uniform(-1, 1))
                    direction = np.array([
                        np.sin(phi) * np.cos(theta),
                        np.sin(phi) * np.sin(theta),
                        np.cos(phi)
                    ])
                    
                    # Ray-mesh intersection
                    si = mesh.ray_intersect(origin, direction)
                    is_valid = si.is_valid()
                    is_target = si.shape == mesh
                    front_face = np.dot(direction, si.normal) > 0
                    
                    if is_valid and is_target and front_face:
                        hits += 1
                count_in[k, j, i] = hits
                
    # 3. Volume Calculation
    volumes = (count_in / sample_count) * voxel_volume
    return volumes

def analytic_voxelization(object_qc, grid_bounds, resolution):
    """
    Voxelize a continuous parametric object into a 3D grid.
    
    Args:
        object_qc: Callable representing the continuous object.
                   Must support `volume_intersection(voxel_bounds)`.
        grid_bounds: Tuple (min_x, min_y, min_z, max_x, max_y, max_z).
        resolution: Tuple (nx, ny, nz).
        
    Returns:
        3D numpy array of shape (nz, ny, nx) with values in [0, 1].
    """
    nx, ny, nz = resolution
    min_x, min_y, min_z, max_x, max_y, max_z = grid_bounds
    
    dx = (max_x - min_x) / nx
    dy = (max_y - min_y) / ny
    dz = (max_z - min_z) / nz
    voxel_volume = dx * dy * dz
    
    grid = np.zeros((nz, ny, nx))
    
    for k in range(nz):
        z_min = min_z + k * dz
        z_max = z_min + dz
        for j in range(ny):
            y_min = min_y + j * dy
            y_max = y_min + dy
            for i in range(nx):
                x_min = min_x + i * dx
                x_max = x_min + dx
                
                voxel_bounds = (x_min, y_min, z_min, x_max, y_max, z_max)
                intersection_vol = object_qc.volume_intersection(voxel_bounds)
                
                membership = min(1.0, max(0.0, intersection_vol / voxel_volume))
                grid[k, j, i] = membership
                
    return grid
