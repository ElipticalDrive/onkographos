# Quilez Ambient Occlusion

Also called: Ambient Occlusion Approximation, Cheap Ambient Occlusion, Constant step ray marching AO, Quilez approach

## Overview
Quilez Ambient Occlusion is a fast approximation for ambient occlusion in Signed Distance Field (SDF) rendering that estimates surface visibility by ray marching along the surface normal [S1, unofficial][S3, unofficial]. Unlike Evans Ambient Occlusion, which measures the difference between the sampled distance and the distance field value at a single point, Quilez's method accumulates occlusion contributions from multiple samples along the normal, weighting closer geometry more heavily [S3, unofficial]. It differs from standard ray marching by using no explicit intersection checks and marching along the normal instead of a hemisphere [S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Surface point $\mathbf{x}$, surface normal $\mathbf{n}$, Signed Distance Field function $f(\mathbf{p})$, maximum step count $k_{max}$, step size $\Delta t$, and scale factor $s$ [S3, unofficial][S4, unofficial].
*   **Output:** Ambient occlusion value $A_O \in [0, 1]$, where 0 is full occlusion and 1 is no occlusion [S3, unofficial].

**Data Structures:**
No complex data structures are required. The method relies on scalar accumulators and the SDF query function [S3, unofficial][S4, unofficial].

**Steps:**
1.  Initialize $k = 1$, $d = 0$, and $occlusion = 0$ [S3, unofficial][S4, unofficial].
2.  While $k < k_{max}$:
    a. Calculate ray position: $r = \mathbf{x} + k \Delta t \mathbf{n}$ [S3, unofficial].
    b. Sample SDF: $d = f(r)$ [S3, unofficial][S4, unofficial].
    c. Accumulate occlusion: $occlusion = occlusion + \frac{1}{2^k} (k \Delta t - d)$ [S3, unofficial][S4, unofficial].
    d. Increment $k = k + 1$ [S3, unofficial][S4, unofficial].
3.  Return $1 - \text{clamp}(s \cdot occlusion, 0, 1)$ [S3, unofficial][S4, unofficial].

**Formulas:**
The ray position at sample $k$ is defined as [S3, unofficial]:
$$ r_n(k) = \mathbf{x} + k \Delta t \mathbf{n} $$
where $k \in \{1, 2, \dots, k_{max}\}$ [S3, unofficial]. Note that $k=0$ is omitted because measuring the distance to the surface point itself yields no useful computation for occlusion [S3, unofficial][S4, unofficial].

In the absence of occlusion, the sum of distance field values equals the sum of ideal distances: $\sum_{k=1}^{k_{max}} f(r_n(k)) = \sum_{k=1}^{k_{max}} \Delta t \cdot k$ [S3, unofficial]. With occlusion, the sum of distance field values is strictly less than the sum of ideal distances: $\sum_{k=1}^{k_{max}} f(r_n(k)) < \sum_{k=1}^{k_{max}} \Delta t \cdot k$ [S3, unofficial].

The ambient occlusion value is calculated as [S3, unofficial]:
$$ A_O = 1 - s \sum_{k=1}^{k_{max}} \frac{1}{2^k} (k \Delta t - f(r_n(k))) $$
The final ambient term applied to shading is $ambient = I_a k_a(\mathbf{x}) A_O$ [S3, unofficial]. The result is clamped to $[0, 1]$ [S3, unofficial][S4, unofficial].

**Parameters:**
*   $k_{max}$: Maximum number of samples. Typically 5 [S2, unofficial][S3, unofficial][S4, unofficial].
*   $\Delta t$: Step size for ray marching. Typical value: 0.01 [S1, unofficial][S4, unofficial].
*   $s$: Scale variable determining the darkening effect. Not given in the sources. [S3, unofficial][S4, unofficial]
*   $lim$: Maximum distance to march. Value: 0.1 [S1, unofficial]. (Used in recursive variants [S1, unofficial].)

**Edge Cases and Pitfalls:**
*   **Banding effect:** The occlusion does not change uniformly in color, resulting in visible "rings" of different shades of gray, attributed to taking steps of finite length along the ray [S3, unofficial].
*   **Boundary handling:** The result is clamped to $[0, 1]$ to prevent out-of-range values [S3, unofficial][S4, unofficial].
*   **Degenerate input:** If the SDF returns negative values (inside geometry), the term $(k \Delta t - d)$ increases, correctly increasing occlusion, but extreme values may require clamping or normalization in practice [S3, unofficial][S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quilez_ao(x, n, sdf, k_max, delta_t, s):
    """
    Calculate Quilez Ambient Occlusion.
    
    Args:
        x (vec3): Surface point.
        n (vec3): Surface normal.
        sdf (function): Signed Distance Field function.
        k_max (int): Maximum number of samples (typically 5).
        delta_t (float): Step size (typically 0.01).
        s (float): Scale factor for darkening.
        
    Returns:
        float: Ambient occlusion value in [0, 1].
    """
    occlusion = 0.0
    k = 1
    
    while k < k_max:
        # Ray position at sample k
        r = x + (k * delta_t) * n
        
        # Sample SDF
        d = sdf(r)
        
        # Accumulate weighted occlusion
        # Weight is 1/2^k, term is (ideal_distance - actual_distance)
        weight = 1.0 / (2 ** k)
        occlusion += weight * (k * delta_t - d)
        
        k += 1
        
    # Clamp and scale result
    ao = 1.0 - s * occlusion
    return max(0.0, min(1.0, ao))
```

**Alternative Implementation (Fixed Steps):**
Some implementations use fixed distances rather than a loop with $k$ [S2, unofficial]:
1.  Initialize $occ = 0.0$, $scale = 1.0$.
2.  Loop $i$ from 0 to 4:
    a. $dist = 0.02 + 0.12 \times i$.
    b. $h = sdf(\mathbf{x} + \mathbf{n} \times dist)$.
    c. $occ += (dist - h) \times scale$.
    d. $scale *= 0.6$.
3.  Return $\text{clamp}(1.0 - occ, 0.0, 1.0)$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(k_{max})$, where $k_{max}$ is the number of samples. Typically constant (~5 steps) [S3, unofficial][S4, unofficial].
*   **Memory Complexity:** $O(1)$, requiring only a few scalar variables [S3, unofficial].
*   **Performance:** Very fast due to low sample count and no intersection checks [S4, unofficial].
*   **Strengths:** Simple to implement, low computational cost, suitable for real-time SDF rendering [S3, unofficial][S4, unofficial].
*   **Limitations:** Can produce banding effects (rings of different shades) due to finite step sizes [S3, unofficial].
*   **When to use:** Real-time rendering with SDFs where performance is critical and high-quality hemisphere sampling is too expensive [S4, unofficial].
*   **When not to use:** Not given in the sources.

## Background
Quilez Ambient Occlusion is a modification of Evans' approach, using ray marching along the surface normal to estimate ambient occlusion [S3, unofficial]. It was presented by Inigo Quilez [S3, unofficial]. The method is based on the concept of constant step ray marching [S3, unofficial][S4, unofficial]. It is used in sphere tracing for implicit surfaces to approximate ambient occlusion [S4, unofficial].

## Sources
[S1] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S2] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S3] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S4] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
