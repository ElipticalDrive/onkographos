# SDF Smooth Union

Also called: LogSumExp SDF Union, Quadratic Polynomial Smooth Minimum, S-Union, Smooth Minimum, Smooth Union, Smooth Union SDF Operator, Smooth Union of Spheres, Spherical SDF Blending

## Overview
SDF Smooth Union is a constructive operator that blends two Signed Distance Function (SDF) values, $a$ and $b$, into a single smooth distance field $d_{\text{out}}$, replacing the sharp geometric intersection of a boolean union with a rounded transition region controlled by a radius parameter $k$ [S1, S2, S3, S4, S6, S8, unofficial]. It sets itself apart from standard boolean operators by providing differentiable approximations of the union, available in polynomial and exponential formulations [S1, S2, S4, S7, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs:** Two scalar distance values $a$ and $b$, and a scalar smoothing radius $k > 0$ [S1, S2, S3, S6, S7, unofficial].
*   **Output:** A single scalar distance $d_{\text{out}}$ representing the smooth union [S1, S2, S3, S6, S7, unofficial].

### Data Structures
The core operator operates on scalar floats [S1, S2, S7, unofficial]. In multi-material contexts, it may operate on structs containing distance and material data, such as `SdResult(distance, material)` [S8, unofficial].

### Formulas
Two dominant formulations exist in the literature [S1, S2, S4, unofficial].

**1. Polynomial Blend**
This method uses a quadratic blend term subtracted from a linear interpolation [S1, S2, S7, unofficial].

$$h = \text{clamp}\left(0.5 + 0.5 \frac{b - a}{k}, 0.0, 1.0\right)$$

$$d_{\text{out}} = \text{mix}(b, a, h) - k \cdot h \cdot (1.0 - h)$$

Where $\text{mix}(a, b, t) = a(1 - t) + b \cdot t$ [S1, S2, S7, unofficial].

**2. Exponential Blend (LogSumExp)**
This method uses the soft-min approximation via exponentials [S4, unofficial].

$$d_{\text{out}} = -\frac{1}{k} \ln \left( e^{-k \cdot a} + e^{-k \cdot b} \right)$$

### Parameters
*   **$k$ (Smoothing Radius):** Controls the width of the transition region between the two shapes [S1, S2, unofficial]. Larger values create a smoother, wider blend between the shapes [S1, S2, S6, S7, unofficial]. Typical values range from $0.33$ to $1.0$ depending on scene scale [S6, S7, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def smooth_union_polynomial(a: float, b: float, k: float) -> float:
    """Computes the smooth union of two SDF values using polynomial interpolation."""
    if k <= 0:
        return min(a, b)
    h = 0.5 + 0.5 * (b - a) / k
    h = max(0.0, min(1.0, h))
    mix_val = b * (1.0 - h) + a * h
    blend_term = k * h * (1.0 - h)
    return mix_val - blend_term

def smooth_union_exponential(a: float, b: float, k: float) -> float:
    """Computes the smooth union using the LogSumExp approximation."""
    if k <= 0:
        return min(a, b)
    val_a = -k * a
    val_b = -k * b
    max_val = max(val_a, val_b)
    log_sum = max_val + math.log(math.exp(val_a - max_val) + math.exp(val_b - max_val))
    return -log_sum / k

def smooth_union(a: float, b: float, k: float, method: str = "polynomial") -> float:
    """Wrapper for smooth union operations."""
    if method == "exponential":
        return smooth_union_exponential(a, b, k)
    return smooth_union_polynomial(a, b, k)
```

### Edge Cases and Pitfalls
*   **Precision:** In the exponential method, direct computation of $e^{-k \cdot a}$ can overflow or underflow for large $a$ or large $k$ [S4, unofficial]. The logsumexp trick (subtracting the max value before exponentiating) is required for numerical stability [S4, unofficial].
*   **Approximation:** The operator produces a smooth approximation rather than an exact distance field in the transition region [S1, S2, unofficial].

## Subtopics

### [Polynomial Smooth Minimum](1827-polynomial-smooth-minimum.md)
Uses cubic polynomial interpolation for blending. Chosen when a specific, controllable curvature is desired without the computational cost of exponentials.

### [Smooth Intersection](1839-smooth-intersection.md)
The dual operator for smooth boolean intersection, often implemented as `-smin(-a, -b, k)`.

### [Multi-Material Distance Fields](1943-multi-material-distance-fields.md)
Extends smooth union to blend not just distances but also material properties across the transition region.

### [Linear SDF Blend](3004-linear-sdf-blend.md)
A simpler, linear interpolation between distances, which does not preserve the distance field property as well as smooth union but is cheaper.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per primitive pair [S4, S6, unofficial]. The polynomial method involves basic arithmetic operations (add, multiply, clamp) [S4, S6, unofficial]. The exponential method involves `exp` and `log`, which are more expensive but still constant time [S4, S6, unofficial].
*   **Memory Complexity:** $O(1)$ for the operator itself [S4, S6, unofficial].
*   **Strengths:** Eliminates sharp edges, allowing for organic merging of shapes (e.g., metaballs, pipes, joints) [S1, S2, S5, S6, unofficial]. Visually smooth results with minimal computational overhead compared to mesh-based smoothing [S1, S2, S5, S6, unofficial].
*   **Limitations:** Does not produce a true distance field in the transition region (it is an approximation) [S1, S2, unofficial].
*   **When to Use:** When creating organic shapes, joints, or rounded edges in ray marching scenes [S1, S2, S5, S6, unofficial].
*   **When Not to Use:** When exact geometric precision is required at the intersection [S1, S2, unofficial].

## Background
The concept of smooth minimum operators in SDFs emerged from the need to create organic, continuous shapes in ray marching without relying on polygonal meshes [S5, unofficial]. The polynomial smooth minimum (`smin`) was popularized by Inigo Quilez in his online tutorials on ray marching and SDFs [S5, S6, unofficial]. The exponential variant (LogSumExp) is derived from the mathematical property that the soft-min function is a differentiable approximation of the min function, widely used in machine learning and optimization [S4, unofficial]. These operators are now standard primitives in GLSL-based ray marching engines and differentiable rendering frameworks [S1, S2, S4, S7, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S3] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
[S4] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S5] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S6] An Introduction to Raymarching - Maxime Garcia, https://typhomnt.github.io/teaching/ray_tracing/raymarching_intro/, fetched 2026-10-27 (unofficial)
[S7] cahekp/Raymarching, https://github.com/cahekp/Raymarching, fetched 2026-10-27 (unofficial)
