# Volume Domain Transformation for Rectangular Slicing

Also called: AABB rotation and expansion, Transformed domain V'

## Overview
Volume Domain Transformation for Rectangular Slicing is a preprocessing technique that transforms an original Axis-Aligned Bounding Box (AABB) volume domain $V$ into a transformed domain $V'$ to ensure all cutting sections are rectangles [S1]. It solves the problem of complex polygon clipping and N-gon generation inherent in slicing arbitrary AABBs with parallel planes, thereby simplifying Layered Triangle Area Light (LTC)-based integration [S1]. Unlike standard view-aligned slicing which may produce irregular polygons, this method guarantees rectangular cross-sections, avoiding the computational overhead of vertex sorting and multiple clipping branches [S1].

## How it works
**Inputs and Outputs**
*   **Input:** The original volume domain $V$, defined as an Axis-Aligned Bounding Box (AABB).
*   **Output:** A transformed volume domain $V'$ (denoted as $V^0$ in [S1]), which is a rotated and expanded AABB.
*   **Geometric Constraint:** The transformation ensures that any plane parallel to the slicing direction intersects $V'$ in a rectangular shape [S1].

**Data Structures**
The domain is represented by its bounding box extents. In the transformed domain $V'$, the box is axis-aligned in a rotated coordinate system. The specific field layout depends on the implementation of the coordinate system rotation, but conceptually it retains the AABB structure (min/max coordinates) relative to the new basis [S1].

**Steps**
The transformation is performed in two sequential steps [S1]:

1.  **Rotation:** Rotate the original AABB domain $V$ such that one of its sides perpendicularly intersects a straight line with direction $\omega_c$ [S1].
    *   $\omega_c$ is defined as the direction to the gravity center of the volume or light source [S1].
    *   This rotation aligns the volume's geometry relative to the primary viewing or lighting direction.

2.  **Expansion (Scaling):** Enlarge (scale) the rotated domain to obtain $V'$ [S1].
    *   The scaling factor is chosen such that $V'$ bounds the original domain $V$ as tightly as possible while maintaining the rectangular cutting section property [S1].
    *   The expansion compensates for the rotation, ensuring the transformed box fully contains the original volume's relevant geometry for the slicing operation [S1].

**Formulas**
Let $V$ be the original AABB.
Let $R$ be the rotation matrix that aligns a side of $V$ perpendicular to $\omega_c$.
Let $S$ be the scaling matrix applied after rotation.
The transformed domain $V'$ is defined as:
$$ V' = S \cdot R \cdot V $$
The specific scaling factor is not given as a closed-form formula in the sources, but is determined by the requirement to bound $V$ tightly while ensuring rectangular intersections [S1].

**Parameters**
*   $\omega_c$: Direction to the gravity center. Typical value: Computed from the volume's emissive or density distribution [S1].

