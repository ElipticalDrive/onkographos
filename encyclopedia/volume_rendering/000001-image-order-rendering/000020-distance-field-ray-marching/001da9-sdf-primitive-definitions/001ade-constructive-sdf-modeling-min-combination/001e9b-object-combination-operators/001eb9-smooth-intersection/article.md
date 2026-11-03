# Smooth Intersection
Also called: S-Intersection

## Overview
Smooth Intersection is a constructive SDF operator that blends the overlap of two shapes to produce a rounded interior corner, serving as the smooth counterpart to the boolean intersection [S1, unofficial][S2, unofficial]. Unlike standard smooth minimums which approximate unions, this operator approximates intersections by mixing the input distances based on their relative positions within the blending radius [S1, unofficial][S2, unofficial]. It is distinguished from its sibling [SDF Smooth Union](1817-sdf-smooth-union.md) by its focus on the interior region where both distances are negative, whereas the union focuses on the exterior where at least one distance is positive [G].

## How it works
**Inputs and Outputs:**
- Inputs: Two scalar signed distance values $d_1$ and $d_2$, and a smoothing parameter $k$ (radius) [S1, unofficial][S2, unofficial].
- Output: A single scalar signed distance value $d$ [S1, unofficial][S2, unofficial].

**Data Structures:**
None required; the operation is scalar [G].

**Parameters:**
- $k$: The blending radius or smoothness parameter. It controls the width of the transition zone between the sharp intersection and the smooth blend [S1, unofficial][S2, unofficial].

**Formulas:**
The operator computes an intermediate blend factor $h$ and then mixes the distances [S1, unofficial][S2, unofficial]:

$$
h = \text{clamp}\left(0.5 - 0.5 \frac{d_2 - d_1}{k}, 0.0, 1.0\right)
$$

$$
d = \text{mix}(d_2, d_1, h) + k \cdot h \cdot (1.0 - h)
$$

Where $\text{mix}(a, b, t) = a \cdot (1 - t) + b \cdot t$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def smooth_intersection(d1: float, d2: float, k: float) -> float:
    # Calculate the blend factor h
    diff = d2 - d1
    h = 0.5 - 0.5 * diff / k
    if h < 0.0:
        h = 0.0
    elif h > 1.0:
        h = 1.0
    
    # Calculate the mixed distance
    mixed = d2 * (1.0 - h) + d1 * h
    
    # Add the smoothing term
    smooth_term = k * h * (1.0 - h)
    
    return mixed + smooth_term
```

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### SDF Smooth Union
Approximates the boolean union with rounded edges. Choose when combining objects where the exterior shape is the primary concern. [SDF Smooth Union](1817-sdf-smooth-union.md)

### Multi-Material Distance Fields
Extends SDFs to handle multiple materials with smooth transitions. Choose when rendering complex scenes with distinct material boundaries. [Multi-Material Distance Fields](1943-multi-material-distance-fields.md)

## Operating characteristics
- **Time Complexity**: $O(1)$ per evaluation [G].
- **Memory Complexity**: $O(1)$ [G].
- **Strengths**: Produces smooth, continuous transitions for interior corners; differentiable, making it suitable for gradient-based optimization [S1, unofficial][S2, unofficial].
- **Limitations**: Not given in the sources.
- **When to use**: When a smooth, rounded interior corner is desired between two intersecting volumes [G].
- **When not to use**: When sharp, precise boolean intersections are required, or when the intersecting surfaces are nearly parallel [G].

## Background
The method originates from PyTorch-based sphere tracing implementations that provide differentiable SDF operations for neural rendering and optimization [S1, unofficial][S2, unofficial]. It is part of the broader family of smooth CSG operators designed to maintain gradient continuity for training implicit surfaces [G].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
