# Volume Domain Transformation for Rectangular Slicing

Also called: AABB rotation and expansion, Transformed domain V'

## Overview
This method is a preprocessing step that transforms the original Axis-Aligned Bounding Box (AABB) volume domain $V$ into a transformed domain $V'$ to ensure all cutting sections are rectangles [S1]. It solves the problem of complex polygon clipping and intersection testing inherent in slicing an AABB with arbitrary planes, which results in N-gons (triangles, quadrangles, pentagons, hexagons) [S1]. By guaranteeing rectangular cross-sections, it simplifies Layered Polygonal Area Light (LTC)-based integration and avoids the need for handling up to 80 different calculation branches for clipped area lights penetrating the plane [S1]. It builds on [View-Aligned Slicing (Proxy Geometry Generation)](7620-view-aligned-slicing-proxy-geometry-generation.md).

## How it works
**Inputs:**
- Original volume domain $V$, defined as an Axis-Aligned Bounding Box (AABB).
- View direction or gravity center direction $\omega_c$.

**Outputs:**
- Transformed volume domain $V'$, where any planar cut perpendicular to $\omega_c$ yields a rectangular intersection.

**Data Structures:**
- The domain is represented by its bounding box extents. The transformation involves a rotation matrix and a scaling factor (expansion).

**Steps:**
1. **Rotation:** Rotate the original AABB domain $V$ such that one of its sides becomes perpendicular to the straight line defined by the direction $\omega_c$ (direction to the gravity center) [S1].
2. **Expansion:** Enlarge (scale) the rotated domain to obtain $V'$ such that it bounds the original domain $V$ as tightly as possible while ensuring that all cutting sections remain rectangles [S1].

**Formulas:**
Let $R$ be the rotation matrix aligning a side of the AABB with the normal of the cutting plane (perpendicular to $\omega_c$). Let $S$ be the scaling factor applied to the rotated domain.
$$ V' = S \cdot (R \cdot V) $$
The scaling $S$ is determined by the requirement that the projection of the original volume onto the cutting plane fits within the rectangular bounds of the transformed domain without introducing non-rectangular intersections [S1].

