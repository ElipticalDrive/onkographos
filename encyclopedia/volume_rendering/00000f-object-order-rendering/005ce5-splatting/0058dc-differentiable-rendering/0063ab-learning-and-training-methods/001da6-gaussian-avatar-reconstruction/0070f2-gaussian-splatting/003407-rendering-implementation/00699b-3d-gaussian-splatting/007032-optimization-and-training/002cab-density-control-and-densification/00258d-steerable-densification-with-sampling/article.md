# Steerable Densification with Sampling

Also called: Score-based ranking densification

## Overview
Steerable Densification with Sampling is a density control method for 3D Gaussian Splatting that uses image loss and per-Gaussian attributes to rank primitives for resampling, executed at a significantly reduced frequency to prevent over-densification [S1]. It differs from standard 3DGS densification by replacing deterministic gradient-based splitting with a probabilistic, score-weighted sampling process, allowing for more interpretable and stable convergence [S1].

## How it works
**Inputs:**
*   Set of $M$ 3D Gaussians $\mathcal{G} = \{g_j\}_{j=1}^M$ [S1].
*   Set of $N$ training camera views $\mathcal{V} = \{v_i\}_{i=1}^N$ [S1].
*   Rendered images $\mathcal{R} = \{r_i\}_{i=1}^N$ corresponding to $\mathcal{V}$ [S1].
*   Ground truth images $\mathcal{I} = \{I_i\}_{i=1}^N$ [G, standard for loss computation].
*   Densification budget $B$ (number of new primitives to add) [S1].
*   Hyperparameters: $\lambda_1=0.5, \lambda_2=0.5$ (saliency weights), scoring weights $w_{\nabla}=50, w_{c}=0.1, w_{D}=50, w_{S}=10, w_{B}=50, w_{z}=5, w_{o}=100, w_{s}=25$ [S1].

**Outputs:**
*   Updated set of Gaussians $\mathcal{G}'$ with $M+B$ primitives (before pruning) [S1].

**Data Structures:**
*   Per-view saliency matrix $S_v \in \mathbb{R}^{H \times W}$ for each view $v$ [S1].
*   Global score vector $S_G \in \mathbb{R}^M$, where $S_{G,j}$ is the score for Gaussian $g_j$ [S1].

**Steps:**

1.  **Compute Per-View Saliency Matrices:** For each view $v_i \in \mathcal{V}$, compute the L1 loss map $L_1(v_i, r_i) = |I_i - r_i|$ [G]. Compute a Laplacian filter map $E(v_i)$ as a high-pass filter on the rendered image [S1]. Define a binary ROI mask $\mathbb{1}_{ROI} \in \{0,1\}^{H \times W}$ [S1]. Calculate saliency:
    $$ S_{v_i} = \mathbb{1}_{ROI} \odot (\lambda_1 L_1(v_i, r_i) + \lambda_2 E(v_i)) $$
    [S1]

2.  **Compute Gaussian Scores:** Initialize $S_G = \mathbf{0} \in \mathbb{R}^M$ [S1]. For each Gaussian $g_j \in \mathcal{G}$, perform an auxiliary rendering pass or use cached projection data to obtain per-pixel attributes for all views $v_i$ where $g_j$ contributes [S1]. For each contributing pixel $p$ in view $v_i$, retrieve attributes: positional gradient magnitude $\nabla_{g_j}$, pixel count $c_{g_j,i}$, binary coverage mask $\mathbb{1}_{g_j}$, pixel-to-center distance $D_{g_j}$, saliency value $S_{v_i}$, blending weight $B_{g_j}$, depth $z_{g_j}$, opacity $o_{g_j}$, and scale $s_{g_j}$ [S1]. Apply median scaling to each attribute across the dataset to remove outliers [S1]. Compute the local score contribution $F$ by summing the median-scaled attributes multiplied by their tunable coefficients for specific use cases and the photometric loss $L_1(v_i, r_i)$ [S1]. Accumulate $F$ over all covered pixels to update $S_{G,j}$ [S1].

3.  **Resampling (Densification):** Sample $B$ indices from $\{1, \dots, M\}$ using $S_G$ as sampling weights [S1]. For each sampled Gaussian $g_{src}$, clone it to create a new Gaussian $g_{new}$ with identical position, scale, and rotation [G]. Set $g_{new}$ opacity to $o_{src} / 2$ to preserve total opacity contribution [G]. Add $g_{new}$ to $\mathcal{G}$ [S1].

