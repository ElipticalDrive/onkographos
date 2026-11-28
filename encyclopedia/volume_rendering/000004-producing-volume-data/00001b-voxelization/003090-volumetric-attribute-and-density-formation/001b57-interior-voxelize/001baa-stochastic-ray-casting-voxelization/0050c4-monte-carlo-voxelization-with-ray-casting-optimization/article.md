# Monte Carlo Voxelization with Ray-Casting Optimization

Also called: Column-based Ray Sampling, Optimized Voxelization Algorithm

## Overview
This method approximates the geometric similarity between a voxel and a 3D object by sampling points within the voxel using rays, avoiding full volumetric intersection tests [S1, unofficial]. It builds on [Stochastic Ray-Casting Voxelization](1067-stochastic-ray-casting-voxelization.md) by organizing sampling into columns and optimizing the evaluation of occupancy and element membership, significantly reducing the computational cost of selecting the best approximating element from a palette [S1, unofficial].

## How it works
**Inputs:**
- A 3D triangle mesh representing the object surface [S1, unofficial].
- A voxel grid of dimensions $I \times J \times K$ [S1, unofficial].
- A palette $P$ of approximating volume elements (e.g., empty $E_0$, full $E_1$, and intermediate shapes) [S1, unofficial].
- Parameters: $t$ (number of rays per column), $p$ (samples per ray segment), constants $D$ and $E$ for empty/full checks [S1, unofficial].
- Scene bounds $[0, x_l] \times [0, y_l] \times [0, z_l]$ defining the voxel space $V$ [S1, unofficial].

**Outputs:**
- For each voxel $(i, j, k)$, the selected element $E_w \in P$ that maximizes the estimated similarity $\hat{B}_v$ [S1, unofficial].

**Data Structures:**
- **Intersection List:** An ordered set $\{z_0, \dots, z_{2n-1}\}$ of $z$-coordinates where the ray intersects the mesh, sorted such that $z_m < z_{m+1}$ [S1, unofficial].
- **Quad-tree:** A spatial acceleration structure for the mesh triangles, used to limit ray-triangle intersection tests to triangles whose projection intersects the leaf containing the ray [S1, unofficial].

**Steps:**
1. **Column Iteration:** Iterate over voxel columns by fixing indices $i$ and $j$, and varying $k$ from $0$ to $K-1$ [S1, unofficial].
2. **Ray Casting:** For each column $(i, j)$, cast $t$ rays parallel to the $z$-axis [S1, unofficial]. Each ray is identified by random $(x, y)$ coordinates sampled from the voxel's cross-section: $(x, y) \in U_r \subset [0, x_l] \times [0, y_l]$ [S1, unofficial].
3. **Intersection Computation:** Compute intersections of each ray with the object surface [S1, unofficial]. Store the $z$-coordinates in the ordered set $\{z_0, \dots, z_{2n-1}\}$ [S1, unofficial]. Due to the two-manifold closed surface assumption, the number of intersections is always even [S1, unofficial].
4. **Occupancy Determination:** For a point $z$ on the ray, the occupancy function $o(x, y, z)$ is defined as:
   $$o(x, y, z) = \begin{cases} 1 & \text{if } z_{2m} < z < z_{2m+1} \text{ for some } m \\ 0 & \text{otherwise} \end{cases}$$ [S1, unofficial].
5. **Segment Sampling:** For the current voxel $(i, j, k)$, identify the segment of the ray intersecting the voxel [S1, unofficial]. Sample $p$ random points within this segment to form the set $U$ for the current column [S1, unofficial]. The total number of samples is $|U| = t \times p$, where $t$ groups of $p$ points share the same $(x, y)$ coordinates [S1, unofficial].
6. **Similarity Estimation:** For each element $E_v \in P$, estimate the similarity $\hat{B}_v$ using the Monte Carlo approximation:
   $$\hat{B}_v = \frac{1}{|U|} \sum_{(x,y,z) \in U} g(e_v(x, y, z), o(x, y, z), x, y, z) \approx G \cdot B_v$$ [S1, unofficial].
   Here $e_v(x, y, z)$ is the membership function of element $E_v$ at point $(x, y, z)$, and $g$ is the metric function (e.g., Nearest Approximation) [S1, unofficial].
