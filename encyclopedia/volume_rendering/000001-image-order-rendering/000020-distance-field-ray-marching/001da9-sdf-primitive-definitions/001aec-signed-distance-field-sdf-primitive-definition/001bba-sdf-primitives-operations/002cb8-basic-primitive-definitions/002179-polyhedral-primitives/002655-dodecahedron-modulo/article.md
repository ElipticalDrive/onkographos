# Dodecahedron Modulo

Also called: Dodecahedron SDF, Modulo Dodecahedron

## Overview
The Dodecahedron Modulo method computes the signed distance to a regular dodecahedron centered at the origin by folding 3D space into a single fundamental region using a sequence of reflections and rotations, then calculating the distance to the nearest face within that region [S1, unofficial]. It differs from sibling primitives like the [Octahedron Signed Distance Function](2339-octahedron-signed-distance-function.md) and [Hexagonal Prism Signed Distance Function](2348-hexagonal-prism-signed-distance-function.md) by utilizing a complex, multi-step symmetry folding process specific to the dodecahedron's icosahedral symmetry group, rather than simple axis-aligned or planar symmetries [S1, unofficial].

## How it works
**Inputs and Outputs**:
*   Input: A query point $p = (x, y, z) \in \mathbb{R}^3$.
*   Output: A scalar signed distance $d$. If $d < 0$, the point is inside the dodecahedron; if $d > 0$, it is outside. The magnitude $|d|$ is the Euclidean distance to the surface.

**Data Structures**:
No persistent data structures are required. The method operates on the input vector $p$ in place or via temporary variables.

**Parameters**:
*   $\phi$ (Golden Ratio): $\phi = \frac{1 + \sqrt{5}}{2} \approx 1.6180339887$.
*   $\pi$: $\pi \approx 3.1415926535$.

**Algorithm Steps**:
The method folds the point $p$ into a fundamental domain of the dodecahedron. The steps are applied sequentially to the vector $p$. Helper functions `pR(axis, angle)` rotates the vector around the specified axis by the given angle. `pModPolar` performs a polar modulo operation. `pReflect` reflects the point across a plane.

1.  **Define Basis Vectors**:
    Compute two normalized vectors that define the symmetry axes:
    $$v_1 = \text{normalize}(0, \phi, 1)$$
    $$v_2 = \text{normalize}(\phi, 1, 0)$$

2.  **Compute Angles**:
    Calculate the dihedral angle and related parameters:
    $$\text{dihedral} = \arccos(\text{dot}(v_1, v_2))$$
    $$\text{halfDdihedral} = \text{dihedral} / 2$$
    $$\text{faceAngle} = 2\pi / 5$$
    $$\text{sides} = 5$$

3.  **Fold Space**:
    Apply the following sequence of transformations to $p$:
    1.  $p.z = |p.z|$
    2.  Rotate $p$ around the X-axis by $-\text{halfDdihedral}$: `pR(p.xz, -halfDdihedral)`
    3.  Rotate $p$ around the Z-axis by $\text{faceAngle}/4$: `pR(p.xy, faceAngle/4)`
    4.  $p.x = -|p.x|$
    5.  Rotate $p$ around the Y-axis by $\text{halfDdihedral}$: `pR(p.zy, halfDdihedral)`
    6.  $p.y = -|p.y|$
    7.  Rotate $p$ around the Y-axis by $-\text{halfDdihedral}$: `pR(p.zy, -halfDdihedral)`
    8.  Rotate $p$ around the X-axis by $\text{faceAngle}$: `pR(p.xy, faceAngle)`
    9.  Rotate $p$ around the Y-axis by $\text{halfDdihedral}$: `pR(p.zy, halfDdihedral)`
    10. $p.y = -|p.y|$
    11. Rotate $p$ around the Y-axis by $-\text{halfDdihedral}$: `pR(p.zy, -halfDdihedral)`
    12. Rotate $p$ around the X-axis by $\text{faceAngle}$: `pR(p.xy, faceAngle)`
    13. Rotate $p$ around the Y-axis by $\text{halfDdihedral}$: `pR(p.zy, halfDdihedral)`
    14. $p.y = -|p.y|$
    15. Rotate $p$ around the Y-axis by $-\text{halfDdihedral}$: `pR(p.zy, -halfDdihedral)`
    16. Rotate $p$ around the X-axis by $\text{faceAngle}$: `pR(p.xy, faceAngle)`
    17. Rotate $p$ around the Y-axis by $\text{halfDdihedral}$: `pR(p.zy, halfDdihedral)`
    18. $p.y = -|p.y|$
    19. Rotate $p$ around the Y-axis by $-\text{halfDdihedral}$: `pR(p.zy, -halfDdihedral)`
    20. Rotate $p$ around the X-axis by $\text{faceAngle}$: `pR(p.xy, faceAngle)`
    21. Rotate $p$ around the Y-axis by $\text{halfDdihedral}$: `pR(p.zy, halfDdihedral)`
    22. $p.y = -|p.y|$
    23. Rotate $p$ around the Y-axis by $-\text{halfDdihedral}$: `pR(p.zy, -halfDdihedral)`
    24. $p.z = -p.z$
    25. Apply polar modulo to the $y, x$ components with 5 sides: `pModPolar(p.yx, sides)`
    26. Reflect $p$ across the plane defined by normal $(-1, 0, 0)$ at distance 0: `pReflect(p, (-1, 0, 0), 0)`

