# Attribute Accumulation and Blending

Also called: Splat attribute blending, Voronoi cardinal attributes

## Overview
Attribute Accumulation and Blending is the fragment processing stage in hardware-accelerated surface splatting that reconstructs continuous and cardinal attributes from overlapping splats into a reconstruction buffer. It distinguishes itself from sibling passes by handling two distinct attribute types: continuous attributes are accumulated via weighted blending, while cardinal attributes are resolved via maximum-weight comparison, effectively assigning each screen-space pixel to the splat with the highest weight within an $\varepsilon$-band. This method operates in conjunction with a ternary depth test to manage z-pass, z-fail, and z-blend conditions.

## How it works
**Inputs and Outputs:**
*   **Input:** Source splat fragment with screen-space depth $z_s$, weight $w_f$, and attributes $\mathbf{a}_s$ (continuous or cardinal). Destination reconstruction buffer state with depth $z_d$, accumulated weight $w_d$, and accumulated attributes $\mathbf{a}_d$.
*   **Output:** Updated reconstruction buffer state $(w_d, \mathbf{a}_d, z_d)$.

**Data Structures:**
The reconstruction buffer stores per-pixel:
*   $z_d$: Depth value (float).
*   $w_d$: Accumulated weight (float).
*   $\mathbf{a}_d$: Accumulated attributes (vector of floats, size depends on attribute type).

**Parameters:**
*   $\varepsilon_{\Delta z}$: Scales depth extents. Typical value: $1.0$ [S1].
*   $\varepsilon_{bias}$: Accounts for numerical inaccuracies. Comparable to `glPolygonOffset`'s unit parameter [S1].
*   $\varepsilon$: The blending band size, computed as $\varepsilon := h_z \cdot \varepsilon_{\Delta z} + \varepsilon_{bias}$, where $h_z$ is the splat's depth range [S1].

**Steps:**
1.  **Ternary Depth Test:** Compare $z_d$ and $z_s$ using the $\varepsilon$ band [S1]:
    *   If $z_d < z_s - \varepsilon$: **z-fail**. Discard fragment.
    *   If $z_d > z_s + \varepsilon$: **z-pass**. Proceed to attribute update.
    *   Otherwise: **z-blend**. Proceed to attribute blending.

2.  **Attribute Update (z-pass):**
    *   Replace destination weight and attributes with source values [S1]:
        $$ (w_d, \mathbf{a}_d) := (w_f, \mathbf{a}_s) $$
    *   Note: The source notes indicate a distinction in the formula representation for continuous vs cardinal attributes in the table, but the text explicitly states "weight $w_d$ and attributes $\mathbf{a}_d$ ... are replaced by ... $(w_f, \mathbf{a}_s)$" for cardinal, and "$(w_f, w_f, \mathbf{a}_s)$" for continuous. However, the tuple structure $(w_d, \mathbf{a}_d)$ implies the destination only stores one weight. The table in [S1] shows:
        *   Continuous: $(w_d, \mathbf{a}_d) := (w_f, w_f, \mathbf{a}_s)$ — This appears to be a typo in the source table or implies a specific internal storage. Given the standard definition of $(w_d, \mathbf{a}_d)$, and the blending formula $(w_d + w_f, ...)$, the weight is a single scalar. The text says "replace destination weight $w_d$ ... with source weight $w_f$". We follow the explicit text instruction for replacement: $w_d \leftarrow w_f, \mathbf{a}_d \leftarrow \mathbf{a}_s$. The extra $w_f$ in the continuous table entry is likely an artifact of the table formatting or a specific internal buffer layout not fully detailed. We assume standard replacement: $w_d = w_f, \mathbf{a}_d = \mathbf{a}_s$.

