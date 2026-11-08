# Adaptive Preintegration Termination Criteria

Also called: Preintegration-based Adaptive Termination

## Overview
Adaptive Preintegration Termination Criteria is a method for determining when to stop subdividing a ray segment during volume rendering, specifically within the context of preintegration. It solves the problem of balancing rendering accuracy against computational cost by recursively splitting intervals only when the estimated error exceeds a threshold. It sets itself apart from standard adaptive sampling [1385-adaptive-sampling.md] by using the difference between two precomputed integral approximations ($C_1$ and $C_2$) as the error metric, rather than just local density or gradient magnitude.

## How it works
The method operates on a recursive subdivision scheme. It computes two approximations of the volume integral over an interval $[d_i, d_{i+1}]$: $C_1$ (using a coarse step) and $C_2$ (using a finer step or midpoint). The decision to subdivide further is based on the relative difference between these values, adapted by the accumulated opacity.

**Inputs and Outputs**
*   **Inputs:** Preintegration tables (opacity, shading), ray parameters ($d_i, d_{i+1}$), accumulated opacity $\alpha_i$ at the start of the interval, scalar field $f(\mathbf{x})$, and transfer function $T$.
*   **Outputs:** Final color contribution for the interval, and a boolean indicating if further subdivision is required.

**Data Structures**
*   **Preintegration Tables:** Packed tables storing opacity and shading values as a function of step size and density [S1, unofficial].
*   **Interval State:** Start distance $d_i$, end distance $d_{i+1}$, accumulated opacity $\alpha_i$.

**Algorithm Steps**
1.  **Compute Approximations:** Calculate $C_1$ and $C_2$ for the current interval $[d_i, d_{i+1}]$ using the preintegration tables. $C_1$ typically represents the integral over the full interval, while $C_2$ represents the sum of integrals over two sub-intervals (e.g., $[d_i, d_{mid}]$ and $[d_{mid}, d_{i+1}]$).
2.  **Calculate Relative Error:** Compute the relative error between the two approximations:
    $$ E_{rel} = \frac{|C_1 - C_2|}{|C_2|} $$
    where $|C| = \sqrt{C.r^2 + C.g^2 + C.b^2}$ is the Euclidean norm of the color vector [S1, unofficial].
3.  **Adapt for Opacity:** Multiply the relative error by the remaining opacity $(1 - \alpha_i)$ to weight the error by the contribution to the final pixel:
    $$ E_{adapted} = (1 - \alpha_i) E_{rel} $$
4.  **Check Termination:** If $E_{adapted} < \varepsilon$ (a small threshold, e.g., $10^{-3}$), the interval is accepted, and the color $C_1$ (or $C_2$) is returned. Otherwise, the interval is subdivided.
5.  **Recursive Subdivision:** Split the interval at the midpoint $d_{mid} = (d_i + d_{i+1})/2$. Crucially, the left sub-interval $[d_i, d_{mid}]$ must be processed first to determine its accumulated opacity $\alpha_{mid}$. This $\alpha_{mid}$ is then used as the starting opacity for processing the right sub-interval $[d_{mid}, d_{i+1}]$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_preintegration_interval(d_start, d_end, alpha_in, preint_tables, epsilon):
    """
    Recursively subdivides an interval based on preintegration error.
    """
    # 1. Compute coarse approximation C1 over [d_start, d_end]
    C1 = compute_preint_color(d_start, d_end, preint_tables)
    
    # 2. Compute fine approximation C2 by splitting at midpoint
    d_mid = (d_start + d_end) / 2.0
    C2_left = compute_preint_color(d_start, d_mid, preint_tables)
    C2_right = compute_preint_color(d_mid, d_end, preint_tables)
    C2 = C2_left + C2_right
    
    # 3. Calculate relative error
    norm_C2 = sqrt(C2.r**2 + C2.g**2 + C2.b**2)
    if norm_C2 == 0:
        return C1, alpha_in # No contribution
    
    diff = sqrt((C1.r - C2.r)**2 + (C1.g - C2.g)**2 + (C1.b - C2.b)**2)
    rel_error = diff / norm_C2
    
    # 4. Adapt error by remaining opacity
    remaining_opacity = 1.0 - alpha_in
    adapted_error = remaining_opacity * rel_error
    
    # 5. Check termination
    if adapted_error < epsilon:
        return C1, alpha_in + compute_opacity(d_start, d_end, preint_tables)
    
    # 6. Recursive subdivision
    # Process left interval first to get alpha_mid
    C_left, alpha_mid = adaptive_preintegration_interval(d_start, d_mid, alpha_in, preint_tables, epsilon)
    
    # Process right interval using alpha_mid
    C_right, alpha_out = adaptive_preintegration_interval(d_mid, d_end, alpha_mid, preint_tables, epsilon)
    
    return C_left + C_right, alpha_out

def compute_preint_color(d_start, d_end, tables):
    """
    Looks up color in preintegration table based on interval length and density.
    Implementation detail: Not given in sources.
    """
    pass
```

**Parameters and Typical Values**
*   **$\varepsilon$:** Termination threshold. Not given in the sources.
*   **$l_{max}, l_{min}$:** Bounds on interval length. Not given in the sources.

**Edge Cases and Pitfalls**
*   **Non-monotonic Functions:** Standard subdivision may miss isosurfaces near extrema. This is handled by an additional check on derivatives (see Subtopics).
*   **Gradient Accuracy:** The extrema check relies on accurate gradient computation. Inaccurate gradients lead to incorrect subdivision decisions [S1, unofficial].
*   **Order of Operations:** The left sub-interval must be evaluated before the right to correctly propagate accumulated opacity [S1, unofficial].

## Subtopics

### Extrema-Based Subdivision
A variant that forces subdivision at scalar function extrema to ensure isosurfaces are not missed in non-monotonic regions. An interval $[d_i, d_{i+1}]$ is subdivided if the directional derivatives at the endpoints have opposite signs: $D_{\mathbf{v}}(f(\mathbf{x}(d_i))) D_{\mathbf{v}}(f(\mathbf{x}(d_{i+1}))) < 0$, where $D_{\mathbf{v}}(f(\mathbf{x})) = \nabla f(\mathbf{x}) \cdot \mathbf{v}$ [S1, unofficial]. This is efficient in MLS frameworks as gradients are already computed for shading. Accuracy depends heavily on gradient precision [S1, unofficial].

## Operating characteristics
*   **Complexity:** Adaptive; complexity depends on the local complexity of the scalar field and transfer function.
*   **Strengths:** Automatically optimal for constant transfer functions and linear scalar fields, as $C_1$ and $C_2$ will be equal within numerical errors [S1, unofficial].
*   **Limitations:** Requires precomputed tables and accurate gradient information for the extrema check.

## Background
This method is described in the context of Volume MLS Ray Casting [S1, unofficial]. It utilizes preintegration tables to accelerate the compositing step while maintaining adaptivity.

## Sources
[S1] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
