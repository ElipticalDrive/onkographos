# Single-Layer Ternary Depth Test

Also called: Single-layer depth test, Ternary Depth Test, Ternary Depth Test for Surface Reconstruction, Three-way z-test, Weyrich et al. depth test, Z-blend test

## Overview
The Single-Layer Ternary Depth Test is a rasterization technique for surface splatting that maintains exactly one surface layer per pixel, using a depth band to decide whether to blend, replace, or discard incoming fragments [S1, unofficial][S2]. It solves the problem of handling overlapping splats during surface reconstruction by replacing the standard binary pass/fail depth test with a ternary logic that accommodates depth uncertainty [S2]. It differs from depth-interval grouping by maintaining only a single layer per pixel, which simplifies the pipeline but causes visibility and occlusion artifacts in complex geometry [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A stream of splat fragments, each with a depth value $z_d$, color, and weight [S1, unofficial][S2].
*   **State:** Each pixel maintains a single stored depth value $z_s$ and a running accumulated sum (e.g., EWA sum) for the single surface layer [S1, unofficial].
*   **Output:** The final color and opacity for the pixel, representing the single closest surface layer [S1, unofficial].

**Data Structures**
*   **Pixel State:** A single depth value $z_s$ is stored in the depth buffer or a dedicated surface layer buffer [S1, unofficial]. No multi-layer stack is maintained [S1, unofficial].

**Parameters**
*   $\varepsilon_{\Delta z}$: A scaling factor for the depth extents of the splat. Typically set to $1.0$ [S2].
*   $\varepsilon_{bias}$: A bias term accounting for numerical inaccuracies, comparable to `glPolygonOffset`'s parameter unit [S2].
*   $h_z$: The depth extent (or half-height) of the splat [S2].

**Algorithm Steps**
1.  **Compute Epsilon Band:** For each incoming splat, compute the epsilon band size $\varepsilon$ based on the splat's depth range $h_z$:
    $$ \varepsilon := h_z \cdot \varepsilon_{\Delta z} + \varepsilon_{bias} $$
    [S2]
2.  **Compare Depths:** For each incoming fragment with depth $z_d$, compare it to the currently stored depth $z_s$ for that pixel [S1, unofficial].
3.  **Apply Ternary Test:** Determine the action based on the difference between the fragment depth and the stored depth [S2]:
    *   **Z-Blend:** If the fragment depth falls within the $\varepsilon$-band of the stored value, i.e., $|z_d - z_s| \le \varepsilon$, blend the fragment into the running EWA sum [S1, unofficial][S2].
    *   **Z-Fail (Discard):** If the fragment is significantly behind the stored layer ($z_d < z_s - \varepsilon$), discard the fragment [S2].
    *   **Z-Pass (Replace):** If the fragment is significantly in front of the stored layer ($z_d > z_s + \varepsilon$), replace the stored layer with this new fragment (updating $z_s$ and resetting the accumulation) [S1, unofficial][S2].
4.  **Normalization:** After processing all fragments for a pixel, the accumulated color and weight are normalized [S1, unofficial]. Because no coverage-based opacity is involved, the normalized output is always fully opaque wherever any surfel contributes, even at silhouette pixels where only a few surfels partially cover the pixel footprint and the accumulated weight is low [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_layer_ternary_depth_test(fragment, pixel_state, epsilon_delta_z=1.0, epsilon_bias=0.0):
    """
    fragment: object with attributes .depth (z_d), .color, .weight, .h_z
    pixel_state: object with attributes .stored_depth (z_s), .accumulated_color, .accumulated_weight
    epsilon_delta_z: scaling factor for depth extents (default 1.0) [S2]
    epsilon_bias: bias for numerical inaccuracies (default 0.0) [S2]
    Returns: Updated pixel_state
    """
    # 1. Compute epsilon band size
    epsilon = fragment.h_z * epsilon_delta_z + epsilon_bias
    
    z_d = fragment.depth
    z_s = pixel_state.stored_depth
    
    # 2. Ternary Depth Test
    if z_d < z_s - epsilon:
        # Z-Fail: Fragment is significantly behind the stored layer
        # Discard fragment, do not update pixel state
        return pixel_state
    
    elif z_d > z_s + epsilon:
        # Z-Pass: Fragment is significantly in front of the stored layer
        # Replace the stored layer
        pixel_state.stored_depth = z_d
        pixel_state.accumulated_color = fragment.color * fragment.weight
        pixel_state.accumulated_weight = fragment.weight
        return pixel_state
    
    else:
        # Z-Blend: Fragment is within the epsilon band
        # Blend into the running sum
        pixel_state.accumulated_color += fragment.color * fragment.weight
        pixel_state.accumulated_weight += fragment.weight
        return pixel_state
```
[G] The pseudocode assumes `fragment.h_z` is available from the splat's geometry. The "Replace" action resets the accumulation, which is consistent with maintaining a *single* layer where the new front-most fragment becomes the sole contributor.

**Edge Cases and Pitfalls**
*   **Hard Opaque Fringes:** Because the normalized output is always fully opaque wherever any surfel contributes, even at silhouette pixels where only a few surfels partially cover the pixel footprint and the accumulated weight is low, the method produces hard, opaque fringes along object boundaries [S1, unofficial]. This results in a "crusty" shell appearance rather than a gradual opacity falloff [S1, unofficial].
*   **Self-Occlusion Artifacts:** The single-layer constraint exacerbates self-occlusion errors at concavities (e.g., inside ears or under chins) [S1, unofficial]. Deeper surfels that should form a second layer are incorrectly rejected by the ternary test, resulting in holes or translucent patches that reveal the background through what should be an opaque, self-occluding object [S1, unofficial].
*   **Usage Context:** The ternary depth test is used only for surface reconstruction; regular depth tests are still required for triangle pipelines [S2].

## Operating characteristics
*   **Complexity:** Time complexity is linear with respect to the number of splats per pixel, as each splat is processed once with constant-time depth comparisons [G]. Memory complexity is $O(1)$ per pixel, as only a single depth value and accumulation buffer are maintained [S1, unofficial].
*   **Performance:** The method is computationally efficient due to the lack of multi-layer management and sorting overhead associated with depth-interval grouping [S1, unofficial].
*   **Strengths:** Simple implementation; low memory footprint; effective for simple, convex surfaces where single-layer assumption holds [S1, unofficial].
*   **Limitations:** Fails to handle complex geometry with self-occlusion or multiple surface layers per pixel [S1, unofficial]. Produces aliasing and silhouette artifacts due to the lack of coverage-based opacity [S1, unofficial].
*   **When to use:** Suitable for simple surface splatting where performance is critical and geometry is relatively simple (e.g., convex objects) [S1, unofficial].
*   **When not to use:** Unsuitable for complex scenes with concavities, self-occlusion, or where accurate silhouette rendering is required [S1, unofficial].

## Background
The Single-Layer Ternary Depth Test was introduced by Weyrich et al. in 2007 as a method for surface splatting [S1, unofficial][S2]. It was designed to handle overlapping splats during surface reconstruction by providing a more robust depth test than the standard binary pass/fail test [S2]. It serves as a baseline for comparison against more advanced techniques like depth-interval grouping, which was introduced to address the limitations of the single-layer approach [S1, unofficial].

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S2] A hardware architecture for surface splatting, https://doi.org/10.1145/1275808.1276490, fetched 2026-10-28