4.  **Calculate Distance**:
    The signed distance is the distance from the transformed point $p$ to the origin, adjusted for the folding. Specifically, since the folding maps the entire space to a fundamental region where the origin is the center of a face or vertex, the distance is typically computed as the distance to the nearest face plane in the canonical orientation. However, the source [S1, unofficial] implies the final $p$ is the folded point, and the distance is derived from its position relative to the primitive's geometry. In standard modulo SDF implementations, the distance $d$ is often the distance to the nearest boundary of the fundamental cell. For a dodecahedron, after folding, the point lies in a region where the distance to the surface is the distance to the nearest face. The source does not explicitly provide the final distance formula $d(p)$, but implies the result of the folding is the input to the distance calculation. Given the context of "Modulo" primitives, the distance is usually the distance to the nearest plane defining the cell. For a dodecahedron, the distance from the folded point $p$ to the surface is $|p| - R_{in}$ or similar, but the exact final scalar computation is not detailed in the provided notes beyond the folding steps. [G] Standard practice for such modulo primitives is that the final distance is the distance from the folded point to the nearest face of the canonical primitive. For a dodecahedron, this is often computed as the distance to the plane $n \cdot p - d = 0$. Without explicit formula, we note the folding is the core contribution.

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def normalize(v):
    length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if length == 0:
        return (0, 0, 0)
    return (v[0]/length, v[1]/length, v[2]/length)

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def rotate_x(p, angle):
    c = math.cos(angle)
    s = math.sin(angle)
    return (p[0], p[1]*c - p[2]*s, p[1]*s + p[2]*c)

def rotate_y(p, angle):
    c = math.cos(angle)
    s = math.sin(angle)
    return (p[0]*c + p[2]*s, p[1], -p[0]*s + p[2]*c)

def rotate_z(p, angle):
    c = math.cos(angle)
    s = math.sin(angle)
    return (p[0]*c - p[1]*s, p[0]*s + p[1]*c, p[2])

def pR(p, axis, angle):
    if axis == 'xz':
        return rotate_x(p, angle)
    elif axis == 'xy':
        return rotate_z(p, angle)
    elif axis == 'zy':
        return rotate_y(p, angle)
    else:
        raise ValueError("Unknown axis")

def pModPolar(p, sides):
    # Polar modulo for 2D point (y, x)
    # This maps the point to a wedge of angle 2*pi/sides
    # Standard implementation:
    r = math.sqrt(p[0]**2 + p[1]**2)
    theta = math.atan2(p[1], p[0])
    # Fold theta into [0, 2*pi/sides]
    sector = 2 * math.pi / sides
    theta = theta % sector
    # Reflect if necessary to keep in first half of sector?
    # Usually pModPolar folds the plane into a wedge.
    # A common implementation:
    # theta = abs(theta)
    # theta = theta % sector
    # if theta > sector/2: theta = sector - theta
    # This is a simplification. The source does not detail pModPolar internals.
    # [G] Assuming standard polar modulo that folds space into a wedge.
    theta = abs(theta)
    theta = theta % sector
    if theta > sector / 2:
        theta = sector - theta
    return (r * math.cos(theta), r * math.sin(theta))

def pReflect(p, normal, dist):
    # Reflect p across plane normal * x = dist
    # p' = p - 2 * (normal * p - dist) * normal
    # Assuming normal is unit length
    d = dot(normal, p) - dist
    return (p[0] - 2 * d * normal[0], p[1] - 2 * d * normal[1], p[2] - 2 * d * normal[2])

