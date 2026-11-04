# Differentiable Voxelization (MC/CF)

Also called: Closed-form Voxelization, Convolution Filter Voxelization, Differentiable 3D Voxelization (Forward/Backward Mode), VoxelizeFunc

## Overview
Differentiable Voxelization (MC/CF) is a PyTorch autograd function that rasterizes 2D polygons or 3D triangle meshes into a grid of winding numbers (occupancy), supporting both Marching Cubes (MC) and Convolution Filter (CF) methods with differentiable backward passes [S1, unofficial]. It sets itself apart from [Monte Carlo 2D Voxelization](1734-monte-carlo-2d-voxelization.md) by providing closed-form analytic gradients for both 2D and 3D primitives, and from [Analytic Moment Voxelization](3307-analytic-moment-voxelization.md) by focusing on binary/signed occupancy grids rather than higher-order polynomial moments [S1, unofficial].

## How it works
**Inputs:**
- `vertices`: Array of shape $(N, D)$ containing vertex positions ($D=2$ for 2D, $D=3$ for 3D), type Float.
- `simplices` (or `faces`/`edges`): Array of shape $(F, D)$ containing vertex indices for triangles (3D) or edges (2D), type `uint32_t`.
- `grid_shape`: Tuple $(H, W)$ for 2D or $(D, H, W)$ for 3D.
- `method`: String `'mc'` or `'cf'`. If `'auto'`, selects `'cf'` if `filter_radius == half_voxel_size`, otherwise `'mc'` [S1, unofficial].
- `num_samples_per_voxel`: Number of samples per voxel for MC forward pass. Default: 16 [S1, unofficial].
- `num_samples_per_simplex`: Number of samples per simplex for MC backward pass. Default: 64 [S1, unofficial].
- `filter_radius`: Radius of the filter. Default: half voxel size. For CF, must equal half voxel size to have effect [S1, unofficial].

**Outputs:**
- `occupancy`: Output tensor of shape `grid_shape`, type Float, containing winding numbers.
- `d_vertices`: Gradient tensor for vertices, shape $(N, D)$, type Float.
- `d_occupancy`: Gradient tensor for occupancy, shape `grid_shape`, type Float.

**Data Structures:**
The implementation uses a grid coordinate system where the world space $[-1, 1]^D$ is mapped to grid indices.
- `step_x = 2.0 / depth` (or width for 2D)
- `step_y = 2.0 / height`
- `step_z = 2.0 / width` (for 3D)
- Grid center optimization uses `center_k = width / 2` [S1, unofficial].

