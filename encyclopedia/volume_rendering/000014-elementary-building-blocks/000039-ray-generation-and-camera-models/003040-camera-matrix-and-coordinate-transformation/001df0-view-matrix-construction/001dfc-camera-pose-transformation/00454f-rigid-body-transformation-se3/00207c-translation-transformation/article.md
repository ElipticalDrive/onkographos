# Translation Transformation


## Overview
A geometric transformation that moves a point by subtracting a given vector [S1, unofficial]. It operates within the rigid body transformation framework by applying a pure displacement without rotation or scaling [G].

## How it works
**Inputs and outputs:**
- Input: Point $p$ (3D vector), translation vector $t$ (3D vector).
- Output: Translated point $p'$ (3D vector).

**Data structures:**
- $p = [p_x, p_y, p_z]^T$
- $t = [t_x, t_y, t_z]^T$

**Steps:**
1. Compute the new position by subtracting the translation vector from the point: $p' = p - t$ [S1, unofficial].

**Formula:**
$$
p' = p - t
$$

**Parameters:**
- $t$: The displacement vector applied to the point [G].

**Edge cases and pitfalls:**
- Precision loss may occur for large coordinate values or very small translation vectors due to floating-point arithmetic [G].
- Boundary conditions depend on the coordinate system bounds; points may fall outside valid ranges after transformation [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def translation_transformation(p, t):
    """
    Translate point p by subtracting vector t.
    
    Args:
        p: 3D vector [x, y, z]
        t: 3D vector [tx, ty, tz]
        
    Returns:
        p_prime: Translated 3D vector
    """
    p_prime_x = p[0] - t[0]
    p_prime_y = p[1] - t[1]
    p_prime_z = p[2] - t[2]
    
    return [p_prime_x, p_prime_y, p_prime_z]
```

## Subtopics

### [Translation Matrix Construction](2873-translation-matrix-construction.md)
Constructs a 4x4 homogeneous transformation matrix that represents a pure translation by a given 4D vector [G]. It solves the problem of embedding a 3D displacement vector into a 4x4 linear algebra structure compatible with standard graphics pipelines, allowing translation to be composed with other affine transformations via matrix multiplication [G]. It differs from [Coordinate Translation](2281-translation-transformation.md) by operating on matrix algebra rather than directly manipulating coordinate data frames [G].

## Operating characteristics
Not given in the sources [G].

## Background
Origin: Not given in the sources [G].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
