# Medial Axis Detection via Foot Point Method
Also called: Foot Point Method for Medial Axis
## Overview
This method detects the medial axis (MA) of a general implicit surface by analyzing characteristic lines (normals) and detecting changes in foot points, providing subgrid accuracy without thinning or post-processing [S1]. It differs from its parent [Distance Field Foot Point Generation](6946-distance-field-foot-point-generation.md) by not just generating surface points, but by identifying the locus of centers of maximal inscribed spheres (the medial axis) to compute the Local Feature Size (LFS) field [S1].
## How it works
**Inputs:**
- Implicit surface function $F(\mathbf{x}) = 0$.
- Grid of query points $\mathcal{Q}$.

**Outputs:**
- Set of medial axis points $\mathcal{M}$.
- Local Feature Size (LFS) field $\lambda(\mathbf{x})$ at grid nodes bounding the isosurface.

**Data Structures:**
- **Medial Axis Set $\mathcal{M}$**: A list of 3D points $\mathbf{m} \in \mathbb{R}^3$.
- **LFS Field $\lambda$**: A scalar field defined on grid nodes $\mathbf{x}_g$ that bound the isosurface.

**Parameters:**
- Threshold angle $\alpha$: Defined by $\cos(\alpha) = 0.9$ [S1].

**Steps:**

1. **Grid Processing**: Iterate over each grid point $\mathbf{q} \in \mathcal{Q}$.
2. **Find Foot Point $\mathbf{p}$**: Compute the nearest point $\mathbf{p}$ on the surface $F(\mathbf{x})=0$ to $\mathbf{q}$ using gradient descent [S1, S17]. The segment $\overline{\mathbf{p}\mathbf{q}}$ is perpendicular to the surface and forms part of a characteristic line [S1].
3. **Find Far Point $\mathbf{q}^*$**: Extend the characteristic line from $\mathbf{p}$ through $\mathbf{q}$ (away from the surface) to find the point $\mathbf{q}^*$ that intersects the far face of the current voxel [S1].
4. **Find Far Foot Point $\mathbf{p}^*$**: Compute the foot point $\mathbf{p}^*$ on the surface corresponding to $\mathbf{q}^*$ using gradient descent [S1].
5. **Check MA Crossing**: Calculate the angle $\theta$ between the vector $\overline{\mathbf{p}\mathbf{q}}$ and $\overline{\mathbf{p}^*\mathbf{q}^*}$. Specifically, check the angle between the segments $\overline{\mathbf{p}\mathbf{q}}$ and $\overline{\mathbf{p}^*\mathbf{q}^*}$ [S1].
   - If the angle between the line segments $\overline{\mathbf{p}\mathbf{q}}$ and $\overline{\mathbf{p}^*\mathbf{q}^*}$ is greater than $\alpha$ (i.e., $\cos(\theta) < 0.9$), the segment $\overline{\mathbf{p}\mathbf{p}^*}$ crosses the medial axis [S1].
6. **Estimate MA Position**: If a crossing is detected, estimate the position of the medial axis point $\mathbf{m}$ along the segment $\overline{\mathbf{p}\mathbf{p}^*}$ using a first-order (tangent plane) approximation to the surface at $\mathbf{p}^*$ [S1].
7. **Construct LFS Field**: After processing all grid points, construct the LFS field $\lambda(\mathbf{x})$. For each grid node $\mathbf{x}_g$ bounding the isosurface, set $\lambda(\mathbf{x}_g)$ to the distance to the closest medial axis point $\mathbf{m} \in \mathcal{M}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def find_foot_point_gradient_descent(q, F, grad_F, max_iter=100, tol=1e-6):
    """
    Finds the nearest point p on the surface F(x)=0 to query point q.
    Uses gradient descent.
    """
    p = q.copy()
    for _ in range(max_iter):
        f_val = F(p)
        if abs(f_val) < tol:
            break
        g = grad_F(p)
        norm_g = np.linalg.norm(g)
        if norm_g < 1e-10:
            break
        # Step along normal direction to approach surface
        step = f_val / norm_g
        p = p - step * g
    return p