**Primal (Forward) Pass:**
1. Validate inputs using `validate_common_voxelize_arguments` [S1, unofficial].
2. Dispatch to 2D or 3D closed-form implementation based on dimension [S1, unofficial].
3. Initialize `occupancy` as a zero tensor [S1, unofficial].
4. For each face (triangle or edge):
   - Extract vertices $v_0, v_1, v_2$ (or $v_0, v_1$ for 2D).
   - Compute edge vectors $e_1 = v_1 - v_0$, $e_2 = v_2 - v_0$.
   - Compute normal $n = \text{normalize}(\text{cross}(e_1, e_2))$ and area $A = 0.5 \|\text{cross}(e_1, e_2)\|$ [S1, unofficial].
   - Determine bounding box in grid coordinates:
     $$ \text{bbox\_min}_i = \max(0, \lfloor \frac{\min(v_{0,i}, v_{1,i}, v_{2,i}) + 1}{\text{step}_i} \rfloor) $$
     $$ \text{bbox\_max}_i = \min(\text{dim\_size}_i - 1, \lfloor \frac{\max(v_{0,i}, v_{1,i}, v_{2,i}) + 1}{\text{step}_i} \rfloor) $$
     [S1, unofficial].
   - **MC Method:** Perform Monte Carlo integration. Generate uniform $u, v$. If $u+v > 1$, reflect: $u = 1-u, v = 1-v$. Compute sample point $p = v_0 + u e_1 + v e_2$. Convert to grid coordinates. Iterate over neighbor voxels. Compute filter weight. Update `occupancy`:
     $$ \text{occupancy}[idx] += \text{filter\_weight} \times \frac{A}{\text{num\_samples\_per\_simplex}} $$
     [S1, unofficial].
   - **CF Method:** Use closed-form polygon splitting.
     - Represent the triangle in barycentric coordinates.
     - Iterate over $x$-slices (planes $x = \text{const}$). Split the polygon barycentrically by each plane $x_i$.
     - For each $x$-slice, iterate over $y$-slices. Split the resulting polygon by $y_j$.
     - For each $z$-slice (or final 2D cell), compute the intersection polygon $P_{xyz}$ in barycentric coordinates.
     - Compute the winding number contribution. For a convex polygon $P$ with vertices $b_k$ in barycentric space, the contribution to the voxel at index $(i,j,k)$ is:
       $$ \text{contrib} = \frac{1}{\text{cell\_volume}} \sum_{\text{triangles in } P} \text{sub\_area} \times (\text{pivot} + b_1 + b_2) $$
       where `pivot` is the centroid of the original triangle barycentric coordinates ($1/3, 1/3, 1/3$) [S1, unofficial].
     - Update `occupancy[idx] += contrib` [S1, unofficial].

**Backward Pass:**
1. Select backward function based on method and dtype [S1, unofficial].
2. Initialize `d_vertices` as zeros [S1, unofficial].
3. For each face:
   - Compute normal velocities for backward pass:
     $$ d\_xbn\_d\_v_0 = n \times (1 - u - v) $$
     $$ d\_xbn\_d\_v_1 = n \times u $$
     $$ d\_xbn\_d\_v_2 = n \times v $$
     [S1, unofficial].
   - **Forward Mode Gradient:** If `d_vertices` is provided as input (forward-mode differentiation), compute $d\_xbn$ by dotting vertex gradients with normal velocities:
     $$ d\_xbn = \text{dot}(d\_v_0, d\_xbn\_d\_v_0) + \text{dot}(d\_v_1, d\_xbn\_d\_v_1) + \text{dot}(d\_v_2, d\_xbn\_d\_v_2) $$
     [S1, unofficial].
   - **Backward Mode Gradient:** If `d_occupancy` is provided as input (backward-mode differentiation):
     - Iterate over affected voxels.
     - Compute `differential_weight = d_occupancy[idx] / cell_volume` [S1, unofficial].
     - For CF method, compute the geometric contribution vector `contrib` (same as primal but using barycentric coordinates) [S1, unofficial].
     - Update vertex gradients:
       $$ d\_vertices[3 \cdot v_k + d] += \text{contrib}[k] \times d\_xbn\_d\_v_k[d] \times \text{differential\_weight} $$
       for $k \in \{0, 1, 2\}$ and $d \in \{0, 1, 2\}$ [S1, unofficial].
     - For MC method, update gradients using the filter weight and area:
       $$ d\_vertices[3 \cdot v_k + d] += d\_xbn\_d\_v_k[d] \times \text{filter\_weight} \times \frac{A}{\text{num\_samples\_per\_simplex}} \times d\_occupancy[idx] $$
       [S1, unofficial].

