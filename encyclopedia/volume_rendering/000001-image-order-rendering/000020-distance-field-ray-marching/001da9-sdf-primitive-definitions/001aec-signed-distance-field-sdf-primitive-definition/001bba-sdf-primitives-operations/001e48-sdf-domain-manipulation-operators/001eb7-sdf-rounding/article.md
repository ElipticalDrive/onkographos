# SDF Rounding

Also called: Distance Offset, Rounding, Rounding Transformation, SDF Offset

## Overview
SDF Rounding is a geometric transformation that rounds the edges of a shape by a specified radius [S1, unofficial][S2, unofficial]. It operates by subtracting a constant distance value from the SDF result [S2, unofficial], distinguishing it from [SDF Domain Manipulation Operators](1730-sdf-domain-manipulation-operators.md) which modify the coordinate space, and [SDF Primitives](1731-sdf-primitives.md) which define base shapes [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A query point $p \in \mathbb{R}^3$ and a base SDF function $d(p)$ returning the signed distance to the primitive surface [G].
- Input: A rounding radius $r \ge 0$ [S1, unofficial][S2, unofficial].
- Output: The rounded signed distance $d_{\text{rounded}}(p)$ [S1, unofficial][S2, unofficial].

**Data Structures:**
- The SDF function $d(p)$ is implemented as a callable returning a scalar or tensor [G].
- The radius $r$ is stored as a scalar float [G].

**Steps:**
1. Evaluate the base SDF at the query point $p$ to obtain the signed distance $d = d(p)$ [G].
2. Subtract the rounding radius $r$ from the distance value [S1, unofficial].
3. Return the result $d_{\text{rounded}} = d - r$ [S2, unofficial].

**Formula:**
$$ d_{\text{rounded}}(p) = d(p) - r $$
Where:
- $d(p)$ is the signed distance from point $p$ to the original primitive surface [G].
- $r$ is the rounding radius parameter [S2, unofficial].

**Parameters:**
- $r$: The distance at which to perform the rounding [S2, unofficial]. A value of $0.1$ is used as an example in reference implementations [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $r$ exceeds the minimum feature size of the primitive, the zero-crossing shifts inward, potentially collapsing thin features or inverting the signed distance sign [G].
- The sign of the distance is preserved relative to the original surface, but the surface location moves inward by $r$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_rounding(sdf_func, radius):
    """
    Applies SDF rounding to a base signed distance function.
    
    Args:
        sdf_func: A callable that takes a point p (vec3) and returns the signed distance.
        radius: A float representing the rounding radius r.
        
    Returns:
        A callable that returns the rounded signed distance.
    """
    def rounded_sdf(p):
        # Evaluate the base SDF
        d = sdf_func(p)
        # Subtract the radius to round the edges
        return d - radius
    return rounded_sdf
```

## Subtopics

### [SDF Onion](1846-sdf-onion.md)
Creates a hollow shell by taking the absolute value of the distance and subtracting a radius. It is a specific application of SDF Rounding that inverts the interior to create a void.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
