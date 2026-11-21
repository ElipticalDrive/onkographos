# Hybrid Transparency

Also called: Core-Tail Blending, Hybrid Transparency Blending, Hybrid Transparency-based blending, Hybrid transparency for 3DGS, Partial depth-ordering blending

## Overview
Hybrid Transparency is a rendering technique that mitigates popping artifacts in 3D Gaussian Splatting by combining depth-ordered alpha blending for the first $K$ fragments (the 'core') with order-independent accumulation for the remaining fragments (the 'tail') [S1, S2]. Unlike standard 3DGS which uses approximate view-space depth sorting, or StopThePop which employs expensive hierarchical sorting, Hybrid Transparency guarantees correct per-pixel depth ordering only for the front-most contributions, allowing the rest to be blended without sorting [S1]. This approach maintains real-time performance while ensuring temporal stability and multi-view consistency, particularly in VR applications [S2].

## How it works
**Inputs:** A set of 3D Gaussians defined by mean $\mu \in \mathbb{R}^3$, opacity $o \in \mathbb{R}$, orientation vectors $t_u, t_v, t_w \in \mathbb{R}^3$, and scales $s_u, s_v, s_w \in \mathbb{R}$ [S1]. Camera parameters (view and projection matrices).
**Outputs:** Final pixel color $C$ and opacity $\alpha$.
**Parameters:**
*   $K$: Number of fragments in the core. Default/Optimal value is 16 [S1, S2].
*   $\tau_K$: Minimum alpha threshold for a fragment to be considered for the core. Default value is 0.05 [S1].

**Data Structures:**
*   **Core Buffer:** A compact 64-bit data structure per thread storing the top $K$ fragments for a pixel. This reduces register usage and swap operations during insertion sort [S2].
*   **Tail Accumulators:** Per-pixel accumulators for RGB color and alpha of all non-core fragments [S1].

**Steps:**

1.  **Splat Staging and Bounding Box Computation:**
    Compute the screen-space bounding box for each 3D Gaussian. This can be done using plane-fitting to determine exact bounding boxes [S2]. Create unordered per-tile fragment lists. Depth ordering is omitted at this stage as it is resolved during blending [S2].

2.  **Pixel-Specific Depth Evaluation ($x_{view}$):**
    For each splat intersecting a pixel, compute the point of maximum contribution along the pixel's viewing ray in view space, denoted as $x_{view}$ [S1]. This value is used to determine if the splat belongs to the core or the tail. It is computed efficiently using intermediate values from the splat evaluation process [S1].
    $$ x_{view} = \text{z-value in view space at point of max contribution} $$
    *Note: Using only the view-space depth of the mean is insufficient and causes popping artifacts [S1].*

3.  **Core Identification and Sorting:**
    Identify the splats contributing to the pixel. Sort these splats based on $x_{view}$. Select the first $K$ splats (or fewer if fewer than $K$ splats exist) that also satisfy $\alpha \ge \tau_K$ to form the 'core' [S1].
    *   If insertion sort is used for the core, reduce tile size to $8 \times 8$ to mitigate thread divergence [S1].

4.  **Core Blending:**
    Alpha-blend the core fragments in their correct depth order using standard $\alpha$-blending [S1].
    Let $C_{core}$ and $\alpha_{core}$ be the accumulated color and opacity of the core.
    $$ C_{core} = \sum_{i=1}^{K} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    $$ \alpha_{core} = 1 - \prod_{i=1}^{K} (1 - \alpha_i) $$
    where $c_i, \alpha_i$ are the color and opacity of the $i$-th core fragment.

5.  **Tail Accumulation:**
    Accumulate the contributions of all remaining $N-K$ fragments (the 'tail') without sorting [S1].
    $$ C_{tail} = \sum_{j=K+1}^{N} c_j $$
    $$ \alpha_{tail} = \sum_{j=K+1}^{N} \alpha_j $$
    *Note: The tail accumulation is order-independent [S1].*

