# Multi Resolution Alpha Blending

Also called: Per-pixel sorting and blending, TRIPS blending

## Overview
Multi Resolution Alpha Blending is a fragment accumulation strategy used in hierarchical point splatting to resolve depth-ordering ambiguities when multiple splats project to the same pixel. It solves the problem of correct alpha compositing in scenes with varying point densities by collecting, sorting, and blending fragments on a per-pixel basis rather than globally. It differs from global sorting methods by operating on small, localized lists, which reduces computational overhead and memory bandwidth usage [S1].

## How it works
**Inputs:** A set of splatted points projected to screen space, each with position $(x, y)$, depth $z$, opacity $\alpha$, color $c$, and a reference to its hierarchical layer index $l$.
**Outputs:** A final color value $C_\Lambda$ for each pixel.

**Data Structures:**
*   **Per-pixel list ($\Lambda_{l_i, x_i, y_i}$):** A dynamic array storing fragments for pixel $(x_i, y_i)$ at layer $l_i$.
*   **Fragment record:** Contains depth $z$, opacity $\alpha$, color $c$, and layer index $l$.
*   **Maximum size:** 16 elements per list [S1].

**Steps:**
1.  **Collection:** For each point, project it to screen space. For every pixel covered by the splat, append the fragment to the corresponding per-pixel list $\Lambda_{l_i, x_i, y_i}$. Store the pair $(z, \text{index})$ to allow efficient sorting [S1].
2.  **Clamping:** If a list exceeds 16 elements, discard the fragments with the largest depth values (furthest from the camera) to maintain the limit [S1].
3.  **Sorting:** Sort each list by depth $z$ in ascending order (front-to-back). Use GPU-friendly sorting algorithms, specifically warp-local (32 threads) and shuffle-based bitonic sorts. The sorting process iteratively replaces the last 16 elements with new unsorted ones until all fragments are processed [S1].
4.  **Blending:** Perform front-to-back alpha blending on the sorted list in the same pass as sorting, leveraging register storage for intermediate values [S1].

**Formulas:**
The final color $C_\Lambda$ for a pixel is computed as [S1]:
$$C_\Lambda = \sum_{m=1}^{|\Lambda|} T_m \cdot \alpha_m \cdot c_m$$

The transmittance $T_m$ at fragment $m$ is [S1]:
$$T_m = \prod_{i=1}^{m-1} (1 - \alpha_i)$$

Where:
*   $|\Lambda|$ is the number of fragments in the list (up to 16).
*   $\alpha_m$ is the opacity of fragment $m$.
*   $c_m$ is the color of fragment $m$.
*   $T_m$ is the transmittance, representing the probability that light passes through all preceding fragments $1$ to $m-1$.

**Parameters:**
*   **Max list size:** 16 [S1].
*   **Minimum contribution weight:** $\epsilon = 0.25$. To prevent the contribution of points from finer levels from disappearing during blending, their weight is clamped to at least this value [S1].

**Edge cases and pitfalls:**
*   **Occlusion errors:** In scenes with large deviations in point density, the neural network may fail to correctly evaluate occlusion in edge cases [S1].
*   **Mitigation:** Include points from coarser layers during blending to ensure robust occlusion handling. The additional cost is minimal (< 0.5ms) [S1].
*   **Precision:** The approximation of limiting lists to 16 elements causes no significant loss of quality because the blending contribution of later (distant) points is very low [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_resolution_alpha_blending(fragments):
    """
    fragments: list of dicts with keys 'x', 'y', 'z', 'alpha', 'color', 'layer'
    """
    # 1. Initialize per-pixel lists
    # Using a dictionary for sparse storage: key=(x, y, layer), value=list of fragments
    pixel_lists = {}
    
    for frag in fragments:
        key = (frag['x'], frag['y'], frag['layer'])
        if key not in pixel_lists:
            pixel_lists[key] = []
        pixel_lists[key].append(frag)
        
    final_colors = {}
    
    # 2. Process each pixel list
    for key, frag_list in pixel_lists.items():
        x, y, layer = key
        
        # 3. Sort by depth (ascending)
        # GPU-friendly sort: warp-local and shuffle-based bitonic sort
        # Implementation detail: iterative replacement of last 16 elements
        # until empty. Here simplified to a standard sort for clarity, 
        # but in practice uses register-based sorting.
        frag_list.sort(key=lambda f: f['z'])
        
        # 4. Clamp to max size 16
        if len(frag_list) > 16:
            frag_list = frag_list[:16]
            
        # 5. Front-to-back blending
        transmittance = 1.0
        accumulated_color = [0.0, 0.0, 0.0]
        
        for frag in frag_list:
            alpha = frag['alpha']
            color = frag['color']
            
            # Clamp minimum contribution weight if necessary (e.g., for fine layers)
            # Note: The source mentions ensuring contribution is at least epsilon=0.25
            # This is typically applied to the weight calculation or alpha scaling
            # depending on the specific implementation of "contribution".
            # Standard alpha blending:
            weight = transmittance * alpha
            accumulated_color = [
                accumulated_color[i] + weight * color[i] for i in range(3)
            ]
            transmittance *= (1.0 - alpha)
            
        final_colors[(x, y)] = accumulated_color
        
    return final_colors
```

## Subtopics

### Neural Reconstruction Network
A compact neural network that consolidates a feature image pyramid of $n$ layers into a single full-resolution image, resembling a U-Net decoder. It incorporates a single gated convolution [YLY* 19] in each layer with a self-bypass connection and a feature size of 32. It requires a significantly smaller and more efficient network than hole-filling networks because the renderer fills gaps autonomously and generates smooth output [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of splats and pixels, but optimized by the small constant factor of the 16-element list limit. Per-pixel sorting is faster than global sorting for this approach due to the higher amount and smaller sizes of points [S1].
*   **Memory Complexity:** Proportional to the number of pixels times the maximum list size (16).
*   **Performance:** The sorting and accumulation pass is critical. The use of GPU-friendly sorting (warp-local, shuffle-based) ensures high throughput. The additional cost of including coarser layer points for occlusion handling is < 0.5ms [S1].
*   **Strengths:** Efficient handling of varying point densities; avoids global sorting overhead; robust occlusion via coarse layer inclusion.
*   **Limitations:** Approximation of limiting to 16 fragments may miss very distant but significant contributions (though deemed negligible); requires careful handling of minimum weight contributions for fine layers [S1].

## Background
Multi Resolution Alpha Blending is a core component of the TRIPS (Trilinear Point Splatting for Real-Time Radiance Field Rendering) method [S1]. It was introduced to address the inefficiencies of global sorting in hierarchical point-based rendering, leveraging the fact that per-pixel lists are small and numerous, making local sorting more efficient on GPU architectures [S1].

## Sources
[S1] TRIPS: Trilinear Point Splatting for Real‐Time Radiance Field Rendering, https://doi.org/10.1111/cgf.15012, fetched 2026-10-27
