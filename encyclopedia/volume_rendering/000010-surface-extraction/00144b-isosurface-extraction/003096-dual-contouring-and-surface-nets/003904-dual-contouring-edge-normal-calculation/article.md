# Dual Contouring Edge Normal Calculation

Also called: Dual Contouring Edge Normal, Edge Normal Calculation via Vertex Gradient Averaging

## Overview
This method computes the surface normal for a grid edge intersected by an isosurface by averaging the vertex gradients at the edge's endpoints. It serves as a critical component in Dual Contouring, where accurate normals are required to define the local plane for projecting feature points [S1, S2, unofficial]. Unlike Marching Cubes, which derives normals from cell-centered gradients, this approach leverages the vertex-based gradient information inherent to the dual grid structure to maintain feature preservation [S1, S2, unofficial].

## How it works
**Inputs:**
- Two 3D grid vertices $p_0$ and $p_1$ defining the edge.
- A scalar density function $d(x, y, z)$ that returns a value (e.g., signed distance or occupancy) for any grid coordinate.
- The isovalue (threshold) $isoLevel$.

**Outputs:**
- A 3D normal vector $\mathbf{n}_{edge}$ for the intersection point on the edge.

**Data Structures:**
- `IsoData`: A structure or tuple containing the density values at $p_0$ ($d_0$), $p_1$ ($d_1$), and the $isoLevel$.
- `Vector3`: A 3D vector type with components $x, y, z$.

**Steps:**

1.  **Sample Edge Densities:**
    Retrieve the density values at the two vertices of the edge:
    $$ d_0 = d(p_0) $$
    $$ d_1 = d(p_1) $$
    Store these along with the isovalue to form the interpolation data.

2.  **Calculate Vertex Gradient at $p_0$:**
    Compute the gradient (normal) at vertex $p_0$ using forward finite differences. Let $\mathbf{n}_0$ be the normal at $p_0$:
    $$ \mathbf{n}_0.x = d(p_0.x + 1, p_0.y, p_0.z) - d_0 $$
    $$ \mathbf{n}_0.y = d(p_0.x, p_0.y + 1, p_0.z) - d_0 $$
    $$ \mathbf{n}_0.z = d(p_0.x, p_0.y, p_0.z + 1) - d_0 $$
    *Note: This assumes a unit grid spacing. If spacing is $\Delta$, divide by $\Delta$.* [S1, S2, unofficial]

3.  **Calculate Vertex Gradient at $p_1$:**
    Compute the gradient (normal) at vertex $p_1$ similarly. Let $\mathbf{n}_1$ be the normal at $p_1$:
    $$ \mathbf{n}_1.x = d(p_1.x + 1, p_1.y, p_1.z) - d_1 $$
    $$ \mathbf{n}_1.y = d(p_1.x, p_1.y + 1, p_1.z) - d_1 $$
    $$ \mathbf{n}_1.z = d(p_1.x, p_1.y, p_1.z + 1) - d_1 $$ [S1, S2, unofficial]