**Pseudocode (Python-like):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def voxelize_cf_3d_backward(vertices, faces, occupancy, d_occupancy, grid_shape):
    """
    Simplified backward pass for CF voxelization.
    vertices: (N, 3)
    faces: (F, 3)
    occupancy: (D, H, W)
    d_occupancy: (D, H, W)
    """
    N, D = vertices.shape
    d_vertices = torch.zeros_like(vertices)
    
    step_x = 2.0 / grid_shape[0]
    step_y = 2.0 / grid_shape[1]
    step_z = 2.0 / grid_shape[2]
    cell_volume = step_x * step_y * step_z
    
    for face_idx in range(faces.shape[0]):
        i0, i1, i2 = faces[face_idx]
        v0 = vertices[i0]
        v1 = vertices[i1]
        v2 = vertices[i2]
        
        e1 = v1 - v0
        e2 = v2 - v0
        normal = torch.cross(e1, e2)
        normal_len = torch.norm(normal)
        
        if normal_len < 1e-12:
            continue
            
        unit_normal = normal / normal_len
        area = 0.5 * normal_len
        
        # Bounding box
        min_x = torch.min(v0[0], torch.min(v1[0], v2[0]))
        max_x = torch.max(v0[0], torch.max(v1[0], v2[0]))
        # ... compute min_y, max_y, min_z, max_z similarly
        
        bbox_min_i = max(0, int((min_x + 1) / step_x))
        bbox_max_i = min(grid_shape[0] - 1, int((max_x + 1) / step_x))
        # ... compute other bbox bounds
        
        # Barycentric coordinates of triangle
        # Triangle in barycentric space: (1,0,0), (0,1,0), (0,0,1)
        # We split this polygon by planes x=const, y=const, z=const
        
        # Pseudocode for polygon splitting is complex; 
        # assuming a helper `split_polygon_bary` exists which returns 
        # a list of polygons in barycentric coords.
        
        remaining_poly = [torch.tensor([1.0, 0.0, 0.0]), 
                          torch.tensor([0.0, 1.0, 0.0]), 
                          torch.tensor([0.0, 0.0, 1.0])]
        
        for i in range(bbox_min_i, bbox_max_i + 1):
            x_hi = -1.0 + (i + 1) * step_x
            # Split remaining_poly by plane x = x_hi
            # This requires projecting barycentric coords to Cartesian x
            # x = dot(bary, tri_x) where tri_x = [v0.x, v1.x, v2.x]
            tri_x = torch.tensor([v0[0], v1[0], v2[0]])
            
            below_poly, above_poly = split_polygon_bary(remaining_poly, tri_x, x_hi)
            remaining_poly = above_poly
            
            if len(below_poly) < 3:
                continue
                
            # For each slice in below_poly, iterate y
            # ... (nested loops for y and z)
            
            # For each voxel (i, j, k) intersected:
            # Compute intersection polygon in barycentric coords
            # Compute contribution
            # Update d_vertices
            
            # Simplified update for one voxel:
            # contrib = compute_barycentric_contribution(barycentric_polygon)
            # differential_weight = d_occupancy[i, j, k] / cell_volume
            
            # d_vertices[i0] += contrib[0] * unit_normal * differential_weight
            # d_vertices[i1] += contrib[1] * unit_normal * differential_weight
            # d_vertices[i2] += contrib[2] * unit_normal * differential_weight
            
            pass
            
    return d_vertices

def split_polygon_bary(polygon_bary, tri_coords, value):
    """
    Split a polygon defined by barycentric vertices by a Cartesian plane.
    Returns (below, above) polygons in barycentric coordinates.
    """
    below = []
    above = []
    n = len(polygon_bary)
    
    for i in range(n):
        curr = polygon_bary[i]
        next_v = polygon_bary[(i + 1) % n]
        
        curr_val = torch.dot(curr, tri_coords)
        next_val = torch.dot(next_v, tri_coords)
        
        curr_below = curr_val <= value
        next_below = next_val <= value
        
        if curr_below:
            below.append(curr)
        else:
            above.append(curr)
            
        if curr_below != next_below:
            d = next_val - curr_val
            if abs(d) > 1e-12:
                t = (value - curr_val) / d
                intersection = curr + t * (next_v - curr)
                below.append(intersection)
                above.append(intersection)
                
    return below, above
