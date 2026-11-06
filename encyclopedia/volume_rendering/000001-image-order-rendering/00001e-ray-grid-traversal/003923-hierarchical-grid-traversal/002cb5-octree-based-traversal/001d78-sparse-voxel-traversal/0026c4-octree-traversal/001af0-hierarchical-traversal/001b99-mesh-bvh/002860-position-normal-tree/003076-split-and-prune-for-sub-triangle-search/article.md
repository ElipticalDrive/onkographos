# Split-and-Prune for Sub-Triangle Search

Also called: Cone Angles Triangle Subdivision Heuristic

## Overview
A recursive strategy to find refraction points on triangles with smooth normals by progressively splitting the triangle into smaller sub-triangles and pruning those that cannot contain a solution [S1]. It enables efficient 2D search for refractive paths where interpolated normals create complex, non-planar solution spaces [S1]. It is distinguished from interval-based refinement by prioritizing speed over guaranteed convergence, typically running 2 to 5 times faster while producing visually equivalent results [S1].

## How it works
**Inputs:**
*   Triangle vertices $V_0, V_1, V_2$ with positions and shading normals.
*   Light direction $\hat{L}$ and view direction $\hat{V}$ at the incident point $P$.
*   Refraction index $\eta$.

**Outputs:**
*   A set of sub-triangles containing potential solutions, ready for 2D Newton-Raphson iteration [S1].

**Data Structures:**
*   **Bounding Cone:** Defined by an axis vector $\hat{A}$ and a semi-angle $\theta$. Two cones overlap if the angle between their axes is less than the sum of their semi-angles [S1].
*   **Spherical Triangle Bounding Box:** A 3D axis-aligned bounding box (AABB) enclosing a spherical triangle on the unit sphere.

**Steps:**

1.  **Compute Bounding Cone for Shading Normals ($\hat{N}_s$):**
    *   Construct the spherical triangle defined by the vertex shading normals $\hat{N}_0, \hat{N}_1, \hat{N}_2$ [S1].
    *   Compute the bounding cone for this spherical triangle. All bounding cones have their apex at the origin [S1].

2.  **Compute Bounding Cone for Half-Vector ($\hat{H}$):**
    *   Compute the spherical triangle for the incoming light direction: vertices are $(L-V_i)/\|L-V_i\|$ for $i=0,1,2$ [S1].
    *   Compute the spherical triangle for the view direction: vertices are $(V-V_i)/\|V-V_i\|$ for $i=0,1,2$ [S1].
    *   Compute 3D AABBs for both spherical triangles [S1].
    *   Scale the view direction AABB by $\eta$ [S1].
    *   Compute the Minkowski sum of the light AABB and the scaled view AABB by adding corresponding min/max bounds [S1]. This yields an AABB for the unnormalized half-vector [S1].
    *   Compute the bounding cone for this resulting AABB [S1].

3.  **Pruning Test:**
    *   Check if the bounding cone for $-\hat{H}$ and the bounding cone for $\hat{N}_s$ overlap [S1].
    *   If they do not overlap, prune the triangle (it cannot contain a solution) [S1].