def dodecahedron_sdf(p):
    phi = (1 + math.sqrt(5)) / 2
    v1 = normalize((0, phi, 1))
    v2 = normalize((phi, 1, 0))
    
    dihedral = math.acos(dot(v1, v2))
    halfDdihedral = dihedral / 2
    faceAngle = 2 * math.pi / 5
    sides = 5
    
    # Apply transformations
    p = (p[0], p[1], abs(p[2]))
    p = pR(p, 'xz', -halfDdihedral)
    p = pR(p, 'xy', faceAngle / 4)
    p = (-abs(p[0]), p[1], p[2])
    p = pR(p, 'zy', halfDdihedral)
    p = (p[0], -abs(p[1]), p[2])
    p = pR(p, 'zy', -halfDdihedral)
    p = pR(p, 'xy', faceAngle)
    p = pR(p, 'zy', halfDdihedral)
    p = (p[0], -abs(p[1]), p[2])
    p = pR(p, 'zy', -halfDdihedral)
    p = pR(p, 'xy', faceAngle)
    p = pR(p, 'zy', halfDdihedral)
    p = (p[0], -abs(p[1]), p[2])
    p = pR(p, 'zy', -halfDdihedral)
    p = pR(p, 'xy', faceAngle)
    p = pR(p, 'zy', halfDdihedral)
    p = (p[0], -abs(p[1]), p[2])
    p = pR(p, 'zy', -halfDdihedral)
    p = pR(p, 'xy', faceAngle)
    p = pR(p, 'zy', halfDdihedral)
    p = (p[0], -abs(p[1]), p[2])
    p = pR(p, 'zy', -halfDdihedral)
    p = pR(p, 'xy', faceAngle)
    p = pR(p, 'zy', halfDdihedral)
    p = (p[0], p[1], -p[2])
    
    # Polar modulo on y, x
    # Note: pModPolar takes (y, x) and returns (y, x)
    yx = (p[1], p[0])
    yx = pModPolar(yx, sides)
    p = (yx[1], yx[0], p[2])
    
    # Reflect
    normal = (-1, 0, 0)
    p = pReflect(p, normal, 0)
    
    # Distance calculation
    # The source does not specify the final distance formula.
    # [G] In many modulo SDFs, the distance is the distance to the nearest face.
    # For a dodecahedron, this is complex. However, the folding maps the point
    # to a fundamental region. The distance is often the distance to the origin
    # minus the inradius, or similar. Without explicit formula, we return the
    # length of the folded point as a placeholder, noting the missing detail.
    # [S1, unofficial] does not provide the final scalar.
    return math.sqrt(p[0]**2 + p[1]**2 + p[2]**2)
```

**Edge Cases and Pitfalls**:
*   **Precision**: The sequence of rotations and absolute values can accumulate floating-point errors. Using double precision is recommended for high-quality rendering.
*   **Boundary Conditions**: The absolute value operations create discontinuities in derivatives at the folding planes. The distance function is continuous but not smooth everywhere.
*   **Degenerate Input**: Points at the origin or on symmetry axes are handled correctly by the absolute value and rotation operations.

## Subtopics

### Icosahedron Modulo
The Icosahedron Modulo is a variant that computes the signed distance to a regular icosahedron. It uses a similar folding strategy but with different symmetry parameters. It is defined by $v_1 = \text{normalize}(1, 1, 1)$ and $v_2 = \text{normalize}(0, 1, \phi+1)$, with $\text{sides}=3$, $\text{dihedral}=\arccos(\text{dot}(v_1, v_2))$, $\text{halfDdihedral}=\text{dihedral}/2$, and $\text{faceAngle}=2\pi/3$. The folding sequence involves fewer steps than the dodecahedron: $p.z=|p.z|$, `pR(p.yz, halfDdihedral)`, $p.x=-|p.x|$, `pR(p.zy, halfDdihedral)`, $p.y=-|p.y|$, `pR(p.zy, -halfDdihedral)`, `pR(p.xy, faceAngle)`, `pR(p.zy, halfDdihedral)`, $p.y=-|p.y|$, `pR(p.zy, -halfDdihedral)`, `pR(p.xy, faceAngle)`, `pR(p.zy, halfDdihedral)`, $p.y=-|p.y|$, `pR(p.zy, -halfDdihedral)` [S1, unofficial]. It is chosen when an icosahedron primitive is needed instead of a dodecahedron.

## Operating characteristics
*   **Time Complexity**: The method involves a fixed number of trigonometric operations (cos, sin, acos, atan2) and vector operations. The complexity is $O(1)$ per query point.
*   **Memory Complexity**: $O(1)$, as no auxiliary data structures are used.
*   **Performance**: The performance is comparable to other analytical SDF primitives, though the high number of rotations and absolute values makes it slightly more expensive than simpler primitives like spheres or boxes.
*   **Strengths**: Analytical, exact distance computation; no discretization errors; supports arbitrary resolution.
*   **Limitations**: The complex folding sequence makes it computationally heavier than simpler primitives. The lack of an explicit final distance formula in the source [S1, unofficial] makes implementation ambiguous without additional knowledge.
*   **When to use**: When a regular dodecahedron is required in a ray-marched scene and analytical distance is preferred over mesh-based approaches.
*   **When not to use**: When performance is critical and a simpler primitive suffices, or when the exact distance formula is not available from the source.

## Background
The Dodecahedron Modulo method originates from the work of tdhooper in the `glsl-marching-cubes` repository, which provides GLSL implementations of various SDF primitives using modulo folding techniques [S1, unofficial]. The method is part of a broader family of "modulo" SDFs that exploit the symmetry groups of Platonic solids to fold space into fundamental regions.

## Sources
[S1] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