```

## Subtopics

### [Differentiable Voxelization via Smoothed Winding Numbers](2325-differentiable-voxelization-via-smoothed-winding-numbers.md)
A method for voxelizing triangle meshes into smoothed winding number voxel grids that provides closed-form gradients with respect to mesh vertex positions, enabling gradient-based optimization of surface representations based on volumetric properties. It differs from the standard CF method by smoothing the winding number field, which can improve gradient stability for noisy or non-manifold inputs.

### [Differentiable Voxelization from Radiative Gaussians](2380-differentiable-voxelization-from-radiative-gaussians.md)
A differentiable voxelizer $V$ to efficiently query a density volume $V \in \mathbb{R}^{X \times Y \times Z}$ from radiative Gaussians, used for 3D regularization. It extends the voxelization concept to Gaussian splats rather than triangle meshes, allowing gradients to flow from the voxel grid back to the Gaussian parameters (position, covariance, opacity).

### Backward Differentiable Closed-form Voxelization
Computes backward-mode gradients of voxel occupancy with respect to vertex positions using closed-form integration [S1, unofficial]. It takes `d_occupancy` as input and produces `d_vertices` as output. It validates inputs using `validate_common_differential_voxelize_arguments` and dispatches to 2D or 3D backward-mode implementations [S1, unofficial].

### Forward Differentiable Closed-form Voxelization
Computes forward-mode gradients, taking `d_vertices` as input and producing `d_occupancy` as output [S1, unofficial]. It validates inputs and dispatches to 2D or 3D forward-mode implementations [S1, unofficial].

### 2D Polygon Rasterization to Grid Moments
Rasterizes a 2D polygon to a destination grid by computing polynomial density moments over the covered cells [S2, unofficial]. It supports polynomial orders 0 (constant), 1 (linear), and 2 (quadratic) [S2, unofficial]. The destination grid is a row-major array of moments with size $(n_x \cdot n_y) \cdot R2D\_NUM\_MOMENTS(polyorder)$ [S2, unofficial]. The grid patch starts at `d*ibox[0]` and ends at `d*ibox[1]` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(F \cdot B)$, where $F$ is the number of faces and $B$ is the average number of voxels in the bounding box of a face. The CF method involves polygon splitting, which can increase the constant factor compared to simple rasterization, but avoids sampling noise.
- **Memory Complexity:** $O(V)$ for the output grid, plus temporary storage for polygon splitting $O(F \cdot B)$ [S1, unofficial].
- **Performance:** The implementation is CPU-only. GPU tensors are automatically moved to CPU for processing and moved back, issuing a warning [S1, unofficial]. The CF method assumes a box filter the size of a voxel. If `filter_radius` does not equal half voxel size, the CF method has no effect and falls back to MC or is invalid [S1, unofficial].
- **Strengths:** Provides exact, noise-free gradients for triangle meshes. Supports both 2D and 3D.
- **Limitations:** CPU-only implementation limits scalability for large meshes. The CF method's coordinate convention (width, height vs height, width) requires a transpose of the output voxels, which regresses performance [S1, unofficial].
- **When to use:** When differentiable rasterization of triangle meshes is required for optimization (e.g., shape optimization, neural rendering).
- **When not to use:** When GPU acceleration is critical, or when working with Gaussian splats (use [Differentiable Voxelization from Radiative Gaussians](2380-differentiable-voxelization-from-radiative-gaussians.md) instead).

## Background
The method originates from the "Differentiable Voxelization of Surface Representations" repository by M. Worchel [S1, unofficial]. It implements closed-form voxelization algorithms for both 2D polygons and 3D triangle meshes, supporting both Marching Cubes and Convolution Filter variants. The 2D polygon rasterization to grid moments is based on the work of Devon Powell (r3d library) [S2, unofficial].

## Sources
[S1] Differentiable Voxelization of Surface Representations, https://github.com/mworchel/differentiable-voxelization, fetched 2026-10-29 (unofficial)
[S2] devonmpowell/r3d, https://github.com/devonmpowell/r3d, fetched 2026-10-27 (unofficial)
