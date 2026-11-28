# Progressive Partitioning for Large-Scale 3DGS

Also called: Progressive Data Partitioning

## Overview
Progressive Partitioning is a data distribution strategy for training 3D Gaussian Splatting on large-scale scenes by dividing the environment into spatially-disjoint cells, each containing a subset of training cameras and 3D Gaussians [S1, S2]. It solves the memory and time bottlenecks of global optimization by enabling parallel, lower-memory training per cell [S2]. It distinguishes itself from naive spatial splitting by using an airspace-aware visibility criterion to ensure sufficient supervision of the volume above the surface, preventing floaters and under-supervision [S2].

## How it works
**Inputs:**
- Point cloud $P$ (initially empty or sparse).
- Set of training views $V$.
- Scene dimensions defining the ground plane.
- Grid dimensions $m \times n$ (number of cells along two axes).
- Visibility threshold $T_h$.
- Boundary expansion ratio $\alpha$ (typically 0.2 or 20%) [S2].

**Outputs:**
- A set of $m \times n$ cells.
- For each cell $j$: a subset of cameras $V_j$ and a subset of points $P_j$.

**Data Structures:**
- **Cell $j$**: Defined by a rectangular region on the ground plane with height $h_j$ and width $w_j$.
- **Point Cloud $P_j$**: List of 3D Gaussian parameters (position, covariance, opacity, spherical harmonics) associated with cell $j$.
- **Camera Set $V_j$**: List of camera poses and intrinsics assigned to cell $j$.

**Parameters:**
- $T_h$: Visibility threshold. Typical value: 25% [S2].
- $\alpha$: Boundary expansion factor. Typical value: 0.2 [S2].
- $m, n$: Grid dimensions. Example: $m=4, n=2$ for 8 cells [S2].

**Steps:**

1. **Camera-Position-Based Region Division**
   Partition the ground plane into a grid of $m \times n$ cells.
   - Divide the ground plane into $m$ sections along one axis (e.g., X), such that each section contains approximately $|V|/m$ views based on camera positions.
   - Subdivide each section into $n$ segments along the other axis (e.g., Y), such that each segment contains approximately $|V|/(m \times n)$ views.
   - Let the $j$-th cell be bounded by a rectangle of size $h_j \times w_j$ [S2].

2. **Position-Based Data Selection (Boundary Expansion)**
   Expand the boundaries of each cell to include context from neighboring areas.
   - Calculate expanded dimensions:
     $$ h'_j = h_j + \alpha h_j $$
     $$ w'_j = w_j + \alpha w_j $$
   - Assign training views $V$ to $V_j$ if the camera position falls within the expanded boundaries of cell $j$.
   - Assign points $P$ to $P_j$ if the point position falls within the expanded boundaries of cell $j$.
   - Note: Initially, $P$ may be empty or sparse; this step primarily assigns views [S2].

