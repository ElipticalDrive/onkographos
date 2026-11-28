# Temporal Optimization Heuristics

Also called: Dynamic DiffBMP, Dynamic DiffBMP for Videos, Dynamic DiffBMP heuristics, Dynamic Rendering with Spatial Constraint, Frame-wise Fidelity Heuristic, Sequential Optimization for Video

## Overview
Temporal Optimization Heuristics is a method for modeling dynamic content in differentiable bitmap primitive (DiffBMP) rendering by optimizing video frames sequentially to achieve both per-frame fidelity and temporal anti-flicker [S1, unofficial]. It addresses two specific failure modes in sequential optimization: "stuck" primitives that block gradients in changing regions, and parameter drift in static regions that causes flicker [S1, unofficial]. Unlike standard sequential optimization which simply warm-starts from the previous frame, this method applies three lightweight heuristics—warm-starting, removing stuck primitives, and freezing unchanged regions—to balance these competing goals [S1, unofficial].

## How it works
The method operates on a sequence of target images $I^f_{target}$ for frames $f=1 \dots N$ [S1, unofficial]. It maintains a set of $N$ bitmap primitives per frame, where each primitive $i$ is defined by parameters $\Theta_i = (x_i, y_i, s_i, \theta_i, \nu_i, c_i)$, representing position, scale, rotation, opacity logit, and color [S1, unofficial]. The opacity is computed as $\alpha_i = \alpha_{max} \sigma(\nu_i)$, where $\sigma$ is the sigmoid function [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   Sequence of target images $\{I^f_{target}\}_{f=1}^N$ [S1, unofficial].
*   Previous frame's optimized parameters $\Theta_{f-1}^\star$ (for $f > 1$) [S1, unofficial].
*   Spatial grid dimensions $(n_h, n_w)$ [S1, unofficial].
*   Thresholds: scale $\tau_{scale}$, opacity $\tau_\alpha$, z-order percentile $\zeta$ [S1, unofficial].
*   Intervention cap $K$ (top-K primitives per region) [S1, unofficial].
*   Decay factor $\eta \in (0, 1)$ [S1, unofficial].

**Outputs:**
*   Optimized parameters $\Theta_f^\star$ for each frame $f$ [S1, unofficial].

### Data Structures
*   **Primitive List:** Array of $N$ primitives [S1, unofficial]. Each primitive $i$ stores $(x_i, y_i, s_i, \theta_i, \nu_i, c_i)$ [S1, unofficial].
*   **Freeze Mask:** Boolean array `freeze` of size $N$, where `freeze[i]` is true if the primitive is frozen [S1, unofficial].
*   **Inter-frame Difference Mask:** Binary mask $D \in \{0, 1\}^{H \times W}$, where $D[x,y]=1$ if pixel $(x,y)$ changed between frames [S1, unofficial].
*   **Spatial Grid:** The image canvas is partitioned into $n_h \times n_w$ regions $R_j$ [S1, unofficial].

### Algorithm Steps
The optimization for frame $f$ proceeds as follows:

1.  **Warm-Start Initialization:**
    Initialize the parameters for frame $f$ ($\Theta_f$) using the optimized parameters from the previous frame ($\Theta_{f-1}^\star$) [S1, unofficial]. For the first frame, standard initialization is used [S1, unofficial].

2.  **Compute Inter-frame Difference Mask:**
    Calculate the difference mask $D$ to identify pixels with content changes between the current target $I^f_{target}$ and the previous target $I^{f-1}_{target}$ [S1, unofficial]:
    $$
    D[x,y] = \begin{cases} 
    1 & \text{if } I^f_{target}[x,y] \neq I^{f-1}_{target}[x,y] \\
    0 & \text{otherwise}
    \end{cases}
    $$
    [S1, unofficial]

3.  **Compute Freeze Masks:**
    For each primitive $i$, compute its bounding box $B_i$ from its position $(x_i, y_i)$, scale $s_i$, and rotation $\theta_i$ [S1, unofficial]. A primitive is marked as frozen if its bounding box does not intersect the support of the difference mask $D$ (i.e., it covers only unchanged pixels) [S1, unofficial]:
    $$
    \text{freeze}_i \leftarrow \mathbb{1}(B_i \cap \text{support}(D) = \emptyset)
    $$
    [S1, unofficial]

4.  **Partition Canvas:**
    Divide the image canvas into a spatial grid of size $n_h \times n_w$ to localize decisions [S1, unofficial]. Let $R_j$ denote the $j$-th region, and $P_j$ be the set of primitives whose centers $(x_i, y_i)$ fall within $R_j$ [S1, unofficial].

5.  **Identify and Decay Stuck Primitives:**
    For each region $R_j$:
    a.  **Score Primitives:** For each non-frozen primitive $i \in P_j$:
        *   Compute opacity: $\alpha_i = \alpha_{max} \sigma(\nu_i)$ [S1, unofficial].
        *   Evaluate three stuck criteria simultaneously [S1, unofficial]:
            1.  Large scale: $c_1(i) = \mathbb{1}(s_i \ge \tau_{scale} \cdot W)$, where $W$ is the image width [S1, unofficial].
            2.  High opacity: $c_2(i) = \mathbb{1}(\alpha_i \ge \tau_\alpha)$ [S1, unofficial].
            3.  Front z-order: $c_3(i) = \mathbb{1}(\text{depth-rank}_i \ge \zeta \cdot |P_j|)$, where $\text{depth-rank}_i$ is the rank of primitive $i$ by depth (front-most is rank 1) within the region [S1, unofficial].
        *   Determine if stuck: $\text{is\_stuck}_i = (c_1(i) + c_2(i) + c_3(i) = 3) \land (\neg \text{freeze}_i)$ [S1, unofficial].
        *   Compute visibility-weighted score: $score_i = s_i \cdot \alpha_i \cdot \text{is\_stuck}_i$ [S1, unofficial].
    b.  **Select and Decay:**
        *   Select the top-$K$ primitives with the highest $score_i$ in region $R_j$ as candidates $S_j$ [S1, unofficial].
        *   For each candidate $i \in S_j$, update its opacity logit:
            $$
            \nu_i^\star \leftarrow \eta \cdot \nu_i
            $$
            [S1, unofficial]
            This decay reduces the opacity of dominant primitives, allowing finer primitives behind them to take over and improving frame-wise fidelity [S1, unofficial].

6.  **Optimization:**
    Perform gradient descent optimization for frame $f$ using the standard DiffBMP rendering equation and losses, respecting the freeze masks (frozen primitives do not update) [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x):
    return 1.0 / (1.0 + math.exp(-x))

ALPHA_MAX = 1.0

def copy_params(params):
    return [p.copy() for p in params]

def temporal_optimization_heuristics(frame_f, frame_f_minus_1, prev_params, 
                                     image_width, image_height, 
                                     n_h, n_w, tau_scale, tau_alpha, 
                                     zeta, K, eta):
    # Step 1: Warm-start initialization
    current_params = copy_params(prev_params)
    N = len(current_params)
    
    # Step 2: Compute inter-frame difference mask D
    H, W, _ = frame_f.shape
    D = [[False] * W for _ in range(H)]
    for y in range(H):
        for x in range(W):
            if frame_f[y][x] != frame_f_minus_1[y][x]:
                D[y][x] = True
                
    # Step 3: Compute freeze masks
    freeze_flags = [False] * N
    for i in range(N):
        B_i = compute_bounding_box(current_params[i].x, current_params[i].y, 
                                   current_params[i].s, current_params[i].theta, image_width, image_height)
        if not intersect(B_i, D):
            freeze_flags[i] = True
            
    # Step 4: Partition canvas into n_h x n_w spatial regions
    regions = create_spatial_grid(n_h, n_w, image_width, image_height)
    
    # Step 5: Identify and decay stuck primitives
    for j, region in enumerate(regions):
        P_j = [i for i in range(N) if point_in_region(current_params[i].x, current_params[i].y, region)]
        
        scores = []
        for i in P_j:
            if freeze_flags[i]:
                continue
                
            alpha_i = ALPHA_MAX * sigmoid(current_params[i].nu)
            
            c1 = 1 if current_params[i].s >= tau_scale * image_width else 0
            c2 = 1 if alpha_i >= tau_alpha else 0
            
            sorted_prims_in_region = sorted(P_j, key=lambda idx: current_params[idx].z)
            depth_rank_i = sorted_prims_in_region.index(i) + 1
            c3 = 1 if depth_rank_i >= zeta * len(P_j) else 0
            
            is_stuck = (c1 + c2 + c3 == 3)
            score = current_params[i].s * alpha_i * is_stuck
            scores.append((i, score))
            
        scores.sort(key=lambda x: x[1], reverse=True)
        top_k_candidates = scores[:K]
        
        for i, score in top_k_candidates:
            current_params[i].nu = eta * current_params[i].nu
            
    return current_params, freeze_flags

def compute_bounding_box(x, y, s, theta, W, H):
    # Not given in the sources.
    pass

def intersect(box, mask):
    # Not given in the sources.
    pass

def create_spatial_grid(n_h, n_w, W, H):
    regions = []
    for r in range(n_h):
        for c in range(n_w):
            y_start = int(r * H / n_h)
            y_end = int((r + 1) * H / n_h)
            x_start = int(c * W / n_w)
            x_end = int((c + 1) * W / n_w)
            regions.append((x_start, y_start, x_end, y_end))
    return regions

def point_in_region(x, y, region):
    x_start, y_start, x_end, y_end = region
    return x_start <= x < x_end and y_start <= y < y_end
```

### Parameters
*   $\tau_{scale}$: Threshold for large scale eligibility. Typical value: Not given in the sources. Used in condition $s_i \ge \tau_{scale} \cdot W$ [S1, unofficial].
*   $\tau_\alpha$: Threshold for high opacity eligibility. Typical value: Not given in the sources. Used in condition $\alpha_i \ge \tau_\alpha$ [S1, unofficial].
*   $\zeta$: Percentile threshold for front z-order. Typical value: Not given in the sources. Targets front-most strokes [S1, unofficial].
*   $K$: Cap on the number of interventions (decays) per spatial region. Typical value: Not given in the sources [S1, unofficial].
*   $\eta$: Decay strength factor for opacity logits. Range: $(0, 1)$. Typical value: Not given in the sources [S1, unofficial].
*   $(n_h, n_w)$: Spatial grid dimensions controlling granularity. Typical value: Not given in the sources [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** The difference mask $D$ uses exact pixel inequality. Compression artifacts or noise may cause false positives in $D$, leading to unnecessary unfreezing of primitives. [G] A thresholded difference (e.g., $|I_f - I_{f-1}| > \epsilon$) is often used in practice to mitigate this, though the source specifies inequality.
*   **Boundaries:** The spatial grid partitioning must handle primitives near region boundaries. The source notes primitives are assigned to a region based on their center $(x_i, y_i)$ [S1, unofficial]. Primitives near boundaries might be evaluated in only one region, potentially missing interactions. [G]
*   **Degenerate Input:** If the previous frame's parameters are poor, warm-starting may propagate errors [S1, unofficial]. The "stuck" heuristic helps mitigate this by decaying dominant primitives that might otherwise block correct optimization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The heuristics add negligible overhead compared to the optimization step itself [S1, unofficial]. The difference mask computation is $O(HW)$ [G]. The freeze mask computation is $O(N)$ [G]. The stuck primitive identification involves sorting primitives within regions, which is $O(N \log N)$ in the worst case but typically $O(N)$ if regions are small [G]. The decay step is $O(N)$ [G]. Overall, the heuristic overhead is linear in the number of primitives and pixels.
*   **Memory Complexity:** The method requires storing the difference mask $D$ ($O(HW)$) and freeze flags ($O(N)$) [G]. This is minimal compared to the memory required for storing primitive parameters and intermediate rendering states.
*   **Measured Performance:**
    *   Removing stuck primitives improved PSNR from 29.26 dB to 29.58 dB in a specific example [S1, unofficial].
    *   Temporal flicker (tLP) drops from $7.41 \pm 8.89$ (none enabled) to $1.84 \pm 1.19$ (all three enabled) [S1, unofficial].
    *   Per-frame PSNR increases from $24.19 \pm 2.36$ to $24.38 \pm 2.23$ [S1, unofficial].
    *   SSIM increases from $0.616 \pm 0.106$ to $0.630 \pm 0.069$ [S1, unofficial].
*   **Strengths:**
    *   Prevents "stuck" primitives from suppressing local detail by decaying their opacity [S1, unofficial].
    *   Prevents flicker in static regions by freezing primitives that do not overlap with the inter-frame difference mask [S1, unofficial].
    *   Lightweight and data-agnostic, adding negligible overhead [S1, unofficial].
*   **Limitations:**
    *   Improvements in flicker suppression can trade off with per-frame fidelity [S1, unofficial].
    *   Requires careful tuning of thresholds ($\tau_{scale}, \tau_\alpha, \zeta, \eta, K$) for different scenes.
*   **When to use:** Suitable for video rendering with DiffBMP where temporal stability is critical and computational resources allow for sequential optimization.
*   **When not to use:** Not suitable for real-time rendering due to sequential dependency. Not suitable if per-frame fidelity is prioritized over temporal stability without careful tuning.

## Background
The method originates from the DiffBMP (Differentiable Rendering with Bitmap Primitives) paper [S1, unofficial]. It was developed to extend DiffBMP from static images to dynamic content (videos) [S1, unofficial]. The approach is inspired by sequential optimization techniques used in 3D Gaussian Splatting (3DGS) [32], but adapted for the specific characteristics of bitmap primitives, which do not explicitly distinguish background [S1, unofficial]. The heuristics address the unique failure modes of DiffBMP, such as primitives getting "stuck" and recoloring instead of relocating [S1, unofficial].

## Sources
[S1] DiffBMP: Differentiable Rendering with Bitmap Primitives, https://doi.org/10.48550/arXiv.2602.22625, fetched 2026-10-27 (unofficial)
