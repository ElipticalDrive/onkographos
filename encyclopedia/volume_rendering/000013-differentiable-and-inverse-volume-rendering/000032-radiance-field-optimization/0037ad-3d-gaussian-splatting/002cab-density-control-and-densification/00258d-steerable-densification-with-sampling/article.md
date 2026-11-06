# Steerable Densification with Sampling

Also called: Score-based ranking densification

## Overview
Steerable Densification with Sampling is a density control method for 3D Gaussian Splatting that uses image loss and per-Gaussian attributes to rank primitives for resampling [S1]. It differs from standard splitting by decoupling the decision to densify from immediate gradient thresholds, instead using a score-based sampling mechanism executed at a significantly reduced frequency to prevent the duplication of misplaced primitives [S1].

## How it works
**Inputs:**
*   Set of $M$ Gaussians $\mathcal{G} = \{g_j\}_{j=1}^M$.
*   Set of $N$ training views $V = \{v_i\}_{i=1}^N$.
*   Rendered views $R = \{r_i\}_{i=1}^N$.
*   Budget $B$: target number of Gaussians to add.
*   Hyperparameters: $\lambda_1=0.5, \lambda_2=0.5$ (saliency weights); scoring weights $w_{\nabla}=50, w_{c}=0.1, w_{D}=50, w_{S}=10, w_{B}=50, w_{z}=5, w_{o}=100, w_{s}=25$ [S1].

**Outputs:**
*   Updated set $\mathcal{G}'$ with $M+B$ Gaussians.

**Data Structures:**
*   **Saliency Matrix $S_v$**: 2D array of shape $(H, W)$ for each view $v$.
*   **Score Vector $S_G$**: 1D array of shape $(M,)$, where $S_{g_j}$ is the score for Gaussian $g_j$.

**Steps:**

1.  **Compute Per-View Saliency:** For each view $v_i \in V$:
    *   Compute L1 loss map $L_1(v_i, r_i)$ between input and rendered view.
    *   Compute Laplacian filter map $E(v_i)$ on the rendered view.
    *   Define binary ROI mask $\mathbb{1}_{ROI}$ (all ones if no ROI specified).
    *   Calculate saliency matrix:
        $$ S_{v_i} = \mathbb{1}_{ROI} \odot (\lambda_1 L_1(v_i, r_i) + \lambda_2 E(v_i)) $$
        [S1]

2.  **Compute Gaussian Scores:** For each Gaussian $g_j \in \mathcal{G}$:
    *   Initialize score $S_{g_j} = 0$.
    *   For each view $v_i \in V$:
        *   Retrieve per-Gaussian attributes:
            *   $\nabla_{g_j}$: Positional gradient magnitude.
            *   $c_{g_j}$: Number of pixels covered by $g_j$ in view $i$.
            *   $\mathbb{1}_{g_j}$: Binary mask of pixels covered by $g_j$.
            *   $D_{g_j}$: Distance of each covered pixel to the Gaussian center.
            *   $S_{v_i}$: Saliency value at each covered pixel.
            *   $B_{g_j}$: Blending weight of $g_j$ at each covered pixel.
            *   $z_{g_j}$: Depth of $g_j$ in view $i$.
            *   $o_{g_j}$: Opacity of $g_j$.
            *   $s_{g_j}$: Scale of $g_j$.
        *   Apply median scaling to each attribute to remove outliers.
        *   Multiply scaled attributes by the photometric loss of view $i$.
        *   Accumulate weighted sum into $S_{g_j}$:
            $$ S_{g_j} \leftarrow S_{g_j} + \sum_{\text{pixels } p \in \text{covered}} \left( w_{\nabla}\nabla_{g_j} + w_{c}c_{g_j} + w_{D}D_{g_j} + w_{S}S_{v_i} + w_{B}B_{g_j} + w_{z}z_{g_j} + w_{o}o_{g_j} + w_{s}s_{g_j} \right) $$
            [S1]
    *   Normalize $S_G$ if necessary for sampling probabilities.

3.  **Resampling:**
    *   Sample $B$ indices from $\mathcal{G}$ with replacement, using $S_G$ as sampling weights.
    *   For each sampled Gaussian $g_{sample}$:
        *   Clone $g_{sample}$ to create a new Gaussian $g_{new}$.
        *   Perturb $g_{new}$'s position slightly (standard 3DGS perturbation) [S1, implied by "resampling"].
        *   Add $g_{new}$ to $\mathcal{G}$.