3. **Visibility-Based Camera Selection**
   Refine the camera set $V_j$ for each cell to ensure adequate supervision of the airspace.
   - For each camera $C_i$ not yet selected for cell $j$:
     - Calculate the projected area of the cell's volume in the image plane of $C_i$.
     - **Airspace-Aware Calculation**: Construct an axis-aligned bounding box (AABB) for the points currently in $P_j$ (or the cell's spatial extent). Project this AABB onto the image plane to get area $\Omega_{ij}$.
     - Calculate visibility:
       $$ \text{Visibility} = \frac{\Omega_{ij}}{\Omega_i} $$
       where $\Omega_i$ is the total pixel area of image $I_i$.
     - If $\text{Visibility} > T_h$, add $C_i$ to $V_j$.
   - **Pitfall Avoidance**: Do not use only the convex hull of surface points ($\Omega_{surf, ij}$) for $\Omega_{ij}$, as this is airspace-agnostic and leads to under-supervision of the volume above the surface and floaters [S2]. Use the AABB of the cell's point cloud extent to capture the full volume [S2].

4. **Coverage-Based Point Selection**
   Initialize $P_j$ with points that are well-constrained by the selected cameras.
   - Identify points in the global point cloud $P$ that are covered by **all** views in $V_j$.
   - Add these points to $P_j$.
   - This provides better initialization for optimization, reducing depth ambiguity and preventing the creation of floaters for objects outside the cell [S2].

5. **Optimization and Cleanup**
   - Optimize the 3D Gaussians in $P_j$ using only cameras in $V_j$.
   - After optimization, remove any 3D Gaussians generated outside the cell's boundaries to prevent artifacts (floaters) in adjacent cells [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_partitioning(point_cloud, views, m, n, threshold=0.25, expansion=0.2):
    """
    point_cloud: List of 3D points (initially sparse or empty)
    views: List of camera objects with position and intrinsics
    m: Number of sections along axis 1
    n: Number of segments along axis 2
    threshold: Visibility threshold Th
    expansion: Boundary expansion ratio alpha
    """
    # Step 1: Camera-Position-Based Region Division
    # Sort views by position along axis 1 and axis 2 to define grid boundaries
    # This creates m*n cells, each with bounds (h_j, w_j)
    cells = create_grid_cells(views, m, n)
    
    # Step 2: Position-Based Data Selection
    # Initialize V_j and P_j for each cell
    cell_data = {}
    for j, cell in enumerate(cells):
        # Expanded boundaries
        h_exp = cell.h * (1 + expansion)
        w_exp = cell.w * (1 + expansion)
        
        # Assign views based on camera position within expanded bounds
        V_j = [v for v in views if is_in_expanded_bounds(v.position, cell.bounds, h_exp, w_exp)]
        
        # Assign initial points (if any) based on position
        P_j = [p for p in point_cloud if is_in_expanded_bounds(p.position, cell.bounds, h_exp, w_exp)]
        
        cell_data[j] = {'V': V_j, 'P': P_j, 'bounds': cell.bounds, 'h': cell.h, 'w': cell.w}

    # Step 3: Visibility-Based Camera Selection
    for j, data in cell_data.items():
        V_j = data['V']
        P_j = data['P']
        bounds = data['bounds']
        h = data['h']
        w = data['w']
        
        # Identify cameras not yet in V_j
        remaining_views = [v for v in views if v not in V_j]
        
        for v in remaining_views:
            # Calculate projected area of the cell's volume in v's image
            # Use AABB of P_j (or cell bounds if P_j is empty)
            # Construct AABB from P_j points or cell bounds
            if P_j:
                aabb = get_aabb(P_j)
            else:
                # Fallback to cell bounds if no points yet
                aabb = get_aabb_from_bounds(bounds, h, w)
            
            projected_area = project_area_to_image(aabb, v)
            image_area = v.image_width * v.image_height
            
            visibility = projected_area / image_area
            
            if visibility > threshold:
                V_j.append(v)
        
        data['V'] = V_j

    # Step 4: Coverage-Based Point Selection
    for j, data in cell_data.items():
        V_j = data['V']
        P_j = data['P']
        
        # Find points covered by ALL views in V_j
        # This requires depth consistency or projection checks
        new_points = []
        for p in point_cloud:
            if is_covered_by_all_views(p, V_j):
                new_points.append(p)
        
        # Add to P_j for better initialization
        data['P'].extend(new_points)

    return cell_data

def is_covered_by_all_views(point, views):
    """
    Check if a 3D point is visible in all provided views.
    This is a simplified check; in practice, it involves projection and depth testing.
    """
    for v in views:
        if not is_visible_in_view(point, v):
            return False
    return True

def is_visible_in_view(point, view):
    """
    Check if point is within view frustum and not occluded.
    Placeholder for specific visibility logic.
    """
    # Project point to image plane
    uv = project_3d_to_2d(point, view)
    # Check if within image bounds
    if not (0 <= uv.x < view.image_width and 0 <= uv.y < view.image_height):
        return False
    # Check depth (simplified)
    # In 3DGS, this might involve checking against existing geometry or depth buffer
    return True
```

## Operating characteristics
- **Time Complexity**: Training time is reduced because each cell is optimized independently and in parallel. The total time is roughly proportional to the time for one cell, rather than the global scene [S2].
- **Memory Complexity**: Memory usage per cell is lower, as each cell only stores a subset of Gaussians and cameras. This allows training on hardware with limited VRAM [S2].
- **Performance**: In experiments with 8 cells, visibility threshold of 25%, and 60,000 iterations per cell, the method achieved high-fidelity reconstruction on large-scale scenes (e.g., Rubble, etc.) [S2].
- **Strengths**: Enables large-scale reconstruction on consumer hardware; reduces floaters and airspace artifacts through airspace-aware visibility; provides better initialization via coverage-based point selection.
- **Limitations**: Requires careful tuning of grid dimensions ($m, n$) and visibility threshold ($T_h$); may introduce seams at cell boundaries if not handled by merging techniques (like those in VastGaussian) [S2].

## Background
Progressive Partitioning was introduced by Lin et al. in the VastGaussian paper (CVPR 2024) as a strategy to adapt 3D Gaussian Splatting for large-scale scenes [S1, S2]. It addresses the limitations of global optimization, such as high memory consumption and long training times, by partitioning the scene into manageable cells [S1, S2]. The method builds on the idea of distributed learning and feature encoding for large scenes [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] VastGaussian: Vast 3D Gaussians for Large Scene Reconstruction, https://doi.org/10.1109/cvpr52733.2024.00494, fetched 2026-10-26
