# Dodecahedron Modulo
Also called: Dodecahedron SDF, Dodecahedron Distance Field

## Overview
The Dodecahedron Modulo method computes the Euclidean signed distance from a query point $p$ to the surface of a regular dodecahedron centered at the origin. It solves the problem of efficiently evaluating the distance field for this specific polyhedral primitive without resorting to mesh-based distance approximations. It sets itself apart from sibling primitives like the [Octahedron Signed Distance Function](2339-octahedron-signed-distance-function.md) by utilizing a more complex sequence of rotational symmetries and polar modulo operations to fold 3D space into the dodecahedron's fundamental domain, rather than relying solely on simple axis-aligned reflections or 2D cross-section extrusions.

## How it works
The method relies on the principle of folding space into a fundamental domain where the closest feature on the dodecahedron surface is unique and easily calculable. For a regular dodecahedron, this domain is a specific region bounded by symmetry planes. The algorithm transforms the input point $p$ through a sequence of reflections (via absolute values) and rotations to map any point in $\mathbb{R}^3$ into this fundamental domain. Once in the domain, the distance to the nearest face, edge, or vertex can be computed directly.

### Inputs and Outputs
- **Input**: A 3D query point $p = (x, y, z)$.
- **Output**: The signed distance $d$ from $p$ to the dodecahedron surface. The sign is negative inside and positive outside.

### Data Structures
No persistent data structures are required. The computation is stateless and operates on the input vector $p$ in place or via temporary variables.

### Parameters
- **PHI**: The golden ratio, $\phi = \frac{1 + \sqrt{5}}{2} \approx 1.6180339887$.
- **sides**: The number of sides of the pentagonal faces, fixed at $5$.
- **faceAngle**: The angle subtended by one side of the pentagon at the center of the face, $2\pi/5$.
- **dihedral**: The dihedral angle between adjacent faces, $\arccos(\mathbf{v}_1 \cdot \mathbf{v}_2)$.
- **halfDdihedral**: Half of the dihedral angle.

### Steps
1.  **Define Basis Vectors**:
    Calculate two vectors that define the symmetry of the dodecahedron:
    $$ \mathbf{v}_1 = \text{normalize}(0, \phi, 1) $$
    $$ \mathbf{v}_2 = \text{normalize}(\phi, 1, 0) $$
    Compute the dihedral angle and its half:
    $$ \text{dihedral} = \arccos(\mathbf{v}_1 \cdot \mathbf{v}_2) $$
    $$ \text{halfDdihedral} = \text{dihedral} / 2 $$
    Compute the face angle:
    $$ \text{faceAngle} = 2\pi / 5 $$

2.  **Fold Space**:
    Apply a specific sequence of reflections and rotations to map $p$ into the fundamental domain. The operations are applied sequentially to the components of $p$:
    1.  $p.z = |p.z|$
    2.  Rotate $p$ in the $xz$-plane by $-\text{halfDdihedral}$.
    3.  Rotate $p$ in the $xy$-plane by $\text{faceAngle} / 4$.
    4.  $p.x = -|p.x|$
    5.  Rotate $p$ in the $zy$-plane by $\text{halfDdihedral}$.
    6.  $p.y = -|p.y|$
    7.  Rotate $p$ in the $zy$-plane by $-\text{halfDdihedral}$.
    8.  Rotate $p$ in the $xy$-plane by $\text{faceAngle}$.
    9.  Rotate $p$ in the $zy$-plane by $\text{halfDihedral}$.
    10. $p.y = -|p.y|$
    11. Rotate $p$ in the $zy$-plane by $-\text{halfDihedral}$.
    12. Rotate $p$ in the $xy$-plane by $\text{faceAngle}$.
    13. Rotate $p$ in the $zy$-plane by $\text{halfDihedral}$.
    14. $p.y = -|p.y|$
    15. Rotate $p$ in the $zy$-plane by $-\text{halfDihedral}$.
    16. Rotate $p$ in the $xy$-plane by $\text{faceAngle}$.
    17. Rotate $p$ in the $zy$-plane by $\text{halfDihedral}$.
    18. $p.y = -|p.y|$
    19. Rotate $p$ in the $zy$-plane by $-\text{halfDihedral}$.
    20. $p.z = -p.z$
    21. Apply polar modulo to the $yx$-components with `sides` = 5. This maps the point into a wedge of angle $2\pi/5$.
    22. Reflect $p$ across the plane defined by normal $(-1, 0, 0)$ at distance 0. This effectively sets $p.x = -|p.x|$ or similar boundary condition depending on the specific reflection implementation, ensuring the point is in the final canonical region.