7. **Element Selection:** Select the element $E_w$ that maximizes the estimated similarity:
   $$B_w = \arg \max_{E_v \in P} \hat{B}_v$$ [S1, unofficial].
8. **Optimizations:** Apply the following optimizations to reduce computation [S1, unofficial]:
   - **Fast Full/Empty Check:** Before evaluating all palette elements, check if the voxel is likely full or empty [S1, unofficial]. Let $z_l$ and $z_u$ be the $z$-coordinates of the voxel's lower and upper faces [S1, unofficial]. If for all ray segments $z_{2m} < z_l < z_u < z_{2m+1}$, the voxel is full [S1, unofficial]. If for all segments $z_{2m+1} < z_l < z_u < z_{2m+2}$, the voxel is empty [S1, unofficial].
   - **Statistical Check:** If $\#o_1 < D|U|$, treat as empty; if $\#o_1 > E|U|$, treat as full, where $\#o_1$ is the count of points where $o(x, y, z) = 1$ [S1, unofficial].
   - **Best Outer/Inner Box:** For the Nearest Approximation metric, if $e_v(x, y, z) \neq o(x, y, z)$ for any point, set $\hat{B}_v = -\infty$ and stop checking other points for that element [S1, unofficial].
   - **Accelerated Intersection:** Use the quad-tree to limit ray-triangle intersection tests [S1, unofficial].
   - **Simplified Membership:** For convex polyhedrons, evaluate $e_v(x, y, z)$ using a combination of linear expressions (e.g., 2-4 constraints) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random

def monte_carlo_voxelization(mesh_triangles, grid_dims, palette, t, p, D, E, x_l, y_l, z_l):
    """
    Voxelize a mesh using Monte Carlo ray-casting optimization.
    Args:
        mesh_triangles: List of triangles, each defined by 3 vertices [(x0,y0,z0), (x1,y1,z1), (x2,y2,z2)].
        grid_dims: Tuple (I, J, K) defining voxel grid dimensions.
        palette: List of volume elements E_v, each with evaluate_membership(point) method.
        t: Number of rays per voxel column.
        p: Number of samples per ray segment within a voxel.
        D: Threshold constant for empty voxel check (0 < D << p_min).
        E: Threshold constant for full voxel check (p_max << E < 1).
        x_l, y_l, z_l: Scene bounds defining the voxel space V.
    Returns:
        voxel_grid: 3D array of selected elements from palette.
    """
    I, J, K = grid_dims
    dx, dy, dz = x_l / I, y_l / J, z_l / K
    voxel_grid = [[[None for _ in range(K)] for _ in range(J)] for _ in range(I)]

    # [G] Quad-tree construction omitted; standard spatial partitioning for triangle acceleration.

    for i in range(I):
        for j in range(J):
            x_min, x_max = i * dx, (i + 1) * dx
            y_min, y_max = j * dy, (j + 1) * dy

            rays = []
            for _ in range(t):
                x = random.uniform(x_min, x_max)
                y = random.uniform(y_min, y_max)
                rays.append((x, y))

            # Compute intersections for all rays in the column
            ray_data = []
            for (x, y) in rays:
                z_coords = sorted(intersect_z_axis_ray(x, y, mesh_triangles))
                ray_data.append((x, y, z_coords))

            for k in range(K):
                z_l_face = k * dz
                z_u_face = (k + 1) * dz

                # Fast Full/Empty geometric check (column-wide)
                col_is_full = True
                col_is_empty = True
                count_inside = 0
                total_samples = t * p

                for (x, y, z_coords) in ray_data:
                    # Check if voxel segment is fully inside or outside for this ray
                    ray_inside = False
                    ray_outside = False
                    for m in range(0, len(z_coords), 2):
                        if z_coords[m] < z_l_face and z_u_face < z_coords[m+1]:
                            ray_inside = True
                        if m + 2 < len(z_coords):
                            if z_coords[m+1] < z_l_face and z_u_face < z_coords[m+2]:
                                ray_outside = True
                    if not ray_inside:
                        col_is_full = False
                    if not ray_outside:
                        col_is_empty = False

                    # Sample points for statistical check
                    for _ in range(p):
                        z = random.uniform(z_l_face, z_u_face)
                        inside = False
                        for m in range(0, len(z_coords), 2):
                            if z_coords[m] < z < z_coords[m+1]:
                                inside = True
                                break
                        if inside:
                            count_inside += 1

                if col_is_full:
                    voxel_grid[i][j][k] = palette[0]  # E_1 (full)
                    continue
                if col_is_empty:
                    voxel_grid[i][j][k] = palette[-1]  # E_0 (empty)
                    continue

                # Statistical Check
                if count_inside < D * total_samples:
                    voxel_grid[i][j][k] = palette[-1]  # Empty
                    continue
                if count_inside > E * total_samples:
                    voxel_grid[i][j][k] = palette[0]  # Full
                    continue

                # Evaluate similarity for each element in palette
                best_element = None
                best_score = float('-inf')

                for element in palette:
                    score = 0
                    is_best = True

                    for (x, y, z_coords) in ray_data:
                        for _ in range(p):
                            z = random.uniform(z_l_face, z_u_face)
                            o_val = 0
                            for m in range(0, len(z_coords), 2):
                                if z_coords[m] < z < z_coords[m+1]:
                                    o_val = 1
                                    break
                            e_val = element.evaluate_membership((x, y, z))
                            if e_val == o_val:
                                score += 1
                            else:
                                is_best = False
                                break
                        if not is_best:
                            break

                    if is_best and score > best_score:
                        best_score = score
                        best_element = element

                voxel_grid[i][j][k] = best_element

    return voxel_grid

