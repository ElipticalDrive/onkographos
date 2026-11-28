# Bend Transformation
## Overview
The Bend Transformation bends 3D space along the X-axis around the Y-axis [S1, unofficial]. It serves as a spatial distortion module that modifies input coordinates before they are evaluated by distance fields or masks, enabling the creation of curved or bent geometries without altering the underlying shape definitions [S1, unofficial]. It is distinct from the sibling [Bend](2044-bend.md) method, which may imply a different axis or formulation, and builds on the rotational logic of its parent [Twist](1722-twist.md).

## How it works
**Inputs:**
- `p`: Input 3D coordinate vector $(x, y, z)$.
- `k`: Bend strength parameter (scalar).

**Outputs:**
- `p'`: Transformed 3D coordinate vector $(x', y', z')$.

**Data Structures:**
- `Vec3`: A vector type with fields `x`, `y`, `z` of type `f32`.

**Steps:**
1. Calculate the rotation angle based on the X-coordinate and the bend strength:
   $$ \theta = k \cdot p.x $$
2. Compute the sine and cosine of the angle:
   $$ c = \cos(\theta) $$
   $$ s = \sin(\theta) $$
3. Apply the rotation in the XY plane. The Z-coordinate remains unchanged:
   $$ p'.x = c \cdot p.x - s \cdot p.y $$
   $$ p'.y = s \cdot p.x + c \cdot p.y $$
   $$ p'.z = p.z $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bend_transformation(p: Vec3, k: f32) -> Vec3:
    # Step 1: Calculate angle
    angle = k * p.x
    
    # Step 2: Calculate trigonometric values
    cos_a = cos(angle)
    sin_a = sin(angle)
    
    # Step 3: Apply rotation matrix to x and y components
    # z component remains unchanged
    new_x = cos_a * p.x - sin_a * p.y
    new_y = sin_a * p.x + cos_a * p.y
    new_z = p.z
    
    return Vec3(new_x, new_y, new_z)
```

**Parameters:**
- `k`: Bend strength. Typical values depend on the desired curvature; larger absolute values result in tighter bends. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Standard floating-point precision is assumed. For very large coordinates or extreme `k` values, precision loss may occur.
- **Boundaries:** The transformation is unbounded. It does not clip or wrap coordinates.
- **Degenerate Input:** If `k=0`, the transformation returns the input vector unchanged.

## Subtopics

### [XYZ Transform](18419-xyz-transform.md)
The XYZ Transform is a module that modifies 3D coordinate space before it reaches a 3D Mask, enabling spatial operations such as moving, rotating, repeating, folding, wrapping, bending, twisting, or distorting (via noise or sine functions) the input coordinates.

## Operating characteristics
**Time Complexity:** $O(1)$ per coordinate evaluation.
**Memory Complexity:** $O(1)$ auxiliary memory.
**Strengths:** Simple, computationally inexpensive, and effective for creating smooth curved structures.
**Limitations:** Only bends along the X-axis around the Y-axis. Not suitable for complex, multi-axis distortions without composition.
**When to use:** When a simple, uniform bend along one axis is required for SDF-based modeling or coordinate space manipulation.
**When not to use:** When bending along other axes or non-uniform bending is needed.

## Background
The Bend Transformation is defined in the `sdf-math-rs` repository by bhubbard [S1, unofficial]. It is part of a suite of mathematical transformations for manipulating 3D space in the context of Signed Distance Fields (SDFs).

## Sources
[S1] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