**Parameters:**
*   Densification frequency: Every 500 iterations (1/5th of standard 3DGS) [S1].
*   Number of views for saliency: $N=10$ uniformly sampled training views [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def steerable_densification(G, V, R, B, lambda1=0.5, lambda2=0.5):
    # Step 1: Compute Saliency Matrices
    S_v = {}
    for v in V:
        L1_map = compute_L1_loss(v.input, v.rendered)
        E_map = compute_laplacian_filter(v.rendered)
        # Assuming ROI is all ones if not specified
        ROI_mask = np.ones_like(L1_map) 
        S_v[v] = ROI_mask * (lambda1 * L1_map + lambda2 * E_map)
        
    # Step 2: Compute Gaussian Scores
    S_G = np.zeros(len(G))
    
    # Weights
    w = {
        'grad': 50, 'count': 0.1, 'dist': 50, 'saliency': 10,
        'blend': 50, 'depth': 5, 'opacity': 100, 'scale': 25
    }
    
    for j, g in enumerate(G):
        score = 0.0
        for v in V:
            # Get attributes for this Gaussian in this view
            # These require an auxiliary render pass or storing during forward pass
            grad_mag = g.gradient_magnitude
            pixel_count = g.pixel_count_in_view(v)
            pixel_mask = g.pixel_mask_in_view(v)
            dist_map = g.distance_map_in_view(v)
            blend_weights = g.blending_weights_in_view(v)
            depth = g.depth_in_view(v)
            opacity = g.opacity
            scale = g.scale
            
            # Median scaling (simplified: assume pre-scaled or median computed globally)
            # In practice, median is computed over all Gaussians for each attribute
            # Here we assume attributes are already scaled or we compute median on the fly
            # Note: The source says "restrict each parameter's range using median scaling"
            # This implies a global median calculation step before accumulation.
            
            # Accumulate score based on covered pixels
            # The formula sums over pixels, weighted by the attributes
            # Simplified: sum of weighted attributes for all covered pixels
            # Note: c_g is count, so it's a scalar. Others are per-pixel or scalar.
            # The source formula is ambiguous on whether c_g is summed per pixel or once.
            # Given c_g is "number of pixels", it is likely a scalar term added once per view.
            
            # Scalar terms
            score += w['grad'] * grad_mag
            score += w['count'] * pixel_count
            score += w['depth'] * depth
            score += w['opacity'] * opacity
            score += w['scale'] * scale
            
            # Per-pixel terms (sum over covered pixels)
            if pixel_mask.any():
                # Saliency, Distance, Blend are per-pixel
                # Note: The source lists D_g, S_v, B_g as matrices/attributes
                # We sum the weighted values for the covered pixels
                score += np.sum(w['dist'] * dist_map[pixel_mask])
                score += np.sum(w['saliency'] * S_v[v][pixel_mask])
                score += np.sum(w['blend'] * blend_weights[pixel_mask])
                
        S_G[j] = score
        
    # Step 3: Resampling
    # Sample B indices with replacement based on S_G
    indices = np.random.choice(len(G), size=B, replace=True, p=S_G / np.sum(S_G))
    
    new_Gaussians = []
    for idx in indices:
        g_orig = G[idx]
        g_new = clone_gaussian(g_orig)
        g_new.position += perturb(g_new.position) # Standard perturbation
        new_Gaussians.append(g_new)
        
    return G + new_Gaussians
```

## Operating characteristics
*   **Complexity:** Adds $O(N \times H \times W)$ for saliency matrix computation plus an auxiliary rendering pass to obtain per-Gaussian attributes for $S_G$ [S1].
*   **Performance:** Reduces the number of superfluous Gaussians compared to standard gradient-based splitting [S1].
*   **Strengths:** Interpretable; integrates image loss directly; reduces wasteful duplication of misplaced primitives by lowering frequency [S1].
*   **Limitations:** Requires additional rendering passes; sensitive to hyperparameter weights [S1].

## Background
Proposed in "Taming 3DGS: High-Quality Radiance Fields with Limited Resources" (2024) [S1]. It addresses the issue of frequent densification causing repeated duplication of misplaced Gaussians by reducing frequency and using loss-guided scoring [S1].

## Sources
[S1] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