6.  **Final Compositing:**
    Combine the core and tail. The tail is treated as a single opaque or semi-transparent layer behind the core.
    $$ C = C_{core} + C_{tail} (1 - \alpha_{core}) $$
    $$ \alpha = \alpha_{core} + \alpha_{tail} (1 - \alpha_{core}) $$
    *Note: The computation of $T_{tail}$ (transmittance of the tail) is more accurate than average-based formulations [S1].*

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_transparency_render(gaussians, camera, K=16, tau_K=0.05):
    # Initialize per-pixel accumulators
    # Core: List of up to K fragments, sorted by depth
    # Tail: Accumulated color (vec3) and alpha (float)
    core_buffers = initialize_core_buffers(K)
    tail_color = zeros(num_pixels, 3)
    tail_alpha = zeros(num_pixels)

    for gaussian in gaussians:
        # 1. Compute screen-space bounding box and tile assignment
        tiles = get_intersecting_tiles(gaussian)
        
        # 2. Evaluate splat at pixel locations
        for tile in tiles:
            pixels = get_pixels_in_tile(tile)
            for pixel in pixels:
                # Compute point of max contribution in view space
                x_view = compute_max_contribution_depth(gaussian, pixel, camera)
                
                # Evaluate Gaussian at this point
                alpha = gaussian.evaluate_alpha(x_view)
                color = gaussian.evaluate_color(x_view)
                
                # Skip if below threshold or too transparent
                if alpha < tau_K:
                    continue
                
                # Determine if this fragment belongs to Core or Tail
                # We need to know if it is among the top K closest fragments.
                # In a parallel implementation, this requires atomic operations 
                # or a two-pass approach. Here we assume a logical flow.
                
                is_core_candidate = (alpha >= tau_K)
                
                if is_core_candidate:
                    # Try to insert into core buffer
                    inserted = try_insert_core(core_buffers[pixel], x_view, color, alpha, K)
                    if inserted:
                        # It is in the core. 
                        # If it displaces an existing core fragment, that fragment 
                        # might move to tail? 
                        # Standard HT implementations usually fix the set of K 
                        # closest. If a new closer one arrives, the farthest core 
                        # one is ejected to tail.
                        ejected = eject_farthest_if_needed(core_buffers[pixel])
                        if ejected:
                            tail_color[pixel] += ejected.color
                            tail_alpha[pixel] += ejected.alpha
                    else:
                        # Not in top K, goes to tail
                        tail_color[pixel] += color
                        tail_alpha[pixel] += alpha
                else:
                    # Below threshold, goes to tail
                    tail_color[pixel] += color
                    tail_alpha[pixel] += alpha

    # Final Compositing
    final_color = zeros(num_pixels, 3)
    final_alpha = zeros(num_pixels)
    
    for pixel in pixels:
        core_res = blend_core(core_buffers[pixel])
        C_core, A_core = core_res.color, core_res.alpha
        
        C_tail = tail_color[pixel]
        A_tail = tail_alpha[pixel]
        
        # Composite Core and Tail
        # Tail is behind the core
        final_color[pixel] = C_core + C_tail * (1.0 - A_core)
        final_alpha[pixel] = A_core + A_tail * (1.0 - A_core)

    return final_color, final_alpha

def try_insert_core(buffer, x_view, color, alpha, K):
    # Insert into sorted list of size <= K
    # If buffer is full and new fragment is closer (smaller x_view) than 
    # the farthest in buffer, replace farthest and return True (ejected)
    # Else return False (not inserted)
    pass

def eject_farthest_if_needed(buffer):
    # Remove the fragment with largest x_view from buffer
    # Return that fragment
    pass

def blend_core(buffer):
    # Standard alpha blending of sorted fragments
    acc_color = vec3(0)
    acc_alpha = 0.0
    for frag in buffer:
        acc_color += frag.color * frag.alpha * (1.0 - acc_alpha)
        acc_alpha += frag.alpha * (1.0 - acc_alpha)
    return acc_color, acc_alpha
```

## Subtopics

### Multi-layers Alpha Blending Approximation
A sorting strategy that approximates the $\tau_{max}$ order of hit particles by storing the $k$-farthest hit particles (typically $k=16$) in a buffer and incrementally alpha-blending the closest hits until transmittance vanishes [S3]. Unlike Hybrid Transparency which stores the $k$-closest and blends farthest hits, MLAB stores farthest and blends closest [S3]. Hybrid Transparency recovers exact $k$-closest particles but requires processing all particles, which can be slow without optimizations [S3].

## Operating characteristics
*   **Time Complexity:** Rendering time is reduced compared to global sorting methods because it avoids the overhead of sorting the tail [S1]. However, it requires processing all splats to determine core/tail membership, which can be slower than simple early-stopping alpha blending if not optimized [S3].
*   **Memory Complexity:** Uses compact 64-bit structures for core fragments to minimize register usage [S2].
*   **Performance:** Achieves a training speedup of $2.1\times$ and rendering speedup of $1.6\times$ compared to standard 3DGS on average [S1]. It is significantly faster than StopThePop [S1].
*   **Strengths:** Eliminates popping artifacts caused by approximate sorting [S1]. Provides temporally stable rendering suitable for VR [S2].
*   **Limitations:** Cannot use early stopping based on transmittance threshold by default, which may impact performance in scenes with high depth complexity [S1]. Slightly lower PSNR than standard 3DGS due to the absence of screen-space aliasing filters [S1].
*   **Edge Cases:**
    *   $K=0$ (no sorting) causes foreground transparency issues and is invalid [S1].
    *   Omitting tail computation during training causes catastrophic failure, though it has a small impact on inference quality [S1].
    *   Insertion sort for the core causes thread divergence; mitigated by using small tile sizes ($8 \times 8$) [S1].

## Background
Hybrid Transparency was originally proposed by Wymann et al. [Wym16] and extended by Maule et al. [MCTB13] for gradient descent-based optimization [S1]. It was integrated into the 3D Gaussian Splatting framework by Hahlbohm et al. [HFW*25] to address popping artifacts and perspective inaccuracies [S1, S2]. The method is particularly relevant for VR/AR applications where high frame rates and multi-view consistency are critical [S2].

## Sources
[S1] Efficient Perspective‐Correct 3D Gaussian Splatting Using Hybrid Transparency, https://doi.org/10.1111/cgf.70014, fetched 2026-10-30
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
[S3] 3DGUT: Enabling Distorted Cameras and Secondary Rays in Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.02425, fetched 2026-10-26
