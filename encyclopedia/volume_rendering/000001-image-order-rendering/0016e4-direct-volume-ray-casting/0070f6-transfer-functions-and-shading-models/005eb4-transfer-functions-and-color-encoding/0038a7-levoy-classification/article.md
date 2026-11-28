# Levoy Classification

Also called: Levoy Opacity Function, Saliency Transfer Function

## Overview
Levoy Classification is a transfer function method that maps raw voxel intensities to opacity based on the difference between the voxel value and a selected threshold, modulated by the local gradient to maintain a constant isosurface thickness [S1, S2, unofficial]. Unlike simple binary thresholding or fixed-width iso-ranges, this method ensures that the visual thickness of a feature remains constant regardless of the steepness of the intensity gradient, making it particularly suitable for visualizing layered structures in chemistry and medical data [S1, S2, unofficial]. It sets itself apart from siblings like [Classified Volume Rendering](3590-classified-volume-rendering.md) by explicitly incorporating local gradient information into the opacity calculation to preserve layer information that would otherwise be lost [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Voxel intensity $f(\mathbf{x})$ at position $\mathbf{x}$, a target threshold value $f_v$, a desired transition thickness $R$, and the local gradient magnitude $\text{grad} = \|\nabla f(\mathbf{x})\|$ [S1, S2, unofficial].
*   **Output:** Opacity $\alpha \in [0, 1]$ [S1, S2, unofficial].

**Data Structures:**
The method requires access to the scalar volume data $f$ and the ability to compute or access the local gradient magnitude at each sample point [S1, S2, unofficial]. No additional lookup tables are required; the function is evaluated analytically [G].

**Parameters:**
*   $f_v$: The threshold value corresponding to the isosurface of interest [S1, S2, unofficial].
*   $R$: The desired constant thickness of the transition region [S1, S2, unofficial].
*   $\alpha_{\max}$: The maximum opacity assigned to the threshold value (typically 1.0) [S1, S2, unofficial].

**Steps:**
1.  Compute the local gradient magnitude $\text{grad} = \|\nabla f(\mathbf{x})\|$ at the sample point [S1, S2, unofficial].
2.  Determine the lower bound of the transition region $F_{\text{min}} = f_v - \text{grad} \cdot R$ [S1, S2, unofficial].
3.  Calculate the opacity $\alpha$ using the linear interpolation formula based on the value difference from the threshold and the gradient-scaled thickness [S1, S2, unofficial]:
    $$ \alpha = \alpha_{\max} \left( 1 - \frac{f_v - f(\mathbf{x})}{\text{grad} \cdot R} \right) $$
4.  Clamp the result to the range $[0, \alpha_{\max}]$ [G]. Specifically, if $f(\mathbf{x}) = f_v$, $\alpha = \alpha_{\max}$ [S1, S2, unofficial]. If $f(\mathbf{x}) \le f_v - \text{grad} \cdot R$, $\alpha = 0$ [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def levoy_classification(f_x, f_v, grad, R, alpha_max):
    """Calculate opacity for a single sample point using Levoy Classification."""
    # Guard against zero gradient to avoid division by zero
    if grad == 0:
        # [G] Fallback to a standard linear ramp based on R alone
        return max(0.0, alpha_max * (1.0 - abs(f_x - f_v) / R))
    
    denom = grad * R
    # Formula from [S1, unofficial]: alpha = a - a * (fv - f(x)) / (grad * R)
    alpha = alpha_max * (1.0 - (f_v - f_x) / denom)
    
    # Clamp to [0, alpha_max]
    return max(0.0, min(alpha, alpha_max))

def raycast_levoy(volume, bounds, origin, direction, step, f_v, R, alpha_max):
    """Full raycasting loop using Levoy Classification for opacity."""
    # [G] Compute ray-box intersection to get t_min and t_max
    t_min, t_max = 0.0, float('inf')
    # Simple AABB intersection logic for bounds (min_x, max_x, min_y, max_y, min_z, max_z)
    # ... (standard implementation omitted for brevity)
    
    # Initialize accumulated color and opacity
    acc_color = [0.0, 0.0, 0.0]
    acc_alpha = 0.0
    t = t_min
    
    while t < t_max and acc_alpha < 0.95: # [G] Early termination threshold
        pos = origin + direction * t
        f_x = volume.sample(pos)
        grad = volume.gradient_magnitude(pos)
        
        alpha = levoy_classification(f_x, f_v, grad, R, alpha_max)
        c = volume.color(f_x)
        
        # Front-to-back compositing
        alpha_contrib = alpha * (1.0 - acc_alpha)
        acc_color = [c[i] * alpha_contrib + acc_color[i] * (1.0 - alpha_contrib) for i in range(3)]
        acc_alpha += alpha_contrib
        
        t += step
        
    return acc_color, acc_alpha
```

**Edge Cases and Pitfalls:**
*   **Zero Gradient:** If $\text{grad} = 0$, the denominator becomes zero; a fallback linear ramp based on $R$ is used [G].
*   **Boundary Conditions:** The formula assumes a linear fall-off within the transition region $[f_v - \text{grad} \cdot R, f_v]$ [S1, S2, unofficial].
*   **Thickness Interpretation:** $R$ represents the thickness in value space scaled by the gradient, not necessarily physical distance, unless the gradient is normalized by sampling rate [G].

## Subtopics

### Chemistry Data Application
Used to maintain constant layer thickness with linear fall-off of opacity, preserving information about layers that simple iso-value extraction would lose [S1, S2, unofficial].

### Medical Data Application
Assumes at most two tissues meet at a voxel, using linear transitions to reflect the linear combination of tissues [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample point, assuming the gradient is pre-computed or efficiently available [S1, S2, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per sample, as no large lookup tables are required [S1, S2, unofficial].
*   **Strengths:** Preserves the visual thickness of features regardless of gradient steepness; effectively visualizes layered structures in chemistry and medical data; simple to implement analytically [S1, S2, unofficial].
*   **Limitations:** Requires accurate gradient computation; sensitive to noise in the gradient; assumes a linear relationship between value difference and opacity within the transition region [S1, S2, unofficial].
*   **When to use:** When visualizing thin layers or interfaces where constant visual thickness is critical, such as in chemical concentration gradients or tissue boundaries [S1, S2, unofficial].

## Background
Levoy Classification was introduced by Marc Levoy in 1990 as part of early direct volume rendering techniques [S1, S2, unofficial]. It was developed to address the limitations of simple iso-surface extraction, which loses information about layers and thickness variations [S1, unofficial]. The method is particularly noted for its application in chemistry and medical data visualization, where maintaining constant isosurface thickness is crucial for accurate interpretation [S1, S2, unofficial]. Levoy also proposed improvements such as front-to-back rendering with early ray termination and the use of hierarchical oct-tree data structures to skip empty cells efficiently [S2, unofficial].

## Sources
[S1] PPT - Direct Volume Rendering (DVR): Ray-casting PowerPoint Presentation - ID:465503, https://slideserve.com/jariath/direct-volume-rendering-dvr-ray-casting, fetched 2026-10-24 (unofficial)
[S2] Direct volume rendering ray casting, https://slidetodoc.com/direct-volume-rendering-dvr-raycasting-jian-huang-this/, fetched 2026-10-29 (unofficial)
