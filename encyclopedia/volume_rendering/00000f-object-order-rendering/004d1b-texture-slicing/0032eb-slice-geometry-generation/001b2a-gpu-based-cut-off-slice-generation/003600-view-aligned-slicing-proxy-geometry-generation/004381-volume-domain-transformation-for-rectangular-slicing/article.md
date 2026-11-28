# Volume Domain Transformation for Rectangular Slicing

Also called: AABB rotation and expansion, Transformed domain V'

## Overview
Volume Domain Transformation for Rectangular Slicing is a preprocessing technique that transforms an original Axis-Aligned Bounding Box (AABB) volume domain $V$ into a transformed domain $V'$ to ensure all cutting sections are rectangles [S1]. It solves the problem of complex polygon clipping and N-gon generation inherent in slicing arbitrary AABBs with parallel planes, thereby simplifying Layered Triangle Area Light (LTC)-based integration [S1]. Unlike standard view-aligned slicing which may produce irregular polygons, this method guarantees rectangular cross-sections, avoiding the computational overhead of vertex sorting and multiple clipping branches [S1].

## How it works
**Inputs and Outputs**
*   **Input:** The original volume domain $V$, defined as an Axis-Aligned Bounding Box (AABB) [S1].
*   **Output:** A transformed volume domain $V'$ (denoted as $V^0$ in [S1]), which is a rotated and expanded AABB [S1].
*   **Geometric Constraint:** The transformation ensures that any plane parallel to the slicing direction intersects $V'$ in a rectangular shape [S1].

**Data Structures**
The domain is represented by its bounding box extents [S1]. In the transformed domain $V'$, the box remains axis-aligned relative to the new rotated basis [S1].

**Steps**
The transformation is performed in two sequential steps [S1]:

1.  **Rotation:** Rotate the original AABB domain $V$ such that one of its sides perpendicularly intersects a straight line with direction $\omega_c$ [S1].
    *   $\omega_c$ is defined as the direction to the gravity center of the volume or light source [S1].

2.  **Expansion (Scaling):** Enlarge (scale) the rotated domain to obtain $V'$ [S1].
    *   The expansion ensures $V'$ bounds the original domain $V$ as tightly as possible while maintaining the rectangular cutting section property [S1].

**Formulas**
The transformation applies a rotation followed by an axis-aligned expansion to the original bounds [S1]:
$$ V' = \text{Expand}(\text{Rotate}(V, \omega_c)) $$
The exact scaling factor for tight bounding is not given as a closed-form formula in the sources [S1].

**Parameters**
*   $\omega_c$: Direction to the gravity center [S1]. Typical value: Computed from the volume's emissive or density distribution [S1].

