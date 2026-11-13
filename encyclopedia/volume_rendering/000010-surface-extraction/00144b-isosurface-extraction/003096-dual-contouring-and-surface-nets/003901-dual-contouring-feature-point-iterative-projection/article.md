# Dual Contouring Feature Point Iterative Projection

Also called: Feature Point Iterative Projection, Iterative Feature Point Projection

## Overview
An iterative method to find a feature point (vertex) in dual contouring by projecting a point onto planes defined by edge midpoints and normals, minimizing the distance to these planes [S1, S2, unofficial]. It differs from standard dual contouring by using a fixed-iteration gradient descent-like approach rather than solving a full quadratic error function (QEF) system, and from MLS Projection by relying on simple plane projections rather than moving least squares fitting [S2, unofficial].

## How it works
**Inputs:**
- A voxel cell (cube) with intersecting edges.
- For each intersecting edge: the midpoint of the edge intersection with the isosurface, and the surface normal at that midpoint.
- The center of the voxel cell.

**Outputs:**
- A single 3D point (the feature point) inside the voxel cell.

**Data Structures:**
- `Plane`: Defined by a normal vector $\mathbf{n}$ and a scalar distance $d$ from the origin, such that the plane equation is $\mathbf{n} \cdot \mathbf{x} + d = 0$.
  - $\mathbf{n}$: Normalized 3D vector.
  - $d$: Scalar, computed as $-\mathbf{n} \cdot \mathbf{p}$ where $\mathbf{p}$ is a point on the plane (e.g., the edge midpoint).

**Parameters:**
- `iterations`: Number of projection steps. Typical value: 5 [S1, S2, unofficial].
- `step_factor`: Multiplier for the averaged distance vector. Typical value: 0.7 [S1, S2, unofficial]. Note: Convergence occurs without scaling (factor 1.0), but 0.7 stabilizes convergence speed [S1, S2, unofficial].

**Steps:**
1. Initialize the current point $\mathbf{c}$ to the center of the voxel cell.
2. For each intersecting edge $i$, create a plane $P_i$ using the edge's midpoint $\mathbf{m}_i$ and normal $\mathbf{n}_i$:
   - $\mathbf{n}_i = \text{normalize}(\mathbf{n}_i)$
   - $d_i = -\mathbf{n}_i \cdot \mathbf{m}_i$
3. Iterate $k$ from 0 to `iterations` - 1:
   a. For each plane $P_i$, calculate the signed distance from $\mathbf{c}$ to the plane:
      $$ \text{dist}_i = \mathbf{n}_i \cdot \mathbf{c} + d_i $$
   b. Calculate the shortest distance vector (displacement) for each plane:
      $$ \mathbf{v}_i = \text{dist}_i \cdot \mathbf{n}_i $$
   c. Average the displacement vectors:
      $$ \mathbf{avg} = \frac{1}{N} \sum_{i=1}^{N} \mathbf{v}_i $$
      where $N$ is the number of intersecting edges (typically 3 for a corner cut, but can vary).
   d. Update the current point:
      $$ \mathbf{c} = \mathbf{c} - \mathbf{avg} \cdot \text{step\_factor} $$
4. Return $\mathbf{c}$ as the feature point.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_feature_point(intersecting_edges, voxel_center):
    """
    Compute the feature point for a voxel cell using iterative projection.
    
    Args:
        intersecting_edges: List of tuples (midpoint, normal) for each edge intersecting the isosurface.
        voxel_center: 3D vector representing the center of the voxel.
        
    Returns:
        3D vector representing the feature point.
    """
    # Step 1: Initialize point to voxel center
    c = voxel_center
    
    # Precompute planes
    planes = []
    for midpoint, normal in intersecting_edges:
        # Normalize normal
        n = normalize(normal)
        # Compute plane distance d such that n.x + d = 0 for point x on plane
        d = -dot(n, midpoint)
        planes.append((n, d))
    
    num_iterations = 5
    step_factor = 0.7
    
    # Step 3: Iterate
    for _ in range(num_iterations):
        avg_displacement = Vector3(0, 0, 0)
        
        for n, d in planes:
            # Calculate signed distance from c to plane
            dist = dot(n, c) + d
            # Calculate displacement vector
            v = n * dist
            avg_displacement += v
        
        # Average displacement vectors
        N = len(planes)
        if N > 0:
            avg_displacement = avg_displacement / N
        
        # Update point
        c = c - avg_displacement * step_factor
        
    return c
```

**Alternative Formulation:**
Instead of subtracting the averaged distance vector, one can project the point onto each plane individually and then average the projected points. This is mathematically equivalent [S2, unofficial]:
1. For each plane $P_i$, calculate the projected point:
   $$ \mathbf{v}_i = \mathbf{c} - \text{dist}_i \cdot \mathbf{n}_i $$
2. Average the projected points:
   $$ \mathbf{c} = \frac{1}{N} \sum_{i=1}^{N} \mathbf{v}_i $$

**Edge Cases and Pitfalls:**
- **Non-intersecting planes:** The algorithm may fail if the planes do not intersect within the voxel, leading to divergence or unstable results [S1, S2, unofficial].
- **Direction of update:** Subtracting the averaged vector moves the point towards the planes (convergence). Adding it moves the point away from the intersection (divergence) [S1, S2, unofficial].
- **Precision:** Ensure normals are normalized before computing plane distances to avoid scaling errors [S1, S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot I)$, where $N$ is the number of intersecting edges per voxel and $I$ is the number of iterations. Typically $N \le 12$ (max edges in a cube) and $I=5$, making it very fast per voxel.
- **Memory Complexity:** $O(N)$ per voxel to store plane parameters.
- **Strengths:** Simple to implement, no need to solve linear systems (unlike QEF), fast convergence with fixed iterations.
- **Limitations:** May not converge to the exact least-squares solution if planes are inconsistent or non-intersecting. Quality depends on normal accuracy.
- **When to use:** When performance is critical and normals are reliable. Suitable for real-time applications or when QEF solving is too expensive.
- **When not to use:** When high geometric accuracy is required in the presence of noisy normals or non-intersecting planes.

## Background
This method originates from community discussions on implementing dual contouring for voxel-based games, specifically addressing issues with feature point placement when normals are calculated via finite differences [S1, S2, unofficial]. It was proposed as a correction to a common implementation error where the update direction was inverted, causing divergence. The approach is a simplified iterative solver for the feature point location problem in dual contouring.

## Sources
[S1] Dual Contouring - Finding the feature point, normals off, https://gamedev.stackexchange.com/questions/111387, fetched 2026-10-28 (unofficial)
[S2] Dual Contouring - Finding the feature point, normals off, https://gamedev.stackexchange.com/questions/111387/dual-contouring-finding-the-feature-point-normals-off?noredirect=1&lq=1, fetched 2026-10-29 (unofficial)