def intersect_z_axis_ray(x, y, triangles):
    # [G] Standard ray-triangle intersection for z-axis parallel rays.
    # For each triangle, check if (x,y) lies within its 2D projection.
    # If so, compute z via barycentric interpolation.
    hits = []
    for tri in triangles:
        x0, y0, z0 = tri[0]
        x1, y1, z1 = tri[1]
        x2, y2, z2 = tri[2]
        denom = (y1 - y2)*(x0 - x2) + (x2 - x1)*(y0 - y2)
        if abs(denom) < 1e-9:
            continue
        l1 = ((y1 - y2)*(x - x2) + (x2 - x1)*(y - y2)) / denom
        l2 = ((y2 - y0)*(x - x0) + (x0 - x2)*(y - y0)) / denom
        l0 = 1.0 - l1 - l2
        if 0 <= l0 <= 1 and 0 <= l1 <= 1 and 0 <= l2 <= 1:
            z_hit = l0*z0 + l1*z1 + l2*z2
            hits.append(z_hit)
    return hits
```

## Operating characteristics
- **Time Complexity:** $O(I J K p t |P|)$, where $I, J, K$ are the voxel grid dimensions, $p$ is samples per ray, $t$ is rays per column, and $|P|$ is the number of palette elements [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Strengths:** Significantly reduces computation by treating empty/full voxels specially and using statistical checks [S1, unofficial]. The column-based approach and ray-casting avoid expensive volumetric intersection tests [S1, unofficial].
- **Limitations:** Relies on stochastic sampling, so results may vary slightly between runs [S1, unofficial]. The quality depends on the number of samples $t$ and $p$ [S1, unofficial]. The method assumes a two-manifold closed surface for the even intersection count [S1, unofficial].
- **When to Use:** When voxelizing complex meshes with a palette of different shapes, and performance is critical [S1, unofficial]. Suitable for applications where approximate geometric similarity is sufficient [S1, unofficial].

## Background
The method was introduced by D. Cavagnino and M. Gribaudo in 2010 as an optimized version of their previous approach for discretizing 3D models using voxel elements of different shapes [S1, unofficial]. It builds on the ray-casting techniques used for anti-aliasing oversampling [TGR04], adapting them for Monte Carlo similarity estimation [S1, unofficial]. The algorithm was demonstrated on models such as a cow head, showing improved visual quality with varying voxel sizes and palette sizes [S1, unofficial].

## Sources
[S1] Discretization of 3D Models Using Voxel Elements of Different Shapes, https://doi.org/10.2312/compaesth/compaesth10/091-098, fetched 2026-10-28 (unofficial)