4.  **Subdivision or Termination:**
    *   If the triangle is not pruned, check the stopping criterion [S1].
    *   **Heuristic Stopping Criterion:** If the sum of the $\hat{H}$ cone angle and the $\hat{N}_s$ cone angle is less than 30 degrees, stop subdividing and proceed to 2D Newton-Raphson iteration [S1].
    *   If the criterion is not met, split the triangle into 4 smaller sub-triangles by splitting each edge in half [S1].
    *   Recursively apply steps 1-4 to each sub-triangle [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def split_and_prune_search(triangle, L_dir, V_dir, eta):
    """
    Recursively split and prune a triangle to find refraction solutions.
    
    Args:
        triangle: Object with vertices (pos, normal) for V0, V1, V2.
        L_dir: Light direction vector.
        V_dir: View direction vector.
        eta: Refraction index.
        
    Returns:
        List of sub-triangles that passed the pruning test and met the stopping criterion.
    """
    # Step 1: Compute bounding cone for shading normals
    n_s_cone = compute_spherical_triangle_bounding_cone(
        triangle.n0, triangle.n1, triangle.n2
    )
    
    # Step 2: Compute bounding cone for half-vector H
    # Compute spherical triangles for L and V
    l_sph_tri = compute_spherical_triangle(triangle.v0, triangle.v1, triangle.v2, L_dir)
    v_sph_tri = compute_spherical_triangle(triangle.v0, triangle.v1, triangle.v2, V_dir)
    
    # Compute AABBs for spherical triangles
    l_box = compute_aabb(l_sph_tri)
    v_box = compute_aabb(v_sph_tri)
    
    # Scale V box by eta
    v_box_scaled = scale_aabb(v_box, eta)
    
    # Minkowski sum
    h_box = minkowski_sum(l_box, v_box_scaled)
    
    # Compute bounding cone for H box
    h_cone = compute_box_bounding_cone(h_box)
    
    # Step 3: Pruning Test
    # Check if cones overlap
    if not cones_overlap(h_cone, n_s_cone):
        return [] # Pruned
    
    # Step 4: Subdivision or Termination
    # Heuristic stopping criterion
    cone_angle_sum = h_cone.angle + n_s_cone.angle
    if cone_angle_sum < 30.0: # Degrees
        # Stop subdividing, return this triangle for Newton-Raphson
        return [triangle]
    else:
        # Split triangle into 4 sub-triangles
        mid01 = midpoint(triangle.v0, triangle.v1)
        mid12 = midpoint(triangle.v1, triangle.v2)
        mid20 = midpoint(triangle.v2, triangle.v0)
        
        sub_triangles = [
            Triangle(triangle.v0, mid01, mid20, triangle.n0, lerp(triangle.n0, triangle.n1), lerp(triangle.n0, triangle.n2)),
            Triangle(mid01, triangle.v1, mid12, lerp(triangle.n0, triangle.n1), triangle.n1, lerp(triangle.n1, triangle.n2)),
            Triangle(mid20, mid12, triangle.v2, lerp(triangle.n0, triangle.n2), lerp(triangle.n1, triangle.n2), triangle.n2),
            Triangle(mid01, mid12, mid20, lerp(triangle.n0, triangle.n1), lerp(triangle.n1, triangle.n2), lerp(triangle.n0, triangle.n2))
        ]
        
        results = []
        for sub_t in sub_triangles:
            results.extend(split_and_prune_search(sub_t, L_dir, V_dir, eta))
        return results

def compute_spherical_triangle_bounding_cone(n0, n1, n2):
    """
    Computes the bounding cone for a spherical triangle defined by three unit vectors.
    """
    # Implementation detail: Compute the smallest cone containing the spherical triangle.
    # This involves finding the axis and semi-angle.
    # [G] Standard geometric computation for spherical triangle bounding cone.
    pass

def compute_spherical_triangle(v0, v1, v2, dir_vec):
    """
    Computes the spherical triangle vertices from world positions and a direction.
    """
    p0 = normalize(dir_vec - v0)
    p1 = normalize(dir_vec - v1)
    p2 = normalize(dir_vec - v2)
    return [p0, p1, p2]

def compute_aabb(spherical_triangle):
    """
    Computes the 3D AABB enclosing the spherical triangle.
    """
    min_x = min(p.x for p in spherical_triangle)
    max_x = max(p.x for p in spherical_triangle)
    min_y = min(p.y for p in spherical_triangle)
    max_y = max(p.y for p in spherical_triangle)
    min_z = min(p.z for p in spherical_triangle)
    max_z = max(p.z for p in spherical_triangle)
    return AABB(min_x, max_x, min_y, max_y, min_z, max_z)

def scale_aabb(box, scale):
    return AABB(box.min_x * scale, box.max_x * scale,
                box.min_y * scale, box.max_y * scale,
                box.min_z * scale, box.max_z * scale)

def minkowski_sum(box1, box2):
    return AABB(box1.min_x + box2.min_x, box1.max_x + box2.max_x,
                box1.min_y + box2.min_y, box1.max_y + box2.max_y,
                box1.min_z + box2.min_z, box1.max_z + box2.max_z)

def compute_box_bounding_cone(box):
    """
    Computes the bounding cone for a 3D AABB.
    """
    # [G] Compute the cone that tightly encloses the box.
    # This typically involves finding the axis aligned with the box diagonal
    # and the semi-angle determined by the furthest corner.
    pass

def cones_overlap(cone1, cone2):
    """
    Checks if two cones overlap.
    Two cones overlap if the angle between their axes is less than the sum of their bounding semi-angles.
    """
    angle_between_axes = angle_between_vectors(cone1.axis, cone2.axis)
    return angle_between_axes < (cone1.angle + cone2.angle)
```

**Edge Cases and Pitfalls:**
*   The heuristic is not guaranteed to find all solutions [S1].
*   It may miss a few paths near caustic cusps on rare occasions [S1].
*   Normal-mapped triangles are not supported in the original description, though bounding cones could be precomputed [S1].

## Subtopics

### Cone Angles Triangle Subdivision Heuristic
A heuristic for triangle subdivision used to find refractive paths, preferred over interval-based refinement for speed in the reported results [S1]. It is typically 2 to 5 times faster than interval-based refinement test, producing visually equivalent images [S1]. It may miss a few paths near caustic cusps on rare occasions, affecting at most a few pixels [S1].

## Operating characteristics
*   **Time Complexity:** Faster than interval-based refinement, typically 2 to 5 times faster depending on the scene [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Speed; produces visually good results [S1].
*   **Limitations:** Not guaranteed to find all solutions; may miss paths near caustic cusps [S1].
*   **When to use:** When visual equivalence is acceptable and speed is prioritized over guaranteed convergence [S1].
*   **When not to use:** When guaranteed convergence is required, in which case the interval-based refinement test should be used [S1].

## Background
Originated in the paper "Single Scattering in Refractive Media with Triangle Mesh Boundaries" [S1]. The method addresses the difficulty of finding refraction points on triangles with interpolated normals, where the problem is 2D rather than 1D due to the potential for multiple solutions not lying within a single incidence plane [S1].

## Sources
[S1] Single Scattering in Refractive Media with Triangle Mesh Boundaries, https://doi.org/10.1145/1531326.1531398, fetched 2026-10-26
