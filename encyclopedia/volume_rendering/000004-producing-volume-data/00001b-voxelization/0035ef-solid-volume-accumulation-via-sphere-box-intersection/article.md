# Solid Volume Accumulation via Sphere-Box Intersection

Also called: Accumulate Clipped Solid Volume (NumPy), Solid Volume Accumulation

## Overview
This method voxelizes a set of spheres by accumulating the exact geometric volume of the intersection between each sphere and the voxels it overlaps. It solves the problem of converting point-cloud data with radii into a dense or sparse volumetric representation where voxel values represent the filled volume of the underlying primitives. It differs from [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by using analytical/numerical integration rather than sampling, and from [Point Cloud Voxelization](1000-point-cloud-voxelization.md) by explicitly handling sphere geometry rather than simple point occupancy.

## How it works
**Inputs:**
- `points`: List of sphere centers $(c_x, c_y, c_z)$.
- `radii`: List of sphere radii $r$.
- `mins`: World-space origin of the grid $(min_x, min_y, min_z)$.
- `voxel_size`: Physical size of a voxel edge $s$.
- `dims`: Grid dimensions $(D_x, D_y, D_z)$.
- `unique_linear`: List of linear indices for occupied voxels (for sparse/compact storage).
- `box_min`, `box_max`: World-space bounds for each voxel in `unique_linear`.
- `order`: Quadrature order for numerical integration.

**Outputs:**
- `solid_by_voxel`: Array or dictionary mapping voxel indices to accumulated intersection volume.

**Data Structures:**
- **Compact Index Map**: A dictionary mapping global linear voxel indices to compact array indices $0 \dots N-1$.
  $$ \text{linear\_to\_compact} = \{ \text{linear}_i : i \mid i \in 0 \dots N-1 \} $$
- **Voxel Bounds**: Pre-computed world-space $[box\_min, box\_max]$ for each voxel in the output structure.

**Steps:**
1. **Generate Quadrature Nodes**: Obtain 1D Gauss-Legendre nodes $w_k$ and weights $q_k$ for the specified `order`. If unavailable, use a standard Legendre-Gauss implementation [S1, unofficial].
2. **Initialize Storage**: Create `solid_by_voxel` of size $N$ (number of unique linear indices) initialized to 0.0.
3. **Iterate Points**: For each sphere with center $C$ and radius $r$:
    a. Calculate bounding voxel range in grid coordinates:
       $$ \text{low}_d = \lfloor \frac{C_d - r - \text{mins}_d}{s} \rfloor $$
       $$ \text{high}_d = \lfloor \frac{C_d + r - \text{mins}_d}{s} \rfloor $$
       Clamp $\text{low}_d \ge 0$ and $\text{high}_d \le D_d - 1$ for each axis $d \in \{x, y, z\}$.
    b. Iterate over all voxel indices $(i, j, k)$ in the range $[\text{low}_x, \text{high}_x] \times [\text{low}_y, \text{high}_y] \times [\text{low}_z, \text{high}_z]$.
    c. Compute linear index:
       $$ L = i + D_x(j + D_y k) $$
    d. Look up compact index $c = \text{linear\_to\_compact}.get(L)$. If $c$ is `None`, skip.
    e. Retrieve voxel bounds $B_{min}, B_{max}$ for compact index $c$.
    f. Compute intersection volume $V_{int}$ between sphere $(C, r)$ and box $[B_{min}, B_{max}]$ using the function below.
    g. Accumulate: `solid_by_voxel[c] += V_int`.

**Sphere-Box Intersection Volume Function:**
Given sphere center $C$, radius $r$, and box bounds $[B_{min}, B_{max}]$:
1. **Clipping**:
   $$ S_{min} = C - r, \quad S_{max} = C + r $$
   $$ C_{min} = \max(B_{min}, S_{min}) $$
   $$ C_{max} = \min(B_{max}, S_{max}) $$
2. **Early Exits**:
   - If any axis has $C_{max}[d] \le C_{min}[d]$, return $0.0$ (no overlap).
   - If the sphere is fully inside the box ($B_{min} \le S_{min}$ and $B_{max} \ge S_{max}$), return $\frac{4}{3}\pi r^3$.
3. **Numerical Integration**:
   - Define local coordinates relative to sphere center:
     $$ x_0 = C_{min}[x] - C_x, \quad x_1 = C_{max}[x] - C_x $$
     $$ y_0 = C_{min}[y] - C_y, \quad y_1 = C_{max}[y] - C_y $$
     $$ z_0 = C_{min}[z] - C_z, \quad z_1 = C_{max}[z] - C_z $$
   - Compute midpoints and half-widths for the clipped box region in XY:
     $$ x_{mid} = \frac{x_0 + x_1}{2}, \quad x_{half} = \frac{x_1 - x_0}{2} $$
     $$ y_{mid} = \frac{y_0 + y_1}{2}, \quad y_{half} = \frac{y_1 - y_0}{2} $$
   - Initialize `total = 0.0`.
   - For each node $u$ with weight $w_u$ in 1D quadrature:
     - For each node $v$ with weight $w_v$ in 1D quadrature:
       - Map to physical XY:
         $$ x = x_{mid} + x_{half} \cdot u $$
         $$ y = y_{mid} + y_{half} \cdot v $$
       - Calculate remaining radial capacity in Z:
         $$ \text{remaining} = r^2 - x^2 - y^2 $$
       - If $\text{remaining} \le 0$, continue (point is outside sphere radius in XY projection).
       - Calculate Z-radius:
         $$ z_{rad} = \sqrt{\text{remaining}} $$
       - Calculate valid Z-height within the clipped box:
         $$ h = \max(0, \min(z_1, z_{rad}) - \max(z_0, -z_{rad})) $$
       - Accumulate:
         $$ \text{total} += w_u \cdot w_v \cdot h $$
   - Return $x_{half} \cdot y_{half} \cdot \text{total}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sphere_box_intersection_volume(center, radius, box_min, box_max, nodes, weights):
    # 1. Sphere bounds
    sphere_min = [center[i] - radius for i in range(3)]
    sphere_max = [center[i] + radius for i in range(3)]
    
    # 2. Clipped intersection box
    clipped_min = [max(box_min[i], sphere_min[i]) for i in range(3)]
    clipped_max = [min(box_max[i], sphere_max[i]) for i in range(3)]
    
    # No overlap
    if any(clipped_max[i] <= clipped_min[i] for i in range(3)):
        return 0.0
        
    # Sphere fully inside box
    if all(box_min[i] <= sphere_min[i] and box_max[i] >= sphere_max[i] for i in range(3)):
        return (4.0 / 3.0) * math.pi * radius ** 3
        
    # 3. Numerical Integration
    x0, y0, z0 = [clipped_min[i] - center[i] for i in range(3)]
    x1, y1, z1 = [clipped_max[i] - center[i] for i in range(3)]
    
    x_mid = 0.5 * (x0 + x1)
    y_mid = 0.5 * (y0 + y1)
    x_half = 0.5 * (x1 - x0)
    y_half = 0.5 * (y1 - y0)
    
    total = 0.0
    for node_x, weight_x in zip(nodes, weights):
        x = x_mid + x_half * node_x
        for node_y, weight_y in zip(nodes, weights):
            y = y_mid + y_half * node_y
            remaining = radius * radius - x * x - y * y
            if remaining <= 0.0:
                continue
            z_radius = math.sqrt(remaining)
            # Height of the sphere segment within the clipped Z range
            height = min(z1, z_radius) - max(z0, -z_radius)
            if height > 0.0:
                total += weight_x * weight_y * height
                
    return x_half * y_half * total

def accumulate_clipped_solid_numpy(points, radii, mins, voxel_size, dims, unique_linear, box_min, box_max, order):
    # Get quadrature nodes/weights
    nodes, weights = make_sphere_clip_quadrature(order) # [G] Standard Legendre-Gauss implementation
    
    # Map linear indices to compact indices
    linear_to_compact = {int(linear): i for i, linear in enumerate(unique_linear)}
    solid_by_voxel = [0.0] * len(unique_linear)
    
    for center, radius in zip(points, radii):
        # Calculate bounding voxel range
        low = [int(math.floor((center[i] - radius - mins[i]) / voxel_size)) for i in range(3)]
        high = [int(math.floor((center[i] + radius - mins[i]) / voxel_size)) for i in range(3)]
        
        # Clamp to grid dimensions
        low = [max(0, low[i]) for i in range(3)]
        high = [min(dims[i] - 1, high[i]) for i in range(3)]
        
        # Iterate over bounding box
        for ix in range(low[0], high[0] + 1):
            for iy in range(low[1], high[1] + 1):
                for iz in range(low[2], high[2] + 1):
                    linear = ix + dims[0] * (iy + dims[1] * iz)
                    compact = linear_to_compact.get(int(linear))
                    if compact is None:
                        continue
                    
                    solid_by_voxel[compact] += sphere_box_intersection_volume(
                        center, float(radius), 
                        box_min[compact], box_max[compact], 
                        nodes, weights
                    )
    return solid_by_voxel
```

**Parameters:**
- `order`: Quadrature order. Higher orders increase accuracy for partial intersections. Not given in the sources.

**Edge Cases:**
- **Precision**: Floating point comparisons for `remaining <= 0.0` and boundary checks should handle epsilon errors if extreme precision is required [G].
- **Degenerate Input**: If `voxel_size` is 0 or `dims` are invalid, the method fails. Clamping prevents out-of-bounds access.

## Subtopics

### [Sphere-Box Intersection Volume via Gaussian Quadrature](8375-sphere-box-intersection-volume-via-gaussian-quadrature.md)
Computes the volume of the intersection between a sphere and an axis-aligned bounding box using numerical integration. It handles cases where the sphere is fully inside, fully outside, or partially intersecting the box. Choose this when exact volume integration is required for partial voxel overlaps.

## Operating characteristics
- **Time Complexity**: $O(N \cdot M \cdot Q^2)$, where $N$ is the number of points, $M$ is the average number of voxels per point's bounding box, and $Q$ is the quadrature order.
- **Memory Complexity**: $O(K)$, where $K$ is the number of unique occupied voxels.
- **Strengths**: Provides exact geometric volume accumulation for spheres, avoiding aliasing artifacts common in rasterization. Efficiently skips empty space using bounding boxes.
- **Limitations**: Slower than simple occupancy methods due to numerical integration. Quadrature order affects performance.
- **When to use**: When accurate density/volume representation is critical for sphere-based point clouds.
- **When not to use**: For real-time applications requiring extreme speed, or for non-spherical primitives.

## Background
Originated in the context of point-cloud voxelization libraries, specifically referenced in `SammySuliman/point-cloud-voxelization` [S1, unofficial]. It is used to generate solid volume data from spherical primitives, often as a preprocessing step for volume rendering or physics simulation.

## Sources
[S1] SammySuliman/point-cloud-voxelization, https://github.com/SammySuliman/point-cloud-voxelization, fetched 2026-10-27 (unofficial)