4.  **Interpolate Edge Normal:**
    Interpolate between $\mathbf{n}_0$ and $\mathbf{n}_1$ based on the density values. The specific interpolation function `LerpByDensity` typically performs a linear interpolation weighted by the distance to the isosurface relative to the edge length.
    $$ \mathbf{n}_{edge} = \text{LerpByDensity}(\mathbf{n}_0, \mathbf{n}_1, d_0, d_1, isoLevel) $$
    *Implementation Note:* If `LerpByDensity` is not explicitly defined, a standard linear interpolation using the ratio of the distance from $p_0$ to the intersection point ($t$) is used:
    $$ t = \frac{isoLevel - d_0}{d_1 - d_0} $$
    $$ \mathbf{n}_{edge} = (1 - t)\mathbf{n}_0 + t\mathbf{n}_1 $$
    Finally, normalize $\mathbf{n}_{edge}$ to unit length. [S1, S2, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_edge_normal(p0, p1, density_func, iso_level):
    """
    Calculate the normal for an edge intersected by an isosurface.
    
    Args:
        p0: Tuple (x, y, z) of the first vertex.
        p1: Tuple (x, y, z) of the second vertex.
        density_func: Function that takes (x, y, z) and returns a scalar density.
        iso_level: The threshold value defining the surface.
        
    Returns:
        A normalized 3D vector representing the edge normal.
    """
    # Step 1: Sample densities at vertices
    d0 = density_func(p0[0], p0[1], p0[2])
    d1 = density_func(p1[0], p1[1], p1[2])
    
    # Check for sign change or intersection validity
    # If both are on the same side of iso_level, no intersection occurs on this edge.
    if (d0 - iso_level) * (d1 - iso_level) > 0:
        return None  # No intersection

    # Step 2: Calculate gradient at p0 using forward differences
    # d0_next_x = density at (p0.x + 1, p0.y, p0.z)
    d0_next_x = density_func(p0[0] + 1, p0[1], p0[2])
    d0_next_y = density_func(p0[0], p0[1] + 1, p0[2])
    d0_next_z = density_func(p0[0], p0[1], p0[2] + 1)
    
    n0_x = d0_next_x - d0
    n0_y = d0_next_y - d0
    n0_z = d0_next_z - d0
    
    # Step 3: Calculate gradient at p1 using forward differences
    # d1_next_x = density at (p1.x + 1, p1.y, p1.z)
    d1_next_x = density_func(p1[0] + 1, p1[1], p1[2])
    d1_next_y = density_func(p1[0], p1[1] + 1, p1[2])
    d1_next_z = density_func(p1[0], p1[1], p1[2] + 1)
    
    n1_x = d1_next_x - d1
    n1_y = d1_next_y - d1
    n1_z = d1_next_z - d1
    
    # Step 4: Interpolate normals
    # Calculate interpolation factor t based on density values
    # Avoid division by zero if d0 == d1
    if d1 == d0:
        t = 0.5
    else:
        t = (iso_level - d0) / (d1 - d0)
        
    # Linear interpolation of normals
    n_edge_x = (1 - t) * n0_x + t * n1_x
    n_edge_y = (1 - t) * n0_y + t * n1_y
    n_edge_z = (1 - t) * n0_z + t * n1_z
    
    # Normalize the resulting normal vector
    magnitude = sqrt(n_edge_x**2 + n_edge_y**2 + n_edge_z**2)
    if magnitude == 0:
        return None # Degenerate case
        
    return (n_edge_x / magnitude, n_edge_y / magnitude, n_edge_z / magnitude)
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Forward differences require accessing neighbors $(x+1, y, z)$, etc. If $p_0$ or $p_1$ lies on the boundary of the grid, these neighbors may be out of bounds. The implementation must handle boundary checks, typically by clamping coordinates or using one-sided differences [S1, S2, unofficial].
- **Degenerate Normals:** If the density function is flat (constant) in a region, the gradient will be zero, resulting in a zero vector. Normalizing a zero vector is undefined. Guard against this by checking the magnitude before normalizing [S1, S2, unofficial].
- **Incorrect Interpolation:** Simply averaging $\mathbf{n}_0$ and $\mathbf{n}_1$ without weighting by the intersection position $t$ can lead to inaccurate normals, especially if the intersection point is close to one vertex [S1, S2, unofficial].
- **Normal Direction Consistency:** Ensure that the gradient direction points consistently (e.g., outward from the surface) across the volume. Inconsistent signs in the density function will flip normals.

## Operating characteristics
- **Time Complexity:** $O(1)$ per edge, as it involves a fixed number of density queries and arithmetic operations.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Simple to implement; uses local information only; compatible with sparse grid structures if density queries are efficient.
- **Limitations:** Forward differences are only first-order accurate; sensitive to grid alignment; may produce noisy normals if the density field is noisy. The source notes that interpolating normals this way may result in incorrect normals and suggests calculating per-face normals instead, though this requires triangle generation first [S1, S2, unofficial].

## Background
This method is derived from implementations of Dual Contouring for isosurface extraction, specifically as discussed in community forums regarding feature point calculation and normal estimation [S1, S2, unofficial]. It relies on the principle that the surface normal at an edge intersection can be approximated by interpolating the vertex gradients.

## Sources
[S1] Dual Contouring - Finding the feature point, normals off, https://gamedev.stackexchange.com/questions/111387, fetched 2026-10-28 (unofficial)
[S2] Dual Contouring - Finding the feature point, normals off, https://gamedev.stackexchange.com/questions/111387/dual-contouring-finding-the-feature-point-normals-off?noredirect=1&lq=1, fetched 2026-10-29 (unofficial)