**Parameters:**
- $\omega_c$: Direction to the gravity center. Not given in the sources.
- Scaling factor: Determined by the tightest bound requirement. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Original AABB Slicing:** Slicing an AABB with arbitrary planes results in N-gons. This requires multiple intersection tests between the cutting plane and edges of the AABB, followed by sorting the intersection points in clockwise/anticlockwise order [S1].
- **LTC Implementation:** For LTC-based rendering, clipped area lights penetrating the plane require handling up to 80 different calculation branches for triangles to hexagons, which is computationally expensive and unsuitable for real-time performance [S1].
- **Roughness:** The method performs sufficiently well for low roughness surfaces, producing results almost identical to path tracing. On highly rough surfaces, the blurriness is smaller than path tracing, yet the approximation holds [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transform_volume_domain(V, omega_c):
    """
    Transforms AABB volume domain V into V' to ensure rectangular slices.
    
    Args:
        V: AABB defined by min and max corners.
        omega_c: Vector direction to gravity center.
        
    Returns:
        V_prime: Transformed AABB.
    """
    # Step 1: Rotate V such that a side is perpendicular to omega_c
    # The rotation matrix R aligns one axis of the AABB with the normal of the cutting plane
    # defined by omega_c.
    R = compute_rotation_matrix_to_align_with_normal(omega_c)
    V_rotated = apply_rotation(V, R)
    
    # Step 2: Expand V_rotated to V' to bound V tightly while maintaining rectangular cuts
    # The expansion ensures that the projection of the original volume fits within the
    # rectangular bounds of the transformed domain.
    # The exact scaling factor depends on the geometry of V and the orientation.
    scaling_factor = compute_tightest_scaling_factor(V, V_rotated)
    V_prime = scale_abb(V_rotated, scaling_factor)
    
    return V_prime

def compute_rotation_matrix_to_align_with_normal(normal):
    """
    Computes a rotation matrix that aligns one of the standard basis vectors
    (representing a side of the AABB) with the given normal vector.
    [G] Standard basis alignment logic: Find the basis vector most aligned with the normal,
    then rotate that basis vector to the normal.
    """
    # Identify the axis of the AABB that will become perpendicular to the cutting plane
    # This axis should be parallel to omega_c after rotation.
    # We need a rotation that maps one of the AABB's local axes (e.g., X, Y, or Z)
    # to the direction of omega_c.
    # For simplicity, assume we map the Z-axis to omega_c.
    # Rotation from Z-axis (0,0,1) to normal (nx, ny, nz).
    # Using Rodrigues' rotation formula or quaternion slerp.
    axis_z = [0, 0, 1]
    rotation = align_vectors(axis_z, normal)
    return rotation

def align_vectors(v1, v2):
    """
    Returns a rotation matrix that rotates vector v1 to align with v2.
    [G] Standard vector alignment implementation.
    """
    v1_norm = normalize(v1)
    v2_norm = normalize(v2)
    dot_product = dot(v1_norm, v2_norm)
    
    # If vectors are parallel or anti-parallel, handle separately
    if abs(dot_product - 1.0) < 1e-6:
        return identity_matrix()
    if abs(dot_product + 1.0) < 1e-6:
        # 180 degree rotation around any perpendicular axis
        # Find a perpendicular axis
        perp = cross(v1_norm, [1, 0, 0])
        if length(perp) < 1e-6:
            perp = cross(v1_norm, [0, 1, 0])
        perp = normalize(perp)
        return rotation_matrix_180(perp)
    
    axis = cross(v1_norm, v2_norm)
    angle = acos(dot_product)
    return rotation_matrix(axis, angle)

def compute_tightest_scaling_factor(V, V_rotated):
    """
    Computes the scaling factor to expand V_rotated such that it bounds V tightly.
    [G] The scaling factor is determined by the maximum extent of the original volume V
    when projected onto the axes of V_rotated.
    """
    # Project original volume V onto the axes of the rotated frame
    # Find the maximum extent required to cover V
    max_extent = compute_max_projection_extent(V, V_rotated)
    # The scaling factor is the ratio of the required extent to the current extent
    # Assuming V_rotated is initially the same size as V, just rotated.
    # We need to expand it to cover the "shadow" of V in the new orientation.
    # This is a simplification; the exact formula depends on the specific geometry.
    # The source states "enlarge the domain to obtain V' such that it bounds the original domain V as tightly as possible".
    return max_extent

def scale_abb(abb, factor):
    """
    Scales an AABB by a factor.
    """
    center = (abb.min + abb.max) / 2
    half_extents = (abb.max - abb.min) / 2
    new_half_extents = half_extents * factor
    new_min = center - new_half_extents
    new_max = center + new_half_extents
    return AABB(new_min, new_max)
```

## Subtopics

## Operating characteristics
- **Time Complexity:** Avoids the $O(N)$ intersection tests and vertex sorting required for N-gon clipping in the original domain [S1]. Eliminates the need for up to 80 calculation branches in the shader for clipped area lights [S1].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Experiments were conducted on an Intel Core i7-8700K 3.7 GHz CPU and NVIDIA GeForce GTX 1080 GPU with 32 GB RAM [S1]. Volume data was $128 \times 128 \times 128$ 3D textures [S1]. Rendering used $N=128$ LPALs [S1].
- **Strengths:** Simplifies computation and implementation by ensuring all cutting sections are rectangles [S1]. Enables real-time operation by avoiding complex polygon clipping [S1].
- **Limitations:** The transformation may introduce some blurriness on highly rough surfaces, though it is smaller than path tracing [S1].
- **When to use:** When using LTC-based rendering for emissive inhomogeneous volumes and real-time performance is required [S1].
- **When not to use:** When high roughness surfaces require exact path tracing results, as the method approximates the blurriness [S1].

## Background
- **Origin:** Proposed in "Real‐time Indirect Illumination of Emissive Inhomogeneous Volumes using Layered Polygonal Area Lights" [S1].
- **History:** The method addresses the computational bottleneck of slicing AABBs with arbitrary planes, which results in N-gons and complex clipping logic for LTC-based area lights [S1].
- **Usage:** Used in real-time indirect illumination rendering of emissive inhomogeneous volumes [S1].

## Sources
[S1] Real‐time Indirect Illumination of Emissive Inhomogeneous Volumes using Layered Polygonal Area Lights, https://doi.org/10.1111/cgf.13851, fetched 2026-10-26