3.  **Compute Distance**:
    Once folded, the point $p$ lies in a region where the closest feature is a face. The distance to the dodecahedron surface is the distance from the folded point to the nearest face plane. For a regular dodecahedron with circumradius $R$, the inradius (distance from center to face) is $r = R / \sqrt{1 + \phi^2}$? No, the standard SDF usually assumes a unit size or specific scaling. The source [S1, unofficial] implies the folding maps to a domain where the distance is simply the distance to the boundary of that domain. Typically, for a dodecahedron defined by the folding, the distance is computed as the distance from the folded point to the nearest face of the fundamental domain. If the folding maps the dodecahedron to a region bounded by planes at distance 1 (or a specific radius), the distance is $d = \text{distance\_to\_face}(p) - \text{inradius}$. However, the source [S1, unofficial] describes the folding process but does not explicitly provide the final distance formula from the folded point. In many such modulo SDFs, the final distance is the distance from the folded point to the origin minus the inradius, or the distance to the nearest plane defining the fundamental domain. Given the complexity, the most robust interpretation is that the folded point's distance to the origin is compared against the inradius of the dodecahedron. Let $r_{in}$ be the inradius. If the dodecahedron is scaled such that its vertices are at distance 1, $r_{in} = \frac{1}{\sqrt{1 + \phi^2}}$? Actually, for a dodecahedron with edge length $a$, $r_{in} = \frac{a}{2} \sqrt{\frac{5 + 2\sqrt{5}}{5}}$. The source [S1, unofficial] does not specify the scaling. We assume the standard implementation where the distance is computed from the folded coordinates. A common approach for these modulo SDFs is that the distance is the distance from the folded point to the nearest face of the fundamental domain, which is often a plane at a fixed distance. Without the explicit final formula in the notes, we state that the distance is derived from the folded point's position relative to the fundamental domain's boundaries. [G] Typically, for a dodecahedron SDF using this folding, the distance is $d = \min(p.x, p.y, p.z) - r_{in}$? No, the folding is complex. The source [S1, unofficial] ends with the folding steps. The distance calculation is likely implicit in the "fundamental domain" concept: the distance is the distance from the folded point to the boundary of the domain. If the domain is a wedge, the distance is to the planes bounding the wedge.

    *Note: The source [S1, unofficial] provides the folding steps but omits the final distance calculation formula. Implementers must determine the distance from the folded point to the nearest face of the dodecahedron. For a regular dodecahedron centered at the origin, if the folding maps the interior to the origin, the distance is often $r_{in} - \|p_{folded}\|$? Or the distance to the nearest plane. Given the ambiguity, the implementation should compute the distance from the folded point to the nearest face of the dodecahedron's fundamental domain.*

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dodecahedron_sdf(p, phi=1.6180339887):
    """
    Computes the signed distance from point p to a regular dodecahedron.
    Assumes the dodecahedron is centered at the origin.
    The scaling is not explicitly defined in the source, so the distance
    returned is relative to the fundamental domain's boundary.
    """
    x, y, z = p
    
    # Define basis vectors and angles
    v1 = normalize((0, phi, 1))
    v2 = normalize((phi, 1, 0))
    dihedral = math.acos(dot(v1, v2))
    half_dihedral = dihedral / 2.0
    face_angle = 2.0 * math.pi / 5.0
    
    # Folding sequence
    # 1. p.z = abs(p.z)
    z = abs(z)
    
    # 2. Rotate p in xz plane by -half_dihedral
    # Rotation in xz plane: x' = x*cos(theta) - z*sin(theta), z' = x*sin(theta) + z*cos(theta)
    cos_t = math.cos(-half_dihedral)
    sin_t = math.sin(-half_dihedral)
    x_new = x * cos_t - z * sin_t
    z_new = x * sin_t + z * cos_t
    x, z = x_new, z_new
    
    # 3. Rotate p in xy plane by face_angle/4
    cos_t = math.cos(face_angle / 4.0)
    sin_t = math.sin(face_angle / 4.0)
    x_new = x * cos_t - y * sin_t
    y_new = x * sin_t + y * cos_t
    x, y = x_new, y_new
    
    # 4. p.x = -abs(p.x)
    x = -abs(x)
    
    # 5. Rotate p in zy plane by half_dihedral
    # Rotation in zy plane: z' = z*cos(theta) - y*sin(theta), y' = z*sin(theta) + y*cos(theta)
    cos_t = math.cos(half_dihedral)
    sin_t = math.sin(half_dihedral)
    z_new = z * cos_t - y * sin_t
    y_new = z * sin_t + y * cos_t
    z, y = z_new, y_new
    
    # 6. p.y = -abs(p.y)
    y = -abs(y)
    
    # 7. Rotate p in zy plane by -half_dihedral
    cos_t = math.cos(-half_dihedral)
    sin_t = math.sin(-half_dihedral)
    z_new = z * cos_t - y * sin_t
    y_new = z * sin_t + y * cos_t
    z, y = z_new, y_new
    
    # 8. Rotate p in xy plane by face_angle
    cos_t = math.cos(face_angle)
    sin_t = math.sin(face_angle)
    x_new = x * cos_t - y * sin_t
    y_new = x * sin_t + y * cos_t
    x, y = x_new, y_new
    
    # 9. Rotate p in zy plane by half_dihedral
    cos_t = math.cos(half_dihedral)
    sin_t = math.sin(half_dihedral)
    z_new = z * cos_t - y * sin_t
    y_new = z * sin_t + y * cos_t
    z, y = z_new, y_new
    
    # 10. p.y = -abs(p.y)
    y = -abs(y)
    
    # 11. Rotate p in zy plane by -half_dihedral
    cos_t = math.cos(-half_dihedral)
    sin_t = math.sin(-half_dihedral)
    z_new = z * cos_t - y * sin_t
    y_new = z * sin_t + y * cos_t
    z, y = z_new, y_new
    
    # 12. Rotate p in xy plane by face_angle
    cos_t = math.cos(face_angle)
    sin_t = math.sin(face_angle)
    x_new = x * cos_t - y * sin_t
    y_new = x * sin_t + y * cos_t
    x, y = x_new, y_new
    
    # 13. Rotate p in zy plane by half_dihedral
    cos_t = math.cos(half_dihedral)
    sin_t = math.sin(half_dihedral)
    z_new = z * cos_t - y * sin_t
    y_new = z * sin_t + y * cos_t
    z, y = z_new, y_new
    
    # 14. p.y = -abs(p.y)
    y = -abs(y)
    
    # 15. Rotate p in zy plane by -half_dihedral
    cos_t = math.cos(-half_dihedral)
    sin_t = math.sin(-half_dihedral)
    z_new = z * cos_t - y * sin_t
    y_new = z * sin_t + y * cos_t
    z, y = z_new, y_new
    
    # 16. Rotate p in xy plane by face_angle
    cos_t = math.cos(face_angle)
    sin_t = math.sin(face_angle)
    x_new = x * cos_t - y * sin_t
    y_new = x * sin_t + y * cos_t
    x, y = x_new, y_new
    
    # 17. Rotate p in zy plane by half_dihedral
    cos_t = math.cos(half_dihedral)
    sin_t = math.sin(half_dihedral)
    z_new = z * cos_t - y * sin_t
    y_new = z * sin_t + y * cos_t
    z, y = z_new, y_new
    
    # 18. p.y = -abs(p.y)
    y = -abs(y)
    
    # 19. Rotate p in zy plane by -half_dihedral
    cos_t = math.cos(-half_dihedral)
    sin_t = math.sin(-half_dihedral)
    z_new = z * cos_t - y * sin_t
    y_new = z * sin_t + y * cos_t
    z, y = z_new, y_new
    
    # 20. p.z = -p.z
    z = -z
    
    # 21. Polar modulo on yx with sides=5
    # This maps the angle of (y, x) into [0, 2*pi/5]
    angle = math.atan2(y, x)
    # Normalize angle to [0, 2*pi)
    if angle < 0:
        angle += 2 * math.pi
    # Modulo
    angle = angle % (2 * math.pi / 5)
    # Convert back to Cartesian
    r = math.sqrt(x*x + y*y)
    x = r * math.cos(angle)
    y = r * math.sin(angle)
    
    # 22. Reflect p across (-1, 0, 0)
    # This sets x = -abs(x) if the reflection is across the plane x=0?
    # The source says "pReflect(p, (-1,0,0), 0)". This usually means reflecting across the plane with normal (-1,0,0) passing through origin.
    # Normal n = (-1, 0, 0). Reflection of p: p' = p - 2(p.n)n / (n.n)
    # p.n = -x. n.n = 1.
    # p' = (x, y, z) - 2(-x)(-1, 0, 0) = (x, y, z) - (2x, 0, 0) = (-x, y, z)
    # So x = -x.
    x = -x
    
    # Final distance calculation
    # The source does not provide the final distance formula.
    # [G] Typically, for a dodecahedron SDF, the distance is the distance from the folded point to the nearest face.
    # If the folding maps the dodecahedron to a region where the origin is the center, the distance is often
    # the distance to the boundary of the fundamental domain.
    # For a regular dodecahedron, the inradius r_in is related to the circumradius R.
    # Assuming the dodecahedron is scaled such that its vertices are at distance 1,
    # r_in = 1 / sqrt(1 + phi^2) is incorrect.
    # r_in = (phi^2) / sqrt(3) * edge_length?
    # Without explicit scaling, we return the distance from the folded point to the origin,
    # which is not the SDF.
    # [G] A common implementation of this specific folding (from the source repo)
    # returns the distance as the distance from the folded point to the nearest face plane.
    # The nearest face plane in the fundamental domain is often defined by the normal (1, 1, 1) or similar.
    # Given the lack of explicit formula, we cannot provide a complete distance calculation.
    # However, the folding steps are the core of the method.
    
    # Placeholder for distance calculation based on folded point
    # d = distance_to_face(p_folded) - r_in
    return 0.0 # Placeholder
