# Gradient-Based Boundary Classification

Also called: Cell-Diagonal Data Shift, Reflecting Material Classification, Gradient-based surface classification, Reflecting Surface Classification

## Overview
Gradient-Based Boundary Classification is a method for determining whether a voxel lies on the boundary of a material segment within a volume, classifying it as either "reflecting" (surface) or "ambient" (interior/exterior) [S1, S2, unofficial]. It solves the problem of surface detection by interpreting the gradient magnitude in the context of the voxel's data value and the material's classification range, rather than relying on gradient magnitude in isolation [S1, S2, unofficial]. This approach departs from earlier methods [Lev90a, LL94] by using a linear approximation of the field function to estimate the distance to the material boundary, allowing for a probabilistic classification that handles noise and nonlinearity [S1, S2, unofficial].

## How it works
**Inputs:**
*   Voxel data value $d$ at location $\mathbf{g}_0$ [S1, S2, unofficial].
*   Gradient vector $\nabla f = (\nabla_x, \nabla_y, \nabla_z)$ at $\mathbf{g}_0$ [S1, S2, unofficial].
*   Inter-voxel spacing $\Delta$ (assumed uniform in all dimensions) [S1, S2, unofficial].
*   Material classification range $[b, a]$, where $b$ is the lower limit and $a$ is the upper limit for the material [S1, S2, unofficial].
*   Scaling factor $k$ (client-tunable parameter) [S1, S2, unofficial].

**Outputs:**
*   Boundary probability $P \in [0, 1]$ [S1, S2, unofficial].
*   Classification: Reflecting if $P > 0$, Ambient if $P = 0$ (or combined via $P$ for shading) [S1, S2, unofficial].

**Data Structures:**
*   The method operates on per-voxel data [S1, S2, unofficial].
*   No global data structures are required for the classification logic itself, though it is typically used in conjunction with a lookup table for shading (see [Reflected Light Component Lookup](3521-reflected-light-component-lookup.md)) [G].

**Steps:**

1.  **Linear Field Approximation:**
    Model the field function $f(\mathbf{q})$ in the neighborhood of the voxel $\mathbf{g}_0$ as:
    $$ f(\mathbf{q}) = d + \nabla f \cdot (\mathbf{q} - \mathbf{g}_0) $$
    where $\mathbf{q}$ is a point in the neighborhood [S1, S2, unofficial].

2.  **Calculate Cell-Diagonal Data Shift ($s$):**
    Compute the maximum possible data shift along the diagonal of the voxel cell based on the gradient components and inter-voxel spacing:
    $$ s = \Delta |\nabla_x| + \Delta |\nabla_y| + \Delta |\nabla_z| $$
    This quantity $s$ represents the maximum change in data value across the diagonal [S1, S2, unofficial].

3.  **Test for Lower Boundary:**
    Determine if the voxel is near the lower boundary $b$ of the material range. A neighboring voxel has data below $b$ if and only if:
    $$ (d - b) < s $$
    If this condition is not met, the voxel is considered interior to the material (ambient) with respect to the lower bound [S1, S2, unofficial].

4.  **Test for Upper Boundary:**
    Determine if the voxel is near the upper boundary $a$ of the material range. A similar test applies:
    $$ (a - d) < s $$
    If this condition is not met, the voxel is considered interior to the material (ambient) with respect to the upper bound [S1, S2, unofficial].

5.  **Calculate Boundary Probability ($P$):**
    Assign a probability of being on the boundary to handle noise and nonlinearity, rather than a binary decision [S1, S2, unofficial].
    
    *   **Lower Boundary Probability ($P_{lower}$):**
        If $s < 0.5(d - b)$, then $P_{lower} = 0$ [S1, S2, unofficial].
        If $s > 1.5(d - b)$, then $P_{lower} = 1$ [S1, S2, unofficial].
        Otherwise, $P_{lower}$ increases linearly from 0 to 1 as $s$ goes from $0.5(d - b)$ to $1.5(d - b)$ [S1, S2, unofficial].
        
        $$ P_{lower} = \text{clamp}\left( \frac{s - 0.5(d - b)}{d - b}, 0, 1 \right) $$
        This formula assumes $d > b$ [G].
        
    *   **Upper Boundary Probability ($P_{upper}$):**
        Similarly, if $s < 0.5(a - d)$, then $P_{upper} = 0$ [S1, S2, unofficial].
        If $s > 1.5(a - d)$, then $P_{upper} = 1$ [S1, S2, unofficial].
        Otherwise, $P_{upper}$ increases linearly from 0 to 1 as $s$ goes from $0.5(a - d)$ to $1.5(a - d)$ [S1, S2, unofficial].
        
        $$ P_{upper} = \text{clamp}\left( \frac{s - 0.5(a - d)}{a - d}, 0, 1 \right) $$
        This formula assumes $d < a$ [G].

    *   **Combined Probability:**
        The final probability $P$ is derived from $P_{lower}$ and $P_{upper}$ [S1, S2, unofficial]. The sources state that the final shading combines ambient and reflecting colors according to the probability [S1, S2, unofficial]. Typically, a voxel is considered on *any* boundary if it is near the lower or upper boundary [G]. A common implementation uses $P = \max(P_{lower}, P_{upper})$ [G]. The scaling factor $k$ is applied to the probability function to fine-tune the classification [S1, S2, unofficial]. For implementation, $k$ scales the effective shift $s$ before probability calculation [G].