4.  **Pruning:** Remove Gaussians with opacity below $0.025$ or excessive size to reduce noise and memory usage [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def steerable_densification(Gaussians, Views, Rendered, GroundTruth, Budget, Iteration):
    # Step 1: Compute Saliency Matrices
    SaliencyMatrices = []
    for v in Views:
        L1_Loss = [[abs(GroundTruth[v][y][x] - Rendered[v][y][x]) for x in range(W)] for y in range(H)]
        Laplacian = apply_laplacian_filter(Rendered[v])
        ROI_Mask = get_roi_mask(v)
        S_v = [[ROI_Mask[y][x] * (0.5 * L1_Loss[y][x] + 0.5 * Laplacian[y][x]) for x in range(W)] for y in range(H)]
        SaliencyMatrices.append(S_v)

    # Step 2: Compute Gaussian Scores
    GaussianScores = [0.0] * len(Gaussians)
    weights = {'grad': 50, 'count': 0.1, 'dist': 50, 'saliency': 10, 'blend': 50, 'depth': 5, 'opacity': 100, 'scale': 25}

    # Collect all attribute values for median scaling
    attr_values = {'grad': [], 'count': [], 'dist': [], 'saliency': [], 'blend': [], 'depth': [], 'opacity': [], 'scale': []}
    for j, g in enumerate(Gaussians):
        # [G: requires auxiliary rendering pass or cached projection data]
        pixels_data = get_gaussian_pixel_data(g, Views)
        for view_idx, view in enumerate(Views):
            if view not in pixels_data: continue
            L1_val = L1_Loss_map[view_idx]
            S_v = SaliencyMatrices[view_idx]
            for p in pixels_data[view]:
                attr_values['grad'].append(p.grad_magnitude)
                attr_values['count'].append(p.pixel_count)
                attr_values['dist'].append(p.distance_to_center)
                attr_values['saliency'].append(S_v[p.y][p.x])
                attr_values['blend'].append(p.blending_weight)
                attr_values['depth'].append(p.depth)
                attr_values['opacity'].append(g.opacity)
                attr_values['scale'].append(g.scale)

    # Compute medians for scaling
    medians = {k: (sorted(v)[len(v)//2] if v else 1.0) for k, v in attr_values.items()}

    for j, g in enumerate(Gaussians):
        score = 0.0
        pixels_data = get_gaussian_pixel_data(g, Views)
        for view_idx, view in enumerate(Views):
            if view not in pixels_data: continue
            L1_val = L1_Loss_map[view_idx]
            S_v = SaliencyMatrices[view_idx]
            for p in pixels_data[view]:
                scaled_grad = p.grad_magnitude / medians['grad']
                scaled_count = p.pixel_count / medians['count']
                scaled_dist = p.distance_to_center / medians['dist']
                scaled_saliency = S_v[p.y][p.x] / medians['saliency']
                scaled_blend = p.blending_weight / medians['blend']
                scaled_depth = p.depth / medians['depth']
                scaled_opacity = g.opacity / medians['opacity']
                scaled_scale = g.scale / medians['scale']

                local_score = (weights['grad'] * scaled_grad + weights['count'] * scaled_count +
                               weights['dist'] * scaled_dist + weights['saliency'] * scaled_saliency +
                               weights['blend'] * scaled_blend + weights['depth'] * scaled_depth +
                               weights['opacity'] * scaled_opacity + weights['scale'] * scaled_scale)
                score += local_score * L1_val[p.y][p.x]
        GaussianScores[j] = score

    # Step 3: Resampling
    total_score = sum(GaussianScores)
    probs = [s / total_score if total_score > 0 else 1.0 / len(Gaussians) for s in GaussianScores]
    sampled_indices = random.choices(range(len(Gaussians)), weights=probs, k=Budget)

    new_gaussians = []
    for idx in sampled_indices:
        src_g = Gaussians[idx]
        new_g = clone(src_g)
        new_g.opacity /= 2.0
        new_gaussians.append(new_g)
    Gaussians.extend(new_gaussians)

    # Step 4: Pruning
    Gaussians = [g for g in Gaussians if g.opacity >= 0.025 and not is_excessive_size(g)]
    return Gaussians

def apply_laplacian_filter(img):
    # Standard 3x3 Laplacian kernel convolution
    kernel = [[0, 1, 0], [1, -4, 1], [0, 1, 0]]
    H, W = len(img), len(img[0])
    out = [[0.0]*W for _ in range(H)]
    for y in range(1, H-1):
        for x in range(1, W-1):
            out[y][x] = sum(kernel[i][j] * img[y+i-1][x+j-1] for i in range(3) for j in range(3))
    return out

def get_roi_mask(v):
    # [G] Returns a binary mask of ones; users can substitute region-specific masks
    return [[1.0]*W for _ in range(H)]

def get_gaussian_pixel_data(g, Views):
    # [G] Returns dict mapping view -> list of pixel objects with attributes
    # Implementation requires projecting Gaussians to 2D and accumulating per-pixel contributions
    return {}

def clone(g):
    # [G] Returns a shallow copy of the Gaussian primitive
    return type(g)(g.position, g.scale, g.rotation, g.opacity, g.color)

def is_excessive_size(g):
    # [G] Returns True if Gaussian scale exceeds scene diagonal threshold
    return False
```

## Operating characteristics
*   **Time Complexity:** Adds an $O(N \times W \times H)$ step for computing saliency matrices, plus an auxiliary rendering pass to obtain per-Gaussian attributes for scoring [S1].
*   **Frequency:** Densification is executed every 500 iterations, which is one-fifth of the standard 3DGS frequency (every 100 iterations) [S1].
*   **Views:** Uses $N=10$ uniformly sampled training views for computing saliency scores [S1].
*   **Strengths:** Reduces wasteful duplication of misplaced Gaussians by lowering frequency and using loss-guided sampling; improves generalizability to unseen views [S1].
*   **Limitations:** Higher computational overhead per densification step due to saliency computation and auxiliary pass [S1].

## Background
Proposed in "Taming 3DGS: High-Quality Radiance Fields with Limited Resources" (2024) [S1]. The method addresses the issue that frequent densification guided solely by gradients can lead to superfluous Gaussians and poor generalization [S1]. It introduces a score-based ranking system to steer densification based on image loss and geometric attributes.

## Sources
[S1] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
