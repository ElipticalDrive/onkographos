# SDF Transformations and Operations
Also called: SDF Boolean and Transform Operations, Smooth Blending and Domain Repetition

## Overview
This method defines the mathematical operations for modifying Signed Distance Fields (SDFs) via geometric transformations, Boolean combinations, and spatial repetitions. It solves the problem of constructing complex procedural geometry from simple primitives without explicit mesh data. What sets it apart is the ability to maintain the distance field property (Euclidean distance to the surface) through min/max blending and coordinate manipulation, enabling efficient ray marching of intricate scenes [S1, unofficial].

## How it works
### Inputs and outputs
- **Inputs**: A 3D point $\mathbf{p}$, primitive SDF functions $d_i(\mathbf{p})$, transformation matrices or functions $T(\mathbf{p})$, and blending parameters $\alpha$.
- **Outputs**: A scalar distance value $d(\mathbf{p})$.

### Data structures
- **Point**: 3D vector $\mathbf{p} = (x, y, z)$.
- **Transformation**: $3 \times 3$ rotation matrix $R$ and translation vector $\mathbf{t}$, or a composed function $T(\mathbf{p})$.
- **Blending parameter**: Scalar $\alpha > 0$ (softness).

### Steps and Formulas

1. **Basic Boolean Operations**
   Combine two SDFs $d_1(\mathbf{p})$ and $d_2(\mathbf{p})$:
   - **Union**: $d_{union}(\mathbf{p}) = \min(d_1(\mathbf{p}), d_2(\mathbf{p}))$
   - **Intersection**: $d_{intersect}(\mathbf{p}) = \max(d_1(\mathbf{p}), d_2(\mathbf{p}))$
   - **Subtraction**: $d_{sub}(\mathbf{p}) = \max(d_1(\mathbf{p}), -d_2(\mathbf{p}))$

2. **Smooth Blending (Soft Union/Intersection)**
   Approximates smooth transitions between shapes using the $\min_\alpha$ operator:
   $$ \min_\alpha(a, b) = \frac{a + b - \sqrt{(a - b)^2 + \alpha^2}}{2} $$
   - **Smooth Union**: $d_{soft\_union}(\mathbf{p}) = \min_\alpha(d_1(\mathbf{p}), d_2(\mathbf{p}))$
   - **Smooth Intersection**: $d_{soft\_intersect}(\mathbf{p}) = \max_\alpha(d_1(\mathbf{p}), d_2(\mathbf{p}))$ where $\max_\alpha(a, b) = \frac{a + b + \sqrt{(a - b)^2 + \alpha^2}}{2}$
   - **Parameter $\alpha$**: Controls softness. Smaller values approximate hard Boolean operations. Typical values: $0.01$ to $0.5$ [S1, unofficial].

3. **Geometric Transformations**
   To transform a primitive, modify the query point $\mathbf{p}$ before evaluating the SDF:
   - **Translation**: $d(\mathbf{p}) = d_{prim}(\mathbf{p} - \mathbf{t})$
   - **Rotation**: $d(\mathbf{p}) = d_{prim}(R^T (\mathbf{p} - \mathbf{t}))$
   - **Scaling**: $d(\mathbf{p}) = d_{prim}(\mathbf{s}^{-1} (\mathbf{p} - \mathbf{t})) / \min(|\mathbf{s}|)$
     - Note: Scaling distorts the distance field. The distance must be divided by the scale factor to maintain correct metric properties. Uniform scaling is preferred to avoid anisotropic distortion [S1, unofficial].

4. **Domain Repetition (Tiling)**
   Repeats a primitive pattern infinitely in space by mapping the query point into a local cell:
   $$ \mathbf{p}_{local} = \mathbf{p} - \text{round}(\mathbf{p} / \text{period}) \times \text{period} $$
   - **Implementation**: Use modulo arithmetic or floor functions to wrap coordinates.
   - **Pitfall**: Discontinuities at tile boundaries can cause artifacts if not handled carefully. The `round` function ensures symmetry around the origin of each tile [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smooth_min(a, b, alpha):
    """Smooth union approximation."""
    h = clamp(0.5 + 0.5 * (b - a) / alpha, 0.0, 1.0)
    return lerp(b, a, h) - alpha * h * (1.0 - h)

def transform_point(p, rotation_matrix, translation, scale):
    """Apply inverse transform to point for SDF evaluation."""
    # Translate
    p = p - translation
    # Rotate (inverse rotation is transpose for orthogonal matrices)
    p = dot(rotation_matrix, p)
    # Scale (divide by scale factor)
    p = p / scale
    return p

def sdf_with_transforms(p, primitive_sdf, rotation, translation, scale):
    """Evaluate SDF with geometric transformations."""
    p_transformed = transform_point(p, rotation, translation, scale)
    return primitive_sdf(p_transformed) / abs(scale)

def domain_repetition(p, period, primitive_sdf):
    """Repeat primitive pattern in 3D space."""
    p_mod = p - floor(p / period + 0.5) * period
    return primitive_sdf(p_mod)
```

## Subtopics

### [Smooth Blending and CSG Operations](14568-smooth-blending-and-csg-operations.md)
Hard Boolean operations using min/max.

### [SDF Primitive Definitions](14567-sdf-primitive-definitions.md)
Basic shapes (sphere, box, etc.).

### [SDF Transformations and Operations](14569-sdf-transformations-and-operations.md)
This article.

## Operating characteristics
- **Time Complexity**: $O(1)$ per primitive evaluation. Total time scales linearly with the number of primitives in the scene.
- **Memory Complexity**: $O(1)$ storage. No volumetric grid required.
- **Strengths**: Infinite resolution, compact representation, efficient for complex scenes with many instances.
- **Limitations**: No internal details (hollow objects are tricky), scaling distorts distance field, smooth blending introduces approximation error.
- **When to use**: Procedural geometry, soft shadows, ambient occlusion approximation, infinite tiling.
- **When not to use**: Need for exact internal geometry, non-uniform scaling with high precision requirements.

## Background
Inigo Quilez produced the first results recognizable as modern SDF in 2007, demonstrating techniques for efficient soft shadows, smooth blending, and domain repetition [S1, unofficial]. This work built upon earlier concepts by Alex Evans (2006) and Keenan Crane (2005), focusing on constraining fields to be Euclidean distances for efficient ray marching [S1, unofficial].

## Sources
[S1] Raymarching Distance Fields, https://iquilezles.org/articles/raymarchingdf/, fetched 2026-10-27 (unofficial)