**Edge Cases and Pitfalls**
*   **N-gon Generation:** Slicing the original AABB with arbitrary planes results in N-gons (triangles, quadrangles, pentagons, hexagons) [S1]. This requires multiple intersection tests between the cutting plane and AABB edges, followed by sorting intersection points in clockwise/anticlockwise order [S1].
*   **Clipping Complexity:** For LTC-based rendering, clipped area lights penetrating the plane require handling up to 80 different calculation branches in the shader program to handle shapes from triangles to hexagons [S1]. This is computationally expensive and unsuitable for real-time operation [S1].
*   **Precision:** The transformation avoids these precision and performance pitfalls by guaranteeing rectangular sections, which simplifies the LTC-based linear transformation and clipping logic [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def transform_volume_domain(V_min, V_max, omega_c):
    """
    Transforms an AABB volume domain V into V' to ensure rectangular slicing sections.
    
    Args:
        V_min: List of 3 floats, minimum coordinates of the original AABB.
        V_max: List of 3 floats, maximum coordinates of the original AABB.
        omega_c: List of 3 floats, direction to the gravity center.
        
    Returns:
        V_prime_min, V_prime_max: Lists of 3 floats, bounds of the transformed AABB.
    """
    # Step 1: Rotate V such that a side perpendicularly intersects omega_c
    R = compute_rotation_matrix_aligning_z_to(omega_c)
    V_rot_min, V_rot_max = apply_rotation_to_aabb(V_min, V_max, R)

    # Step 2: Enlarge the rotated domain to tightly bound the original domain V
    # Not given in the sources.
    V_prime_min, V_prime_max = expand_aabb_to_contain(V_rot_min, V_rot_max, V_min, V_max)

    return V_prime_min, V_prime_max

def compute_rotation_matrix_aligning_z_to(target_dir):
    # Aligns local Z-axis (0,0,1) with target_dir
    t = normalize(target_dir)
    if abs(t[2]) < 0.9999:
        u = normalize(cross(t, [0, 0, 1]))
    else:
        u = normalize(cross(t, [1, 0, 0]))
    v = cross(t, u)
    # Return rotation matrix where columns are the new basis vectors
    return [[u[0], v[0], t[0]],
            [u[1], v[1], t[1]],
            [u[2], v[2], t[2]]]

def normalize(v):
    l = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return [v[0]/l, v[1]/l, v[2]/l]

def cross(a, b):
    return [a[1]*b[2] - a[2]*b[1],
            a[2]*b[0] - a[0]*b[2],
            a[0]*b[1] - a[1]*b[0]]

def apply_rotation_to_aabb(min_pt, max_pt, R):
    # Transform 8 corners of the AABB by R
    corners = []
    for x in (min_pt[0], max_pt[0]):
        for y in (min_pt[1], max_pt[1]):
            for z in (min_pt[2], max_pt[2]):
                pt = [x, y, z]
                rotated = [
                    R[0][0]*pt[0] + R[0][1]*pt[1] + R[0][2]*pt[2],
                    R[1][0]*pt[0] + R[1][1]*pt[1] + R[1][2]*pt[2],
                    R[2][0]*pt[0] + R[2][1]*pt[1] + R[2][2]*pt[2]
                ]
                corners.append(rotated)
    new_min = [min(c[0] for c in corners), min(c[1] for c in corners), min(c[2] for c in corners)]
    new_max = [max(c[0] for c in corners), max(c[1] for c in corners), max(c[2] for c in corners)]
    return new_min, new_max

def expand_aabb_to_contain(rot_min, rot_max, orig_min, orig_max):
    # Compute the tightest axis-aligned box in the rotated frame that contains the original AABB
    # The rotated basis vectors are the columns of R (stored as rows in the matrix above)
    # We project the original AABB's corners onto the rotated basis vectors
    basis = [[R[0][0], R[1][0], R[2][0]],
             [R[0][1], R[1][1], R[2][1]],
             [R[0][2], R[1][2], R[2][2]]]
    corners = []
    for x in (orig_min[0], orig_max[0]):
        for y in (orig_min[1], orig_max[1]):
            for z in (orig_min[2], orig_max[2]):
                pt = [x, y, z]
                proj = [
                    basis[0][0]*pt[0] + basis[0][1]*pt[1] + basis[0][2]*pt[2],
                    basis[1][0]*pt[0] + basis[1][1]*pt[1] + basis[1][2]*pt[2],
                    basis[2][0]*pt[0] + basis[2][1]*pt[1] + basis[2][2]*pt[2]
                ]
                corners.append(proj)
    expanded_min = [min(c[0] for c in corners), min(c[1] for c in corners), min(c[2] for c in corners)]
    expanded_max = [max(c[0] for c in corners), max(c[1] for c in corners), max(c[2] for c in corners)]
    return expanded_min, expanded_max
```

## Subtopics

### [Deformable Slicing Surface](11614-deformable-slicing-surface.md)
Deformable Slicing Surface (DS) is a method for generating slice geometry in hyper-dimensional radiance fields that maps spatial positions to varying ambient coordinates, replacing fixed axis-aligned planes. It solves the problem of fading artifacts and inefficient representation capacity inherent in Axis-Aligned Slicing Plane (AP) methods by allowing different parts of the scene to reference distinct regions of the ambient template space, resulting in sharper details and smoother interpolations between states.

### [Volume-Slicing Lattice Rendering](15780-volume-slicing-lattice-rendering.md)
Volume-Slicing Lattice Rendering is a real-time volumetric rendering method that approximates complex fluid volumes, specifically fire, as a lattice of hexahedral volumes aligned along parametric curves. It solves the performance bottleneck of raycasting, which requires prohibitively expensive voxel densities for real-time production, by leveraging shader hardware acceleration to merge flame elements visually. Unlike its sibling [Deformable Slicing Surface](11614-deformable-slicing-surface.md), which adapts the slicing geometry to the volume's shape, this method uses a fixed lattice structure derived from the Frenet Frame, building on the domain transformation of [Volume Domain Transformation for Rectangular Slicing](11012-volume-domain-transformation-for-rectangular-slicing.md) to ensure rectangular cutting sections.

## Operating characteristics
*   **Performance:** The transformation eliminates complex intersection tests, vertex sorting, and multiple clipping branches (up to 80 branches) required for N-gons, enabling real-time operation [S1].
*   **Measured Performance:** Experiments were conducted on an Intel Core i7-8700K CPU and NVIDIA GeForce GTX 1080 GPU with 32 GB of RAM [S1]. Volume data was simulated using Houdini and stored in 3D textures of size 128 × 128 × 128 [S1]. All rendering images were calculated using N = 128 LPALs [S1].
*   **Strengths:** Guarantees rectangular cutting sections, simplifying implementation and improving performance for LTC-based rendering [S1]. Avoids the need for complex polygon clipping [S1].
*   **Limitations:** The transformation is specific to the LTC-based rendering pipeline and may not be directly applicable to other volume rendering techniques that do not rely on rectangular area light approximations [S1].
*   **When to use:** Use when rendering emissive inhomogeneous volumes using Layered Polygonal Area Lights (LPALs) and real-time performance is required [S1].
*   **When not to use:** Avoid if the rendering pipeline does not use LTC-based area light integration or if the volume geometry does not benefit from rectangular slicing approximations [S1].

## Background
The method was introduced in the paper "Real‐time Indirect Illumination of Emissive Inhomogeneous Volumes using Layered Polygonal Area Lights" [S1]. It addresses the limitations of slicing axis-aligned rectangular domains, which produce N-gons and require complex clipping for LTC-based rendering [S1]. The technique is part of a broader effort to enable real-time indirect illumination for volumetric data by simplifying the geometric representation of light transport [S1].

## Sources
[S1] Real‐time Indirect Illumination of Emissive Inhomogeneous Volumes using Layered Polygonal Area Lights, https://doi.org/10.1111/cgf.13851, fetched 2026-10-26
