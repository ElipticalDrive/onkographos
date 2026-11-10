# Recursive Bisection Interpolation

Also called: Midpoint Refinement, Surface Shifting

## Overview
Recursive Bisection Interpolation refines edge intersection points by recursively bisecting the edge until the midpoint crosses the isovalue, reducing geometric holes between different Levels of Detail (LODs) [S1, unofficial]. It sets itself apart from its parent [Linear Intersection Approximation](1232-linear-intersection-approximation.md) by using a binary search on the scalar field values rather than a single linear interpolation step, ensuring the intersection point lies within a sub-segment that contains the zero-crossing [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Edge endpoints $a$ and $b$, their scalar field values $value_a$ and $value_b$, and the target isovalue $iso$ [S1, unofficial].
- Output: Intersection point $p$ on the edge [S1, unofficial].

**Data Structures:**
Not given in the sources.

**Parameters:**
- `surface_shift` (int): Boolean flag (0 or 1) to enable recursive bisection interpolation instead of standard linear interpolation [S1, unofficial].

**Steps:**
1. Compute the midpoint $c = \frac{a + b}{2}$ and its scalar value $value_c$ [S1, unofficial].
2. Check if $(value_a - iso)(value_c - iso) < 0$; if true, recurse on half $[a, c]$ [S1, unofficial].
3. Else check if $(value_b - iso)(value_c - iso) < 0$; if true, recurse on half $[b, c]$ [S1, unofficial].
4. Otherwise, return linear interpolation on $[a, b]$ [S1, unofficial].

**Formula:**
- Linear interpolation fallback:
  $$p = a + \frac{iso - value_a}{value_b - value_a}(b - a)$$
- Midpoint crossing check:
  $$(value_a - iso)(value_c - iso) < 0$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recursive_bisection_interpolation(a, b, value_a, value_b, iso):
    # Compute midpoint
    c = (a + b) / 2.0
    
    # [G] value_c is obtained by evaluating the scalar field at position c
    value_c = field_at(c)
    
    # Check first half [a, c]
    if (value_a - iso) * (value_c - iso) < 0:
        return recursive_bisection_interpolation(a, c, value_a, value_c, iso)
    
    # Check second half [b, c]
    if (value_b - iso) * (value_c - iso) < 0:
        return recursive_bisection_interpolation(b, c, value_b, value_c, iso)
    
    # Fallback to linear interpolation
    if value_b == value_a:
        return a  # [G] Fallback for degenerate case where values are identical
    
    t = (iso - value_a) / (value_b - value_a)
    return a + t * (b - a)

def field_at(p):
    # [G] Standard scalar field evaluation (e.g., trilinear interpolation from voxel grid)
    raise NotImplementedError("Scalar field evaluation must be provided by the renderer.")
```

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### [Extended Time Lag for Parallel Boundaries](5675-extended-time-lag-for-parallel-boundaries.md)
A geometric constraint technique used during hierarchical isosurface extraction to prevent refinement dependencies and topological artifacts at the interfaces of adjacent sub-blocks. It extends the standard time lag method by enforcing a local inclusion check against block boundaries, ensuring that edge contractions do not create geometric inconsistencies between neighbors. Unlike the original time lag which applies globally, this extension specifically targets the borders of current sub-blocks to maintain consistency in parallel or partitioned rendering environments.

## Operating characteristics
- Strengths: Reduces geometric holes between different LODs by ensuring intersection points are more accurately located within the scalar field's zero-crossing [S1, unofficial].
- Limitations: Not given in the sources.
- Measured Performance: Not given in the sources.
- Time Complexity: Not given in the sources.
- Memory Complexity: Not given in the sources.
- When to use: Not given in the sources.
- When not to use: Not given in the sources.

## Background
Recursive Bisection Interpolation, also known as Surface Shifting, is implemented in the OpenGL Marching Cubes/Tetrahedra Geometry Shader framework by AnBenLa [S1, unofficial]. It is used in conjunction with Simple LOD and Transvoxel algorithm implementations to improve mesh quality [S1, unofficial]. The method was described in the context of a GitHub repository for real-time volume visualization [S1, unofficial].

## Sources
[S1] AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, https://github.com/AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, fetched 2026-10-27 (unofficial)