**Edge Cases and Pitfalls**
*   **N-gon Generation:** Slicing the original AABB with arbitrary planes results in N-gons (triangles, quadrangles, pentagons, hexagons) [S1]. This requires multiple intersection tests between the cutting plane and AABB edges, followed by sorting intersection points in clockwise/anticlockwise order [S1].
*   **Clipping Complexity:** For LTC-based rendering, clipped area lights penetrating the plane require handling up to 80 different calculation branches in the shader program to handle shapes from triangles to hexagons [S1]. This is computationally expensive and unsuitable for real-time operation [S1].
*   **Precision:** The transformation avoids these precision and performance pitfalls by guaranteeing rectangular sections, which simplifies the LTC-based linear transformation and clipping logic [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transform_volume_domain(V, omega_c):
    """
    Transforms an AABB volume domain V into V' to ensure rectangular slicing sections.
    
    Args:
        V: AABB object with min and max coordinates.
        omega_c: Vector representing the direction to the gravity center.
        
    Returns:
        V_prime: Transformed AABB object.
    """
    # Step 1: Rotate V such that a side perpendicularly intersects omega_c
    # The rotation matrix R is derived from omega_c.
    # Note: The exact construction of R is not specified in [S1], 
    # but it aligns a volume side perpendicular to omega_c.
    R = compute_rotation_matrix_perpendicular_to(omega_c, V)
    V_rotated = apply_rotation(V, R)
    
    # Step 2: Expand the rotated domain to obtain V'
    # The expansion ensures V' bounds V tightly while maintaining rectangular cuts.
    # The scaling factor is determined by the geometry of V_rotated and the 
    # requirement to contain V.
    V_prime = expand_to_bound(V_rotated, V)
    
    return V_prime

def compute_rotation_matrix_perpendicular_to(omega_c, V):
    """
    Computes a rotation matrix R such that a side of V is perpendicular to omega_c.
    Implementation detail: [G] This typically involves aligning one of the 
    principal axes of V with the normal of the plane defined by omega_c 
    and the volume center, or rotating V so that omega_c lies in the plane 
    of one of V's faces.
    """
    # Placeholder for specific rotation logic not detailed in [S1]
    # The source states: "Rotate V such that a side of V perpendicularly 
    # intersects a straight line with direction omega_c"
    pass

def expand_to_bound(V_rotated, V_original):
    """
    Scales V_rotated to V_prime to tightly bound V_original.
    Implementation detail: [G] The scaling factor is likely determined by 
    the maximum extent of V_original projected onto the axes of V_rotated.
    """
    # Placeholder for specific scaling logic not detailed in [S1]
    pass
```

## Subtopics

### [Deformable Slicing Surface](11614-deformable-slicing-surface.md)
Deformable Slicing Surface (DS) is a method for generating slice geometry in hyper-dimensional radiance fields that maps spatial positions to varying ambient coordinates, replacing fixed axis-aligned planes. It solves the problem of fading artifacts and inefficient representation capacity inherent in Axis-Aligned Slicing Plane (AP) methods by allowing different parts of the scene to reference distinct regions of the ambient template space, resulting in sharper details and smoother interpolations between states.

### [Volume-Slicing Lattice Rendering](15780-volume-slicing-lattice-rendering.md)
Volume-Slicing Lattice Rendering is a real-time volumetric rendering method that approximates complex fluid volumes, specifically fire, as a lattice of hexahedral volumes aligned along parametric curves. It solves the performance bottleneck of raycasting, which requires prohibitively expensive voxel densities for real-time production, by leveraging shader hardware acceleration to merge flame elements visually. Unlike its sibling [Deformable Slicing Surface](11614-deformable-slicing-surface.md), which adapts the slicing geometry to the volume's shape, this method uses a fixed lattice structure derived from the Frenet Frame, building on the domain transformation of [Volume Domain Transformation for Rectangular Slicing](11012-volume-domain-transformation-for-rectangular-slicing.md) to ensure rectangular cutting sections.

## Operating characteristics
*   **Time Complexity:** The transformation is a preprocessing step, so its runtime cost is negligible during rendering. It significantly reduces the per-pixel or per-slice computational cost by eliminating the need for complex intersection tests, vertex sorting, and multiple clipping branches (up to 80 branches) required for N-gons [S1].
*   **Memory Complexity:** Requires storing the transformed domain $V'$ instead of $V$. The memory overhead is minimal, as it involves storing additional transformation parameters (rotation and scaling) [S1].
*   **Measured Performance:** The method enables real-time operation by simplifying the LTC-based linear transformation and clipping logic [S1]. Experiments were conducted on an Intel Core i7-8700K CPU and NVIDIA GeForce GTX 1080 GPU with 128x128x128 volume data [S1].
*   **Strengths:** Guarantees rectangular cutting sections, simplifying implementation and improving performance for LTC-based rendering [S1]. Avoids the need for complex polygon clipping [S1].
*   **Limitations:** The transformation is specific to the LTC-based rendering pipeline and may not be directly applicable to other volume rendering techniques that do not rely on rectangular area light approximations [S1]. The exact scaling factor for tight bounding is not explicitly defined, potentially leading to overestimation if not carefully computed [S1].
*   **When to use:** Use when rendering emissive inhomogeneous volumes using Layered Polygonal Area Lights (LPALs) and real-time performance is required [S1].
*   **When not to use:** Avoid if the rendering pipeline does not use LTC-based area light integration or if the volume geometry does not benefit from rectangular slicing approximations [S1].

## Background
The method was introduced in the paper "Real‐time Indirect Illumination of Emissive Inhomogeneous Volumes using Layered Polygonal Area Lights" [S1]. It addresses the limitations of slicing axis-aligned rectangular domains, which produce N-gons and require complex clipping for LTC-based rendering [S1]. The technique is part of a broader effort to enable real-time indirect illumination for volumetric data by simplifying the geometric representation of light transport [S1].

## Sources
[S1] Real‐time Indirect Illumination of Emissive Inhomogeneous Volumes using Layered Polygonal Area Lights, https://doi.org/10.1111/cgf.13851, fetched 2026-10-26
