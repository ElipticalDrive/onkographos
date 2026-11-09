# Split-and-Prune for Sub-Triangle Search

Also called: Cone Angles Triangle Subdivision Heuristic

## Overview
A recursive strategy to find refraction points on triangles with smooth normals by progressively splitting the triangle into smaller sub-triangles and pruning those that cannot contain a solution [S1]. It enables efficient 2D search for refractive paths where interpolated normals create complex, non-planar solution spaces [S1]. It is distinguished from interval-based refinement by prioritizing speed over guaranteed convergence, typically running 2 to 5 times faster while producing visually equivalent results [S1].

## How it works
**Inputs:**
*   Triangle vertices $V_0, V_1, V_2$ with positions and shading normals [S1].
*   Light direction $\hat{L}$ and view direction $\hat{V}$ at the incident point $P$ [S1].
*   Refraction index $\eta$ [S1].

**Outputs:**
*   A set of sub-triangles containing potential solutions, ready for 2D Newton-Raphson iteration [S1].

**Data Structures:**
*   **Bounding Cone:** Defined by an axis vector $\hat{A}$ and a semi-angle $\theta$ [S1].
*   **Spherical Triangle Bounding Box:** A 3D axis-aligned bounding box (AABB) enclosing a spherical triangle on the unit sphere [S1].

**Steps:**

1.  **Compute Bounding Cone for Shading Normals ($\hat{N}_s$):**
    *   Construct the spherical triangle defined by the vertex shading normals $\hat{N}_0, \hat{N}_1, \hat{N}_2$ [S1].
    *   Compute the bounding cone for this spherical triangle [S1]. All bounding cones have their apex at the origin [S1].

2.  **Compute Bounding Cone for Half-Vector ($\hat{H}$):**
    *   Compute the spherical triangles defined by the directions $(\hat{L}-V_i)/\|\hat{L}-V_i\|$ and $(\hat{V}-V_i)/\|\hat{V}-V_i\|$ for $i=0,1,2$ [S1].
    *   Compute 3D AABBs for both spherical triangles [S1].
    *   Scale the view direction AABB by $\eta$ [S1].
    *   Compute the Minkowski sum of the light AABB and the scaled view AABB by adding corresponding min/max bounds [S1]. This yields an AABB for the unnormalized half-vector [S1].
    *   Compute the bounding cone for this resulting AABB [S1].

3.  **Pruning Test:**
    *   Check if the bounding cone for $-\hat{H}$ and the bounding cone for $\hat{N}_s$ overlap [S1].
    *   If they do not overlap, prune the triangle (it cannot contain a solution) [S1]. Two cones overlap if the angle between their axes is less than the sum of their bounding semiangles [S1].

