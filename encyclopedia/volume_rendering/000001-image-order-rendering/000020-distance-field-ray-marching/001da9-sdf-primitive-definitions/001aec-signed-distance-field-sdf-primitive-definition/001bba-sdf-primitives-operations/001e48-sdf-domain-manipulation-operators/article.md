# SDF Domain Manipulation Operators

Also called: Domain Manipulation Operators, SDF Modifiers, SDF domain transformations, Space Partitioning, pSomething operators

## Overview
SDF Domain Manipulation Operators modify the input position vector $\mathbf{p}$ to partition space into cells, creating periodic, mirrored, or reflected spatial structures for Signed Distance Functions (SDFs) [S1, unofficial][S2, unofficial]. They transform the coordinate space before the primitive SDF is evaluated, allowing for the creation of complex patterns like grids and symmetries with minimal computational overhead [S1, unofficial][S2, unofficial]. Unlike [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md), which combines scalar fields via boolean logic, domain operators transform the geometry's location in space directly [S2, unofficial].

## Shared principles
All domain manipulation operators follow a common convention: the input position vector $\mathbf{p}$ is passed as the first argument and modified in place (`inout p`) [S1, unofficial][S2, unofficial]. For operators acting on subsets of dimensions, specific components are supplied (e.g., `<p.x>` or `<p.zx>`) [S1, unofficial].

The core mechanism involves partitioning the space into cells of a defined size. The operator maps the query point $\mathbf{p}$ to a local coordinate within its cell and returns a cell index (or identifier) [S1, unofficial][S2, unofficial]. This cell index can be used as a random seed for procedural variation or symmetry tracking [S1, unofficial][S2, unofficial].

For cell index 0, $\mathbf{p}$ remains unchanged, and cells are centered on the origin, meaning objects do not need to be manually translated to fit the grid [S1, unofficial][S2, unofficial].

The general transformation for periodic repetition in one dimension is:
$$ c = \text{floor}\left(\frac{p + \text{size}/2}{\text{size}}\right) $$
$$ p' = \text{mod}(p + \text{size}/2, \text{size}) - \text{size}/2 $$
where $c$ is the cell index and $p'$ is the transformed position [S1, unofficial][S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b):
    """Standard dot product for vectors of equal length."""
    return sum(x * y for x, y in zip(a, b))

def p_mod(p, size):
    """Maps p to the local coordinate within its cell and returns the cell index c."""
    halfsize = size * 0.5
    c = math.floor((p + halfsize) / size)
    p = (p + halfsize) % size - halfsize
    return c

def p_mod_mirror(p, size):
    """Maps p to a mirrored local coordinate within its cell and returns the cell index c."""
    halfsize = size * 0.5
    c = math.floor((p + halfsize) / size)
    p = (p + halfsize) % size - halfsize
    # Mirror every other cell
    p *= (c % 2) * 2 - 1
    return c

def p_mirror(p, dist):
    """Reflects p across a plane at distance dist and returns the sign s."""
    s = 1.0 if p >= 0.0 else -1.0
    p = abs(p) - dist
    return s

def p_reflect(p, plane_normal, offset):
    """Reflects p across a plane defined by normal and offset, returns sign of t."""
    t = dot(p, plane_normal) + offset
    if t < 0.0:
        p = p - (2.0 * t) * plane_normal
    s = 1.0 if t >= 0.0 else -1.0
    return s
```

## Subtopics

### [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md)
[CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md) creates complex geometric shapes from simple SDF primitives via boolean operations: union, intersection, and difference. It combines scalar fields using binary node operations to construct complex implicit surfaces. Unlike explicit mesh-based CSG, which requires complex boolean intersection algorithms on polygonal geometry, SDF CSG reduces scene composition to simple arithmetic on distance values. This allows for the rendering of near-tangent intersections and complex composite geometries with minimal computational overhead per sample.

### [Finite Repetition](2869-finite-repetition.md)
[Finite Repetition](2869-finite-repetition.md) creates a bounded grid of repeated instances of a base SDF primitive, limiting the replication count to a specified range in each dimension. Unlike infinite tiling operators, it restricts the domain to a finite box, preventing geometry from extending to infinity. It is chosen when a localized, tiled structure is required without the computational cost or visual artifacts of infinite repetition.

### Boolean Subtraction with Domain Repetition
Creating complex geometry by subtracting domain-repeated smaller objects from a non-repeated larger object (e.g., domain repeated cubes subtracted from a larger sphere) [S3, unofficial]. This combines domain repetition with subtracting distance fields from each other to create complex objects [S3, unofficial].

### Symmetry Y
Reflects an SDF across the XZ plane [S4, unofficial]. The step is to take the absolute value of the Y component of $\mathbf{p}$ [S4, unofficial].

### Symmetry X
Reflects an SDF across the YZ plane [S4, unofficial]. The step is to take the absolute value of the X component of $\mathbf{p}$ [S4, unofficial].

### Symmetry Z
Reflects an SDF across the XY plane [S4, unofficial]. The step is to take the absolute value of the Z component of $\mathbf{p}$ [S4, unofficial].

## Background
The concept of using domain manipulation for ray marching was popularized in the context of Signed Distance Field rendering for real-time graphics [S3, unofficial]. The techniques allow for the creation of complex geometry by combining domain repetition with subtracting distance fields [S3, unofficial].

## Sources

[S1] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S2] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S3] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S4] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