def detect_medial_axis(grid_points, F, grad_F, voxel_size):
    """
    Detects medial axis points and constructs LFS field.
    """
    medial_axis_points = []
    cos_alpha = 0.9
    threshold = cos_alpha
    
    # Note: In practice, grid_points should be those bounding the isosurface
    # or a dense grid covering the volume of interest.
    
    for q in grid_points:
        # Step 2: Find foot point p for q
        p = find_foot_point_gradient_descent(q, F, grad_F)
        
        # Step 3: Find q* along characteristic intersecting far face of voxel
        # Characteristic direction is from p to q (or q to p, but we go away from p)
        # Vector pq
        vec_pq = q - p
        # Normalize
        norm_pq = np.linalg.norm(vec_pq)
        if norm_pq < 1e-10:
            continue
        dir_char = vec_pq / norm_pq
        
        # Find intersection with far face of voxel containing q
        # Assuming axis-aligned voxel grid with size voxel_size
        # This requires determining which face is the "far" face relative to p
        # For simplicity, we extend by one voxel length in the direction of the characteristic
        # A robust implementation would compute exact ray-box intersection
        q_star = q + dir_char * voxel_size 
        
        # Step 4: Find foot point p* for q*
        p_star = find_foot_point_gradient_descent(q_star, F, grad_F)
        
        # Step 5: Check angle between pq and p*q*
        # Vector p*q*
        vec_p_star_q_star = q_star - p_star
        norm_p_star_q_star = np.linalg.norm(vec_p_star_q_star)
        
        if norm_p_star_q_star < 1e-10:
            continue
            
        dir_p_star_q_star = vec_p_star_q_star / norm_p_star_q_star
        
        # Cosine of angle between the two characteristic directions
        # Note: The text says "angle between line segments pq and p*q*"
        # This usually implies the angle between the vectors originating from the surface
        # or the vectors along the characteristics.
        cos_theta = np.dot(dir_char, dir_p_star_q_star)
        
        if cos_theta < threshold:
            # Step 6: Estimate MA position along p-p*
            # Using first-order tangent plane approximation at p*
            # The MA point m is the intersection of the normal at p and the normal at p*?
            # Or intersection of the characteristic lines?
            # The text says "position of the MA along pp* can be found by using a 
            # first-order (tangent plane) approximation to the surface at the point p*"
            # This implies projecting p onto the tangent plane at p*?
            # Or finding the intersection of the line pp* with the tangent plane at p*?
            # Let's assume the latter: Intersection of line segment pp* with tangent plane at p*
            
            # Tangent plane at p*: normal is grad_F(p*)
            n_star = grad_F(p_star)
            norm_n_star = np.linalg.norm(n_star)
            if norm_n_star < 1e-10:
                continue
            n_star = n_star / norm_n_star
            
            # Line segment pp*: P(t) = p + t * (p_star - p), t in [0, 1]
            # Plane: (x - p_star) . n_star = 0
            # (p + t*(p_star - p) - p_star) . n_star = 0
            # (p - p_star + t*(p_star - p)) . n_star = 0
            # (p - p_star) . n_star + t * (p_star - p) . n_star = 0
            # t = - (p - p_star) . n_star / (p_star - p) . n_star
            # t = (p_star - p) . n_star / (p_star - p) . n_star = 1?
            # This would mean the intersection is at p*. That doesn't seem right for MA detection.
            
            # Re-reading: "first-order (tangent plane) approximation to the surface at point p*"
            # The medial axis is the locus of centers of spheres tangent to the surface.
            # The center of such a sphere lies on the normal.
            # The MA point m is the intersection of the normal at p and the normal at p*?
            # If we approximate the surface locally as planes, the intersection of the two normals
            # gives the center of the circle tangent to both planes.
            
            # Normal at p
            n_p = grad_F(p)
            norm_n_p = np.linalg.norm(n_p)
            if norm_n_p < 1e-10:
                continue
            n_p = n_p / norm_n_p
            
            # Intersection of line p + t*n_p and line p_star + s*n_star
            # This is a standard line-line intersection in 3D.
            # If lines are skew, we find the closest points.
            # However, for small distances, they might intersect.
            # Let's find the point m on the segment pp* that is closest to the intersection of normals?
            # The text says "position of the MA along pp*".
            # Let's assume the MA point is the intersection of the normal at p and the tangent plane at p*?
            # No, the MA is equidistant.
            
            # Let's stick to the most literal interpretation of "first-order approximation at p*"
            # to find the position along pp*.
            # Perhaps it means: The MA point m is the projection of p onto the tangent plane at p*?
            # Or the projection of p* onto the tangent plane at p?
            
            # Given the ambiguity, a common approach in such foot-point methods is:
            # m = p + t * (p_star - p)
            # where t is determined by the geometry of the normals.
            # Let's use the intersection of the normal at p with the tangent plane at p*.
            # Line: L(t) = p + t * n_p
            # Plane: (x - p_star) . n_star = 0
            # (p + t*n_p - p_star) . n_star = 0
            # t = (p_star - p) . n_star / (n_p . n_star)
            
            denom = np.dot(n_p, n_star)
            if abs(denom) < 1e-10:
                continue
            t = np.dot(p_star - p, n_star) / denom
            
            # Clamp t to [0, 1] to stay on segment pp*
            t = max(0.0, min(1.0, t))
            m = p + t * (p_star - p)
            
            medial_axis_points.append(m)
            
    # Step 7: Construct LFS field
    # For each grid node bounding the isosurface, find distance to closest m in medial_axis_points
    # This is a separate pass.
    return medial_axis_points