4.  **Subdivision or Termination:**
    *   If the triangle is not pruned, check the stopping criterion [S1].
    *   **Heuristic Stopping Criterion:** If the sum of the $\hat{H}$ cone angle and the $\hat{N}_s$ cone angle is less than 30 degrees, stop subdividing and proceed to 2D Newton-Raphson iteration [S1].
    *   If the criterion is not met, split the triangle into 4 smaller sub-triangles by splitting each edge in half [S1].
    *   Recursively apply steps 1-4 to each sub-triangle [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class AABB:
    def __init__(self, min_x, max_x, min_y, max_y, min_z, max_z):
        self.min_x, self.max_x = min_x, max_x
        self.min_y, self.max_y = min_y, max_y
        self.min_z, self.max_z = min_z, max_z

class Triangle:
    def __init__(self, v0, v1, v2, n0, n1, n2):
        self.v0, self.v1, self.v2 = v0, v1, v2
        self.n0, self.n1, self.n2 = n0, n1, n2

def midpoint(p1, p2):
    return ((p1[0]+p2[0])/2, (p1[1]+p2[1])/2, (p1[2]+p2[2])/2)

def normalize(v):
    mag = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return (v[0]/mag, v[1]/mag, v[2]/mag) if mag > 0 else (0,0,0)

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def angle_between_vectors(v1, v2):
    return math.acos(max(-1.0, min(1.0, dot(normalize(v1), normalize(v2)))))

def compute_spherical_triangle_bounding_cone(n0, n1, n2):
    # [G] Standard computation: axis is normalized sum, semi-angle is max angle to vertices
    axis = normalize((n0[0]+n1[0]+n2[0], n0[1]+n1[1]+n2[1], n0[2]+n1[2]+n2[2]))
    angle = max(angle_between_vectors(n0, axis), angle_between_vectors(n1, axis), angle_between_vectors(n2, axis))
    return (axis, angle)

def compute_aabb(sph_tri):
    xs = [p[0] for p in sph_tri]
    ys = [p[1] for p in sph_tri]
    zs = [p[2] for p in sph_tri]
    return AABB(min(xs), max(xs), min(ys), max(ys), min(zs), max(zs))

def scale_aabb(box, s):
    return AABB(box.min_x*s, box.max_x*s, box.min_y*s, box.max_y*s, box.min_z*s, box.max_z*s)

def minkowski_sum(b1, b2):
    return AABB(b1.min_x+b2.min_x, b1.max_x+b2.max_x,
                b1.min_y+b2.min_y, b1.max_y+b2.max_y,
                b1.min_z+b2.min_z, b1.max_z+b2.max_z)

def compute_box_bounding_cone(box):
    # [G] Axis is direction to box center, semi-angle is max angle to any corner
    cx, cy, cz = (box.min_x+box.max_x)/2, (box.min_y+box.max_y)/2, (box.min_z+box.max_z)/2
    axis = normalize((cx, cy, cz))
    corners = [
        (box.min_x, box.min_y, box.min_z), (box.max_x, box.min_y, box.min_z),
        (box.min_x, box.max_y, box.min_z), (box.max_x, box.max_y, box.min_z),
        (box.min_x, box.min_y, box.max_z), (box.max_x, box.min_y, box.max_z),
        (box.min_x, box.max_y, box.max_z), (box.max_x, box.max_y, box.max_z)
    ]
    angle = max(angle_between_vectors(c, axis) for c in corners)
    return (axis, angle)

def cones_overlap(cone1, cone2):
    # Two cones overlap if the angle between their axes is less than the sum of their bounding semiangles
    return angle_between_vectors(cone1[0], cone2[0]) < (cone1[1] + cone2[1])

def split_and_prune_search(triangle, L_dir, V_dir, eta):
    # Step 1: Compute bounding cone for shading normals (Ns)
    n_s_cone = compute_spherical_triangle_bounding_cone(triangle.n0, triangle.n1, triangle.n2)

    # Step 2: Compute bounding cone for half-vector (H)
    # Spherical triangle for L
    l_sph_tri = [normalize((L_dir[0]-v[0], L_dir[1]-v[1], L_dir[2]-v[2])) for v in (triangle.v0, triangle.v1, triangle.v2)]
    # Spherical triangle for V
    v_sph_tri = [normalize((V_dir[0]-v[0], V_dir[1]-v[1], V_dir[2]-v[2])) for v in (triangle.v0, triangle.v1, triangle.v2)]

    l_box = compute_aabb(l_sph_tri)
    v_box = compute_aabb(v_sph_tri)
    v_box_scaled = scale_aabb(v_box, eta)
    h_box = minkowski_sum(l_box, v_box_scaled)
    h_cone = compute_box_bounding_cone(h_box)

    # Step 3: Pruning Test
    # Check if bounding cone for -H and Ns overlap
    neg_h_cone = (-h_cone[0][0], -h_cone[0][1], -h_cone[0][2]), h_cone[1]
    if not cones_overlap(neg_h_cone, n_s_cone):
        return [] # Pruned

    # Step 4: Subdivision or Termination
    cone_angle_sum_deg = math.degrees(h_cone[1] + n_s_cone[1])
    if cone_angle_sum_deg < 30.0:
        return [triangle] # Stop subdividing, proceed to Newton-Raphson
    else:
        mid01 = midpoint(triangle.v0, triangle.v1)
        mid12 = midpoint(triangle.v1, triangle.v2)
        mid20 = midpoint(triangle.v2, triangle.v0)
        # Interpolate normals for sub-triangles [G]
        n01 = normalize((triangle.n0[0]+triangle.n1[0], triangle.n0[1]+triangle.n1[1], triangle.n0[2]+triangle.n1[2]))
        n12 = normalize((triangle.n1[0]+triangle.n2[0], triangle.n1[1]+triangle.n2[1], triangle.n1[2]+triangle.n2[2]))
        n20 = normalize((triangle.n2[0]+triangle.n0[0], triangle.n2[1]+triangle.n0[1], triangle.n2[2]+triangle.n0[2]))

        sub_triangles = [
            Triangle(triangle.v0, mid01, mid20, triangle.n0, n01, n20),
            Triangle(mid01, triangle.v1, mid12, n01, triangle.n1, n12),
            Triangle(mid20, mid12, triangle.v2, n20, n12, triangle.n2),
            Triangle(mid01, mid12, mid20, n01, n12, n20)
        ]
        results = []
        for sub_t in sub_triangles:
            results.extend(split_and_prune_search(sub_t, L_dir, V_dir, eta))
        return results
```

**Edge Cases and Pitfalls:**
*   The heuristic is not guaranteed to find all solutions [S1].
*   It may miss a few paths near caustic cusps on rare occasions, affecting at most a few pixels [S1].
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