3.  **Attribute Blending (z-blend):**
    *   **Continuous Attributes:** Blend weighted source attributes into the destination [S1]:
        $$ w_d := w_d + w_f $$
        $$ \mathbf{a}_d := \mathbf{a}_d + w_f \cdot \mathbf{a}_s $$
    *   **Cardinal Attributes:** Compare weights. If $w_d < w_f$, replace destination with source; otherwise keep destination [S1]:
        $$ \text{if } w_d < w_f \text{ then } (w_d, \mathbf{a}_d) := (w_f, \mathbf{a}_s) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def accumulate_attributes(z_d, w_d, a_d, z_s, w_f, a_s, is_continuous, epsilon):
    """
    Updates reconstruction buffer state based on ternary depth test.
    
    Args:
        z_d (float): Current depth in buffer.
        w_d (float): Current accumulated weight in buffer.
        a_d (vector): Current accumulated attributes in buffer.
        z_s (float): Depth of incoming splat fragment.
        w_f (float): Weight of incoming splat fragment.
        a_s (vector): Attributes of incoming splat fragment.
        is_continuous (bool): True if attributes are continuous, False for cardinal.
        epsilon (float): Blending band size.
        
    Returns:
        tuple: (new_z_d, new_w_d, new_a_d)
    """
    # Ternary Depth Test
    if z_d < z_s - epsilon:
        # z-fail: Discard fragment
        return (z_d, w_d, a_d)
    
    if z_d > z_s + epsilon:
        # z-pass: Replace destination
        # Source [S1] states replacement for both types, though table formatting varies.
        # Text explicitly says: replace w_d with w_f, a_d with a_s.
        return (z_s, w_f, a_s)
    
    # z-blend: Accumulate
    if is_continuous:
        # Continuous attributes are blended
        new_w_d = w_d + w_f
        new_a_d = a_d + w_f * a_s
        return (z_d, new_w_d, new_a_d)
    else:
        # Cardinal attributes: Maximum weight comparison
        if w_d < w_f:
            return (z_s, w_f, a_s)
        else:
            return (z_d, w_d, a_d)
```

## Subtopics

### Attribute Normalization and Surface Completion
This variant handles the post-accumulation phase where normalized attributes are fed to fragment shader units for deferred shading. After accumulation, continuous attributes are normalized by dividing accumulated attributes by accumulated weight: $a_f = a_d / w_d$ [S1]. Surface completion is detected using higher-level signals such as `glDrawElements()`, `glDrawArrays()`, or `glEnd()` after rendering individual splats [S1]. A key pitfall is that accumulated splats define a surface only after all affecting splats have been processed; completion cannot be reliably deduced from individual splats or accumulated values alone [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment per splat. The ternary test and attribute update are constant-time operations.
*   **Memory Complexity:** $O(N)$ for the reconstruction buffer, where $N$ is the number of screen pixels. The buffer stores depth, weight, and attributes per pixel.
*   **Performance:** The method avoids deep depth buffers, which were tested but found to have insufficient benefit to justify storage costs and complexity [S1].
*   **Strengths:** Efficiently handles overlapping splats via the $\varepsilon$-band blending. Supports both continuous and cardinal attributes.
*   **Limitations:** Requires careful tuning of $\varepsilon$ parameters to avoid artifacts from near-degenerate splats or orthogonality issues [S1]. The blending band size depends on the splat's depth range $h_z$.
*   **When to use:** Hardware-accelerated surface splatting where both continuous (e.g., color, normal) and cardinal (e.g., material ID) attributes need to be reconstructed.
*   **When not to use:** When exact depth ordering is required without blending (use z-pass only). When memory bandwidth is extremely limited and deep buffers are not an option but high overlap exists (the $\varepsilon$-band may cause excessive blending).

## Background
The method originates from hardware architectures for surface splatting, specifically described in [S1]. It builds on the EWA screen-space filter approximation by Botsch et al. [2005] for efficient ray-splat intersection and filtering. The ternary depth test was introduced to handle the accumulation of overlapping splats without the complexity of deep depth buffers used in earlier software implementations by Pfister et al. [2000] and Räsänen [2002].

## Sources
[S1] A hardware architecture for surface splatting, https://doi.org/10.1145/1275808.1276490, fetched 2026-10-28