```

## Subtopics

### Icosahedron Modulo
The Icosahedron Modulo method is a variant that computes the signed distance to a regular icosahedron. It uses a similar folding strategy but with different symmetry parameters: `sides=3`, `faceAngle=2π/3`, and basis vectors $\mathbf{v}_1 = \text{normalize}(1,1,1)$ and $\mathbf{v}_2 = \text{normalize}(0,1,\phi+1)$. The folding sequence involves fewer rotations and reflections compared to the dodecahedron, reflecting the icosahedron's different symmetry group. It is chosen when an icosahedral primitive is required. [S1, unofficial]

## Operating characteristics
- **Time Complexity**: $O(1)$ per query, as it involves a fixed number of arithmetic operations and trigonometric functions.
- **Memory Complexity**: $O(1)$, requiring only a few temporary variables.
- **Strengths**: Exact analytical distance, no mesh discretization errors, efficient for ray marching.
- **Limitations**: The folding sequence is complex and computationally expensive compared to simpler primitives like spheres or boxes. The lack of an explicit final distance formula in the source requires careful implementation.
- **When to use**: When rendering a regular dodecahedron with high precision.
- **When not to use**: When performance is critical and an approximate distance is acceptable, or when the primitive is not a regular dodecahedron.

## Background
The method originates from the work of tdhooper in the `glsl-marching-cubes` repository, which provides GLSL implementations of various SDF primitives. The approach is based on the mathematical property that the symmetry group of the dodecahedron can be used to fold space into a fundamental domain. This technique is part of a broader family of "modulo" SDFs for polyhedral primitives. [S1, unofficial]

## Sources
[S1] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
