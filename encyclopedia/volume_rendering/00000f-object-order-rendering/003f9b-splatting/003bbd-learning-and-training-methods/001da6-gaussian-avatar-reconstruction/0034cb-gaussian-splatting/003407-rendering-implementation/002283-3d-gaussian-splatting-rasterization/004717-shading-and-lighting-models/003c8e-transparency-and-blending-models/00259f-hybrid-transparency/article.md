# Hybrid Transparency

Also called: Core-Tail Blending, Hybrid Transparency Blending, Hybrid Transparency-based blending, Hybrid transparency for 3DGS, Partial depth-ordering blending

## Overview
Hybrid Transparency is a blending formulation that combines depth-ordered alpha blending for the first $K$ fragments (the 'core') with order-independent accumulation for the remaining fragments (the 'tail') to mitigate popping artifacts while maintaining real-time performance [S1, S2]. It solves the issue of incorrect per-pixel blending order caused by standard 3D Gaussian Splatting (3DGS) approximations, which leads to visual popping during camera motion [S1, S2]. Unlike [Analytic Transmittance 3DGS](4440-analytic-transmittance-3dgs.md) which refines the density integration, or [SparseOIT Rasterizer](7562-sparseoit-rasterizer.md) which uses full order-independent transparency, Hybrid Transparency avoids global sorting and expensive order-independent techniques by only sorting the most visually significant front-most splats [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians defined by mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$ (or scales and orientation), opacity $o$, and color $\mathbf{c}$ [S1, S2].
*   **Output:** Final pixel color $C$ and accumulated alpha $\alpha$.

**Data Structures:**
*   **Core Buffer:** Each thread uses a compact 64-bit data structure to store core fragments during blending. This reduces register usage and swaps during insertion sort [S2].
*   **Tile Fragment Lists:** Unordered per-tile fragment lists are created. Depth ordering is omitted during staging as hybrid transparency resolves depth ordering during blending [S2].

**Parameters:**
*   $K$: The number of foremost contributions blended explicitly in the core. Default/optimal value is 16 [S1, S2].
*   $\tau_K$: Minimum alpha of a fragment to be considered for the core. Default value is 0.05 [S1].

**Steps:**
1.  **Splat Staging and Bounding:** During splat staging, compute exact bounding boxes of each 3D Gaussian in screen space using plane-fitting [S2]. Create unordered per-tile fragment lists [S2]. Apply load balancing during tile fragment creation similar to Radl et al. [S2].
2.  **Depth Evaluation:** For each splat contributing to a pixel, compute the pixel-specific depth value $x_{view}$ (the z-value in view space). This is the splat's point of maximum contribution along the pixel ray in view space [S1]. It is computed efficiently using intermediate values from splat evaluation [S1].
    $$ x_{view} = \text{point of maximum contribution along pixel ray in view space} $$
    [S1]
3.  **Core Identification and Sorting:** Identify the initial $K$ splats contributing to each pixel color based on $x_{view}$. Sort these $K$ fragments correctly by depth per pixel [S1]. Note: Using only the view-space depth of the mean is insufficient and causes popping; correct per-pixel depth ordering is required [S1].
4.  **Core Blending:** Alpha-blend the first $K$ fragments (the core) in their correct depth order per pixel using standard $\alpha$-blending [S1].
5.  **Tail Accumulation:** Accumulate the contributions of the remaining fragments (the tail, $N-K$) using an order-independent residual. This involves accumulating the RGB color and alpha of all remaining splats without sorting [S1].
6.  **Final Composition:** Compute the final pixel color using the core blend result and the tail accumulation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_transparency_blend(fragments, K=16, tau_K=0.05):
    """
    fragments: List of splat contributions for a pixel, each containing:
               - color (RGB)
               - alpha (scalar)
               - x_view (depth in view space at point of max contribution)
    K: Number of core fragments to sort and blend correctly.
    tau_K: Minimum alpha threshold for core inclusion.
    """
    # Filter fragments that might be in the core based on alpha threshold
    # Note: In practice, all fragments are processed, but only top K by depth 
    # are sorted. The threshold tau_K is used to determine if a fragment 
    # is significant enough to be considered for the core if depth is ambiguous 
    # or as a heuristic.
    
    # Step 1: Identify potential core candidates
    # We need to find the K fragments with the smallest x_view (closest to camera)
    # that also satisfy alpha >= tau_K? 
    # The source says "identify the top K splats by depth" and "minimum alpha ... to be considered".
    # Implementation detail: Sort all fragments by x_view.
    
    # Sort all fragments by x_view (ascending)
    sorted_frags = sorted(fragments, key=lambda f: f.x_view)
    
    # Select the first K fragments as the Core
    core_frags = sorted_frags[:K]
    
    # The rest are the Tail
    tail_frags = sorted_frags[K:]
    
    # Step 2: Blend Core (Depth Ordered)
    # Standard front-to-back alpha blending
    core_color = vec3(0.0)
    core_alpha = 0.0
    
    for frag in core_frags:
        # Standard alpha blending: C_out = C_in + C_frag * alpha_frag * (1 - alpha_in)
        # But usually: C_out = C_frag * alpha_frag + C_in * alpha_in * (1 - alpha_frag)
        # Assuming pre-multiplied alpha or standard formulation:
        # C_new = C_frag + C_old * (1 - alpha_frag)
        # alpha_new = alpha_frag + alpha_old * (1 - alpha_frag)
        
        # Using standard non-premultiplied color blending logic often used in 3DGS:
        # C = C_frag * alpha_frag + C_old * alpha_old * (1 - alpha_frag)
        # alpha = alpha_frag + alpha_old * (1 - alpha_frag)
        
        # However, 3DGS typically uses pre-multiplied alpha for accumulation.
        # Let's assume standard accumulation:
        
        weight = frag.alpha * (1.0 - core_alpha)
        core_color = core_color + frag.color * weight
        core_alpha = core_alpha + frag.alpha * (1.0 - core_alpha)
        
        # Early exit if fully opaque? 
        # Hybrid transparency doesn't support early exit by default [S1], 
        # but we can stop if core_alpha >= 1.0 for efficiency.
        if core_alpha >= 1.0:
            break
            
    # Step 3: Accumulate Tail (Order Independent)
    # Accumulate RGB and Alpha of all remaining splats without sorting.
    # The source notes that T_tail is computed accurately.
    # T_tail is the transmittance of the tail.
    # C_tail = sum(C_i * alpha_i * T_prev_i) ? 
    # Since it's order independent, we just sum the contributions?
    # Maule et al. [MCTB13] and the extension in [S1] suggest:
    # The tail is blended without sorting. 
    # A common order-independent approach for the tail is to accumulate 
    # the color and alpha as if they were blended in any order, 
    # but this is an approximation.
    # The source says: "Accumulate the RGB color and alpha of all remaining splats (tail) without sorting."
    # And: "Compute final color using the core blend and tail accumulation."
    
    # Let's assume the tail is accumulated into a residual color and alpha.
    tail_color = vec3(0.0)
    tail_alpha = 0.0
    
    for frag in tail_frags:
        # Simple accumulation? Or order-independent blending?
        # If it's truly order-independent, we can't just add.
        # However, the source contrasts it with "blending without any sorting" which causes issues.
        # The "tail" in Hybrid Transparency usually refers to the unsorted remainder.
        # In [MCTB13], the tail is blended using an order-independent technique.
        # A simple approximation is to accumulate the color and alpha.
        # But to be precise, let's look at the formula reference.
        # Formula (17) and (18) in [S1] are not fully explicit in the text provided,
        # but the text says "accumulates the RGB color and alpha ... to then compute its color".
        
        # Let's assume a standard accumulation for the tail components:
        # This is an approximation.
        tail_color = tail_color + frag.color * frag.alpha
        tail_alpha = tail_alpha + frag.alpha * (1.0 - tail_alpha) # Approximate accumulation
        
    # Step 4: Final Composition
    # Combine Core and Tail.
    # The tail is treated as a single opaque/semi-transparent layer behind the core?
    # Or is it blended with the core result?
    # Usually: C_final = C_core + C_tail * (1 - alpha_core)
    # alpha_final = alpha_core + alpha_tail * (1 - alpha_core)
    
    final_color = core_color + tail_color * (1.0 - core_alpha)
    final_alpha = core_alpha + tail_alpha * (1.0 - core_alpha)
    
    return final_color, final_alpha
```
*Note: The pseudocode above is a simplified implementation of the logic described. The source [S1] emphasizes that the tail accumulation is more accurate than the average-based formulation of Maule et al. [MCTB13] and allows pre-computing partial derivatives. The exact mathematical formulation for the tail accumulation in [S1] relies on the specific extension of Maule et al., which is not fully detailed in the provided text snippets, but the logical flow is Core Sort+Blend -> Tail Accumulate -> Composite.*

**Edge Cases and Pitfalls:**
*   **Incorrect Depth Metric:** Using only the view-space depth of the mean causes popping artifacts. Must use $x_{view}$ (point of max contribution) [S1].
*   **K=0:** Blending without any sorting is invalid as it causes foreground transparency issues [S1].
*   **Training vs. Inference:** Omitting tail computation during training causes catastrophic failure, though it causes only a small drop in image quality during inference [S1].
*   **Divergence:** Insertion sort for per-pixel core causes significant divergence. Mitigated by reducing tile size to 8x8 [S1].

## Subtopics

### Multi-layers Alpha Blending Approximation
A sorting strategy to approximate the $\tau_{max}$ order of hit particles along a ray by storing the $k$-farthest hit particles in a buffer and incrementally alpha-blending the closest hits until transmittance vanishes [S3].
*   **Parameter:** $k$: Number of farthest particles to store in the buffer. Typical value: 16 [S3].
*   **Comparison:** MLAB stores the $k$-farthest particles and blends closest hits incrementally. Hybrid Transparency (HT) stores the $k$-closest particles and blends farthest hits incrementally. HT recovers exact $k$-closest particles but requires processing all particles, which is slow without optimizations [S3].

## Operating characteristics
*   **Time Complexity:** Rendering time is reduced because it avoids global depth-sorting overhead for the tail [S1]. It is significantly faster than StopThePop [RSP*24] which uses hierarchical sorting for all splats [S1].
*   **Measured Performance:** On Mip-NeRF360 datasets, Hybrid Transparency achieves a training speedup of 2.1× and renders 1.6× faster on average compared to standard 3DGS [S1]. It is faster than StopThePop [RSP*24] (e.g., 2.007s vs 5.116s blending time on Bicycle scene) [S1].
*   **Memory Complexity:** Uses a compact 64-bit data structure for core fragments, reducing register usage [S2].
*   **Strengths:** Mitigates popping artifacts; maintains real-time performance; avoids global presorting; perspective-correct.
*   **Limitations:** Slightly lower PSNR than 3DGS (possibly due to absence of anti-aliasing) [S1]; default implementation accumulates all tail contributions, which can be slow in scenes with high depth complexity (no early exit) [S1]; insertion sort divergence requires small tile sizes (8x8) [S1].
*   **When to use:** VR/AR applications or fast-paced scenes where popping artifacts are noticeable and high frame rates are required [S1, S2].
*   **When not to use:** Scenes with extremely high depth complexity where tail accumulation becomes a bottleneck and optimizations like depth prepass are not implemented [S1].

## Background
Hybrid Transparency was originally proposed by Wymann et al. [Wym16] (cited as [MCTB13] in the context of Maule et al.'s work, likely a typo in the source or different citation style, but [Wym16] is the alias) and later integrated into the 3D Gaussian Splatting framework by Hahlbohm et al. [HFW*25] [S1, S2]. It was developed to address the "popping" artifacts inherent in standard 3DGS rendering, which arise from approximate depth sorting. The method extends the formulation by Maule et al. [MCTB13] to improve behavior within gradient descent-based optimization schemes [S1]. It is used in VR editing tools (e.g., Splatshop) and efficient 3DGS renderers to provide temporally stable rendering without the performance cost of hierarchical sorting or ray tracing [S1, S2].

## Sources
[S1] Efficient Perspective‐Correct 3D Gaussian Splatting Using Hybrid Transparency, https://doi.org/10.1111/cgf.70014, fetched 2026-10-30
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
[S3] 3DGUT: Enabling Distorted Cameras and Secondary Rays in Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.02425, fetched 2026-10-26