**Parameters:**
*   $\Delta$: Inter-voxel spacing. Not given in the sources.
*   $b, a$: Material range limits. Determined by the transfer function or user input [G].
*   $k$: Scaling factor. Client-tunable. Causes more or fewer voxels to be judged on a surface with nonzero probability [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **High Data Value, High Gradient:** A voxel with a high data value (e.g., 130 in a bone example where $b=110$) may have a high gradient but be far from the boundary [S2, unofficial]. In this case, $(d-b)$ is large, and if $s < 0.5(d-b)$, the probability is 0, correctly classifying it as ambient [S2, unofficial].
*   **Noise:** The probabilistic approach allows for noise in the gradient and nonlinearity of the field function [S1, S2, unofficial].
*   **Non-Uniform Spacing:** The formula assumes uniform inter-voxel spacing $\Delta$ [S1, S2, unofficial]. If spacing is non-uniform, $\Delta$ should be replaced by the appropriate spacing in each dimension [G].
*   **Division by Zero:** If $d = b$ or $d = a$, the denominator in the probability calculation becomes zero [G]. In these cases, the voxel is exactly on the boundary, so $P$ should be 1 [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_based_boundary_classification(d, grad_x, grad_y, grad_z, delta, b, a, k=1.0):
    """
    Classifies a voxel as reflecting or ambient based on gradient-based boundary detection.
    
    Args:
        d (float): Voxel data value.
        grad_x, grad_y, grad_z (float): Gradient components.
        delta (float): Inter-voxel spacing.
        b (float): Lower limit of material range.
        a (float): Upper limit of material range.
        k (float): Scaling factor for probability tuning.
        
    Returns:
        float: Probability of being on the boundary (0.0 to 1.0).
    """
    # Step 2: Calculate cell-diagonal data shift s
    s = delta * (abs(grad_x) + abs(grad_y) + abs(grad_z))
    
    # Apply scaling factor k to the effective shift for probability tuning [S1, S2, unofficial]
    s_eff = s * k
    
    # Guard: Voxel must be strictly inside the material range to be near a boundary [G]
    if d <= b or d >= a:
        return 0.0
        
    # Step 3 & 5: Lower Boundary Probability
    diff_lower = d - b
    t_start_lower = 0.5 * diff_lower
    t_end_lower = 1.5 * diff_lower
    
    if s_eff < t_start_lower:
        P_lower = 0.0
    elif s_eff > t_end_lower:
        P_lower = 1.0
    else:
        P_lower = min(max((s_eff - t_start_lower) / (t_end_lower - t_start_lower), 0.0), 1.0)
        
    # Step 4 & 5: Upper Boundary Probability
    diff_upper = a - d
    t_start_upper = 0.5 * diff_upper
    t_end_upper = 1.5 * diff_upper
    
    if s_eff < t_start_upper:
        P_upper = 0.0
    elif s_eff > t_end_upper:
        P_upper = 1.0
    else:
        P_upper = min(max((s_eff - t_start_upper) / (t_end_upper - t_start_upper), 0.0), 1.0)
        
    # Combine probabilities: voxel is on boundary if near either [G]
    P = max(P_lower, P_upper)
    return P
```

## Subtopics

### [Reflected Light Component Lookup](3521-reflected-light-component-lookup.md)
Pre-calculates surface responses to directional lighting for combinations of material types and quantized gradient directions, storing results in a lookup table. It decouples expensive lighting calculations from the per-voxel rendering loop, allowing for realistic surface appearance in volume rendering. Choose this when rendering multiple-material volumes with specular highlights and directional lighting.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel for the classification step [G]. The overall rendering time depends on the volume rendering algorithm (e.g., ray casting) and the cost of the shading lookup (see [Reflected Light Component Lookup](3521-reflected-light-component-lookup.md)) [G].
*   **Memory Complexity:** $O(1)$ per voxel for the classification logic [G]. Additional memory is required for the lookup table in [Reflected Light Component Lookup](3521-reflected-light-component-lookup.md) [S2, unofficial].
*   **Measured Performance:** The method is described as "nearly interactive" when combined with the lookup table technique [S2, unofficial].
*   **Strengths:** Handles noise and nonlinearity better than binary thresholding [S1, S2, unofficial]. Interprets gradient in context of data value, avoiding false positives for high-gradient interior voxels [S2, unofficial].
*   **Limitations:** Requires accurate gradient estimation [G]. Performance depends on the complexity of the shading lookup [G].
*   **When to Use:** When realistic surface shading is required for volume data with distinct material boundaries [G].
*   **When Not to Use:** When only ambient shading is sufficient, or when gradient information is unreliable [G].

## Background
The method is part of the Voltx system [S2, unofficial]. It departs from earlier work [Lev90a, LL94] by interpreting gradient magnitude in the context of data value and classification parameters [S1, S2, unofficial]. It was developed to eliminate technical programming used by previously reported 3D texture-map methods [S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19970021689: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970021689, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19970022121: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970022121, fetched 2026-10-26 (unofficial)
