# Ellipsoid Signed Distance Function

Also called: Ellipsoid Approximation, Ellipsoid Improved, Ellipsoid SDF

## Overview
Computes the signed distance from a point $p$ to an ellipsoid defined by semi-axes $r$ [S2, unofficial]. It provides a fast approximation (V1) or a more accurate iterative-free approximation (V2) [S1, unofficial].

## How it works
**Inputs:** Point $p \in \mathbb{R}^3$, semi-axes radii $r \in \mathbb{R}^3$ (where $r_i > 0$) [S1, S2, unofficial].
**Output:** Signed distance $d$.

**Data Structures:**
- $p$: 3-component vector.
- $r$: 3-component vector.

**Steps:**
1. Compute $k_1 = \| p / r \|$.
2. Compute $k_2 = \| p / (r \circ r) \|$, where $\circ$ is component-wise multiplication [S1, unofficial].
3. Return $d = \frac{k_1 (k_1 - 1.0)}{k_2}$ [S2, unofficial].

**Formula:**
$$ d = \frac{\| p / r \| (\| p / r \| - 1)}{\| p / (r \circ r) \|} $$

**Parameters:**
- $r$: Semi-axes lengths. Must be positive [S1, S2, unofficial].

**Edge Cases:**
- If $r_i = 0$, the function is undefined. [G] Clamp $r$ to a small epsilon if necessary.
- If $p = 0$, $k_1=0, k_2=0$, resulting in $0/0$. [G] Return $-\min(r)$ for the interior case.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ellipsoid_sdf(p, r):
    # p: vec3, r: vec3
    # Avoid division by zero when p is at the origin
    if p == (0.0, 0.0, 0.0):
        return -min(r)
        
    # Step 1: k1 = length(p / r)
    k1 = length(p / r)
    
    # Step 2: k2 = length(p / (r * r))
    r_sq = r * r
    k2 = length(p / r_sq)
    
    # Step 3: Return k1 * (k1 - 1.0) / k2
    # Guard against k2 == 0 to prevent inf/nan
    if k2 == 0.0:
        return -min(r)
        
    d = k1 * (k1 - 1.0) / k2
    
    return d
```

## Subtopics

### [Sphere Signed Distance Function](1237-sphere-signed-distance-function.md)
Special case where $r_x=r_y=r_z$. Use for spheres.

### [Capsule Signed Distance Function](2439-capsule-signed-distance-function.md)
Cylinder + hemispheres. Use for tubular structures.

### [Signed Box Distance Function](1092-signed-box-distance-function.md)
Axis-aligned or oriented boxes. Use for rectangular volumes.

## Operating characteristics
- **Time Complexity:** $O(1)$, constant number of arithmetic operations [G].
- **Memory Complexity:** $O(1)$.
- **Accuracy:** V2 is significantly more accurate than V1, especially for highly anisotropic ellipsoids. However, it is still an approximation and not the exact SDF [S1, unofficial].
- **Strengths:** Fast, differentiable, smooth.
- **Limitations:** Not exact. May cause artifacts in high-precision ray marching if the error exceeds the step size [G].

## Background
The ellipsoid SDF is a standard primitive in SDF libraries. V1 and V2 are attributed to Inigo Quilez [S1, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
