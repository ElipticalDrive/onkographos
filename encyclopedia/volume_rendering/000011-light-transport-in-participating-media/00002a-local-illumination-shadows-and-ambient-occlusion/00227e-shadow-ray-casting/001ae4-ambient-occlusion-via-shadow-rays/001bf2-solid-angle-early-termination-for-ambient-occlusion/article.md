# Solid Angle Early Termination for Ambient Occlusion

Also called: Solid Angle Culling for AO, Solid Angle Thresholding for AO

## Overview
Solid Angle Early Termination for Ambient Occlusion accelerates shadow ray traversal in sparse voxel structures by stopping traversal when a node subtends a solid angle below a specified threshold, effectively using a lower level-of-detail representation for distant points [S1]. It differs from [Evans Ambient Occlusion](1083-evans-ambient-occlusion.md) and [Zhukov Ambient Occlusion](2245-zhukov-ambient-occlusion.md) by relying on geometric projection thresholds rather than hemisphere sampling or precomputed angular distributions, and from [Incremental Local Ambient Occlusion](1089-incremental-local-ambient-occlusion.md) by using a static threshold rather than an adaptive accumulation scheme [S1].

## How it works
**Inputs:** A sparse voxel structure (e.g., Sparse Voxel Octree or DAG) [S1]; a shading point $P$; a cone angle threshold $\theta_{thresh}$ [S1].
**Outputs:** A traversal decision indicating whether to continue or terminate traversal for the current branch [S1].

**Data Structures:**
The method operates on nodes of the sparse voxel hierarchy [S1].

**Steps:**
1.  **Project Node:** Project the node onto the unit hemisphere centered at $P$ [S1].
2.  **Calculate Solid Angle:** Compute the solid angle $\Omega$ subtended by the node [S1].
3.  **Compare Threshold:** If $\Omega < \Omega_{thresh}$, terminate traversal for this branch [S1].
4.  **Continue:** If $\Omega \ge \Omega_{thresh}$, continue traversal to children or process the node [S1].

**Formulas:**
The threshold is defined by a cone angle $\theta_{thresh}$, where $\Omega_{thresh} = 2\pi(1 - \cos(\theta_{thresh}))$ [S1].

**Parameters:**
*   Cone angle threshold: $9^\circ$ (corresponding to $0.07$ sr) [S1].

**Edge Cases and Pitfalls:**
*   **Darkening:** The method results in images that are darker than ground-truth because distant contributions are ignored [S1].
*   **Quality:** Despite the darkening, it maintains high quality for near-contact occlusion [S1].
*   **Precision:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def should_traverse(node_C, point_P, theta_thresh_rad):
    # Vector from shading point to node center
    u = node_C.center - point_P
    dist_sq = dot(u, u)
    if dist_sq == 0:
        return True  # Point inside node, always traverse
    dist = sqrt(dist_sq)
    
    # Project node onto unit hemisphere and calculate solid angle [S1]
    solid_angle = compute_solid_angle_aabb(node_C, point_P)
    
    # Threshold in steradians [S1]
    omega_thresh = 0.07
    
    if solid_angle < omega_thresh:
        return False
    return True

def compute_solid_angle_aabb(box, point):
    # Standard geometric calculation for solid angle of an AABB from a point [G]
    # Implementation projects the 8 corners to the unit sphere and computes 
    # the area of the resulting spherical polygon using the spherical excess formula.
    corners = []
    hx, hy, hz = box.dimensions.x / 2, box.dimensions.y / 2, box.dimensions.z / 2
    for dx in [-hx, hx]:
        for dy in [-hy, hy]:
            for dz in [-hz, hz]:
                corners.append(vec3(box.center.x + dx, box.center.y + dy, box.center.z + dz))
    
    # Project to unit sphere
    unit_corners = [normalize(c - point) for c in corners]
    
    # Compute spherical polygon area (solid angle) by decomposing faces into triangles [G]
    solid_angle = 0.0
    faces = [
        (0, 1, 3, 2), (4, 5, 7, 6), (0, 1, 5, 4),
        (2, 3, 7, 6), (0, 2, 6, 4), (1, 3, 7, 5)
    ]
    for f in faces:
        v1, v2, v3, v4 = unit_corners[f[0]], unit_corners[f[1]], unit_corners[f[2]], unit_corners[f[3]]
        solid_angle += solid_angle_triangle(v1, v2, v3)
        solid_angle += solid_angle_triangle(v1, v3, v4)
        
    return abs(solid_angle)

def solid_angle_triangle(v1, v2, v3):
    # Oosterom and Strackee formula for solid angle of a triangle from origin [G]
    u1, u2, u3 = normalize(v1), normalize(v2), normalize(v3)
    numerator = dot(u1, cross(u2, u3))
    denominator = 1.0 + dot(u1, u2) + dot(u2, u3) + dot(u3, u1)
    return 2.0 * atan2(numerator, denominator)
```

## Subtopics

### [Octree Subdivision for Multiple Scattering](3356-octree-subdivision-for-multiple-scattering.md)
Not given in the sources.

## Operating characteristics
**Time and Memory Complexity:**
The method reduces traversal time by skipping distant nodes [S1]. The memory consumption is determined by the sparse voxel structure (e.g., DAG) [S1].

**Measured Performance:**
In the SANMIGUEL scene, the method provides faster traversal speed for AO rays compared to full traversal [S1].

**Scaling:**
The method scales with the density of the scene and the chosen threshold [S1].

**Strengths and Limitations:**
*   **Strengths:** Faster than standard traversal; maintains high quality for near-contact occlusion [S1].
*   **Limitations:** Results in images that are darker than ground-truth [S1].

**When to Use:**
Use when performance is critical and slight darkening in distant areas is acceptable [S1].

## Background
The method was introduced in the context of high-resolution sparse voxel DAGs [S1]. It leverages the properties of the SVO and DAG scene representations to optimize shadow ray casting [S1].

## Sources
[S1] High resolution sparse voxel DAGs, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