def compute_lfs_field(grid_nodes, medial_axis_points):
    """
    Computes the LFS field lambda(x) for grid nodes bounding the isosurface.
    """
    lfs_field = {}
    for x in grid_nodes:
        min_dist = float('inf')
        for m in medial_axis_points:
            dist = np.linalg.norm(x - m)
            if dist < min_dist:
                min_dist = dist
        lfs_field[x] = min_dist
    return lfs_field
```

**Edge Cases and Pitfalls:**
- **Gradient Descent Failure**: If the gradient is zero or the surface is not smooth, `find_foot_point_gradient_descent` may fail or converge to a local minimum. [G] Ensure the implicit function is differentiable and use a robust solver with fallbacks.
- **Voxel Intersection**: Determining the "far face" of the voxel for $\mathbf{q}^*$ requires precise ray-box intersection logic. [G] Use a standard ray-box intersection algorithm (e.g., slab method) to find the exact exit point of the ray from the voxel containing $\mathbf{q}$.
- **Degenerate Normals**: If the normal at $\mathbf{p}$ or $\mathbf{p}^*$ is near-zero, the tangent plane approximation is invalid. Check for small norms before division.
- **Singular Points**: The method detects the MA proper, not the singular points where the MA terminates (e.g., vertices of a polyhedron). [S1]

## Subtopics

### [Sizing Field Construction](10996-sizing-field-construction.md)
The sizing field construction generates a scalar field $h(\mathbf{x})$ that controls particle distribution density to ensure $\varepsilon$-sampling and smooth triangle gradation for isosurface meshing. It differs from raw geometric sampling by enforcing a $\delta$-Lipschitz continuity constraint on the field, preventing abrupt changes in triangle size that would degrade mesh quality. This method builds on the medial axis detection provided by [Medial Axis Detection via Foot Point Method](10987-medial-axis-detection-via-foot-point-method.md) by utilizing the resulting local feature size to initialize the field.

## Operating characteristics
- **Time Complexity**: Dominated by the gradient descent for foot point computation for each grid point and each $\mathbf{q}^*$. If $N$ is the number of grid points, and $K$ is the average number of iterations for gradient descent, the complexity is $O(N \cdot K)$. The LFS field construction is $O(N_{nodes} \cdot |\mathcal{M}|)$, which can be optimized with spatial indexing (e.g., k-d tree) to $O(N_{nodes} \log |\mathcal{M}|)$.
- **Memory Complexity**: $O(N)$ to store grid points and intermediate results, plus $O(|\mathcal{M}|)$ for the medial axis points.
- **Strengths**: Subgrid accuracy, no thinning or post-processing required, robust to free parameters [S1].
- **Limitations**: Relies on accurate gradient computation and foot point finding. May miss singular points of the MA [S1].
- **When to Use**: When high-quality meshing is required and the LFS field is needed for adaptive sampling.
- **When Not to Use**: If the surface is not smooth or gradients are unreliable.

## Background
The method was developed as part of work on topology, accuracy, and quality of isosurface meshes using dynamic particles [S1]. It builds on the foot point method described in [17] (not cited in the provided sources, but referenced in [S1]). The specific paper is "Topology, Accuracy, and Quality of Isosurface Meshes Using Dynamic Particles" [S1].

## Sources
[S1] Topology, Accuracy, and Quality of Isosurface Meshes Using Dynamic Particles, https://doi.org/10.1109/tvcg.2007.70604, fetched 2026-10-30
