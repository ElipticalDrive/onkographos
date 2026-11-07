# SDF Smooth Union

Also called: Smooth Union, Smooth Minimum, S-Union, LogSumExp SDF Union, Smooth Union SDF Operator

## Overview
SDF Smooth Union is a constructive operator that blends two Signed Distance Function (SDF) values, $a$ and $b$, into a single smooth distance field $d_{out}$, replacing the sharp geometric intersection of a boolean union with a rounded transition region controlled by a radius parameter $k$ [S1, S2, S3, S4, S6, S8, unofficial]. It sets itself apart from its siblings by offering two primary mathematical implementations: a **polynomial** blend (often called `smin`) that uses cubic interpolation for a specific curvature profile, and an **exponential** blend (LogSumExp) that provides a different smoothness characteristic; unlike the standard `min` operator, it preserves visual continuity but requires careful handling of the Lipschitz condition to avoid ray marching artifacts [S1, S2, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs:** Two scalar distance values $a$ and $b$ (the distances from the query point to the surfaces of two primitives), and a scalar smoothing radius $k > 0$ [S1, S2, S3, S6, S7, unofficial].
*   **Output:** A single scalar distance $d_{out}$ representing the smooth union [S1, S2, S3, S6, S7, unofficial].

### Data Structures
No complex data structures are required for the core operator itself. It operates on scalar floats. However, in multi-material contexts, it may operate on structs containing distance and material data [S7, unofficial].

### Formulas
There are two dominant formulations found in the sources. They produce visually similar but mathematically distinct results.

**1. Polynomial Smooth Minimum (Cubic Blend)**
This method uses a cubic polynomial to blend the distances [S1, S2, S6, S7, unofficial].

$$
h = \text{clamp}\left(0.5 + 0.5 \frac{b - a}{k}, 0.0, 1.0\right)
$$

$$
d_{out} = \text{mix}(b, a, h) - k \cdot h \cdot (1.0 - h)
$$

Where $\text{mix}(b, a, h) = b(1 - h) + a \cdot h$ [S1, S2, S6, unofficial].

**2. Exponential Smooth Minimum (LogSumExp)**
This method uses the soft-min approximation via exponentials [S4, unofficial].

$$
d_{out} = -\frac{1}{k} \ln \left( e^{-k \cdot a} + e^{-k \cdot b} \right)
$$

Or equivalently:
$$
d_{out} = \frac{1}{-k} \text{logsumexp}_{i \in \{a,b\}}(-k \cdot S_i)
$$

### Parameters
*   **$k$ (Smoothing Radius):** Controls the width of the transition region. Larger values create a smoother, wider blend; smaller values create sharper transitions closer to the boolean `min` [S1, S2, S6, S7, unofficial].
    *   *Typical Values:* $0.33$ to $1.0$ depending on scene scale [S6, S7, unofficial].
    *   *Constraint:* $k$ must be significantly smaller than the scale of the objects to prevent geometric distortion [S1, S2, unofficial].

### Pseudocode
The following implementation combines the polynomial approach (most common in GLSL-based ray marching) and includes the edge case handling found in some sources [S3, S6, S7, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smooth_union_polynomial(a: float, b: float, k: float) -> float:
    """
    Computes the smooth union of two SDF values using cubic interpolation.
    
    Args:
        a: Distance to first primitive.
        b: Distance to second primitive.
        k: Smoothing radius. Must be > 0.
        
    Returns:
        Smoothed distance value.
    """
    if k <= 0:
        # Fallback to standard boolean min if k is invalid
        return min(a, b)

    # Calculate interpolation weight h
    # h is 0 when a << b, 1 when a >> b, 0.5 when a == b
    h = 0.5 + 0.5 * (b - a) / k
    
    # Clamp h to [0.0, 1.0] to ensure stability outside the blend region
    h = max(0.0, min(1.0, h))
    
    # Linear interpolation between b and a
    mix_val = b * (1.0 - h) + a * h
    
    # Subtract the cubic blend term
    # This term is 0 when h=0 or h=1, and max at h=0.5
    blend_term = k * h * (1.0 - h)
    
    return mix_val - blend_term

def smooth_union_exponential(a: float, b: float, k: float) -> float:
    """
    Computes the smooth union using the LogSumExp approximation.
    
    Args:
        a: Distance to first primitive.
        b: Distance to second primitive.
        k: Smoothing radius. Must be > 0.
        
    Returns:
        Smoothed distance value.
    """
    if k <= 0:
        return min(a, b)
        
    # Compute -k * a and -k * b
    val_a = -k * a
    val_b = -k * b
    
    # Find max for numerical stability (logsumexp trick)
    max_val = max(val_a, val_b)
    
    # Compute logsumexp
    # log(exp(val_a) + exp(val_b)) = max_val + log(exp(val_a - max_val) + exp(val_b - max_val))
    log_sum = max_val + math.log(math.exp(val_a - max_val) + math.exp(val_b - max_val))
    
    # Final result: -1/k * logsumexp
    return -log_sum / k

def smooth_union(a: float, b: float, k: float, method: str = "polynomial") -> float:
    """
    Wrapper for smooth union operations.
    
    Args:
        method: "polynomial" or "exponential".
    """
    if method == "exponential":
        return smooth_union_exponential(a, b, k)
    else:
        return smooth_union_polynomial(a, b, k)
```

### Edge Cases and Pitfalls
1.  **Lipschitz Continuity Loss:** The resulting SDF preserves the Lipschitz continuity ($|\nabla d| \le 1$) only if the gradients of the two input SDFs are orthogonal [S1, S2, unofficial]. If the gradients deviate by more than $30^\circ$ or if surfaces are parallel and close, continuity is lost, leading to ray marching overshooting or jittering [S1, S2, unofficial].
2.  **Parallel Surfaces:** The polynomial blend assumes the surfaces are approaching each other. If two large parallel planes are blended, the resulting geometry may be incorrect or distorted [S1, S2, unofficial].
3.  **Precision:** In the exponential method, direct computation of $e^{-k \cdot a}$ can overflow or underflow for large $a$ or large $k$. The "logsumexp trick" (subtracting the max value before exponentiating) is required for numerical stability [S4, unofficial].
4.  **Parameter Scaling:** If $k$ is too large relative to the object size, the shapes will merge too early, creating unnatural bulges [S1, S2, unofficial].

## Subtopics

### [Polynomial Smooth Minimum](1827-polynomial-smooth-minimum.md)
Uses cubic polynomial interpolation for blending. Chosen when a specific, controllable curvature is desired without the computational cost of exponentials.

### [Smooth Intersection](1839-smooth-intersection.md)
The dual operator for smooth boolean intersection, often implemented as `-smin(-a, -b, k)`.

### [Multi-Material Distance Fields](1943-multi-material-distance-fields.md)
Extends smooth union to blend not just distances but also material properties (color, roughness) across the transition region.

### [Linear SDF Blend](3004-linear-sdf-blend.md)
A simpler, linear interpolation between distances, which does not preserve the distance field property as well as smooth union but is cheaper.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per primitive pair. The polynomial method involves basic arithmetic operations (add, multiply, clamp). The exponential method involves `exp` and `log`, which are more expensive but still constant time [S4, S6, unofficial].
*   **Memory Complexity:** $O(1)$ for the operator itself. No additional storage is needed beyond the input distances [S1, S2, unofficial].
*   **Strengths:** Eliminates sharp edges, allowing for organic merging of shapes (e.g., metaballs, pipes, joints). Visually smooth results with minimal computational overhead compared to mesh-based smoothing [S1, S2, S5, S6, unofficial].
*   **Limitations:** Does not produce a true distance field in the transition region (it is an approximation). Can cause ray marching artifacts if the Lipschitz condition is violated (gradients not orthogonal) [S1, S2, unofficial].
*   **When to Use:** When creating organic shapes, joints, or rounded edges in ray marching scenes where performance is critical and exact distance precision in the blend zone is not required [S1, S2, S5, S6, unofficial].
*   **When Not to Use:** When exact geometric precision is required at the intersection, or when blending parallel surfaces where the Lipschitz condition is likely to fail [S1, S2, unofficial].

## Background
The concept of smooth minimum operators in SDFs emerged from the need to create organic, continuous shapes in ray marching without relying on polygonal meshes. The polynomial smooth minimum (`smin`) was popularized by Inigo Quilez in his online tutorials on ray marching and SDFs [S5, S6, unofficial]. The exponential variant (LogSumExp) is derived from the mathematical property that the soft-min function is a differentiable approximation of the min function, widely used in machine learning and optimization [S4, unofficial]. These operators are now standard primitives in GLSL-based ray marching engines and differentiable rendering frameworks [S1, S2, S4, S7, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S3] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
[S4] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S5] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S6] An Introduction to Raymarching - Maxime Garcia, https://typhomnt.github.io/teaching/ray_tracing/raymarching_intro/, fetched 2026-10-27 (unofficial)
[S7] cahekp/Raymarching, https://github.com/cahekp/Raymarching, fetched 2026-10-27 (unofficial)
