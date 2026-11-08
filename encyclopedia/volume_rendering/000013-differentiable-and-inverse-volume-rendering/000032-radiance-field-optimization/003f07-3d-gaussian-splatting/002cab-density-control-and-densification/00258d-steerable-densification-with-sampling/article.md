# Steerable Densification with Sampling

Also called: Score-based ranking densification

## Overview
Steerable Densification with Sampling is a density control method for 3D Gaussian Splatting that replaces heuristic thresholding with a score-based ranking system to select candidates for resampling [S1]. It differs from standard 3DGS densification by incorporating image loss and per-pixel saliency directly into the selection probability, allowing for a significantly reduced densification frequency (every 500 iterations) while maintaining high quality [S1].

## How it works
**Inputs:**
*   Set of $N$ camera views $V = \{v_i\}_{i=1}^N$.
*   Set of $M$ fitted Gaussians $G = \{g_j\}_{j=1}^M$.
*   Set of $N$ rendered views $R = \{r_i\}_{i=1}^N$.
*   Budgeted target number of Gaussians to add $B$.
*   Hyperparameters: $\lambda_1=0.5, \lambda_2=0.5$ for saliency; weights for scoring function parameters.

**Outputs:**
*   Updated set of Gaussians $G'$ with $B$ new primitives added via resampling.

**Data Structures:**
*   **Saliency Matrix $S_v$**: A 2D matrix of size $H \times W$ (height $\times$ width) for each view $v$. Elements are floating-point scores.
*   **Score Vector $S_G$**: A 1D vector of size $M$. $S_{g_j}$ is the accumulated score for Gaussian $g_j$.

**Numbered Steps:**

1.  **Compute Per-View Saliency Matrices:**
    For each view $v \in V$:
    $$ S_v = \mathbb{1}_{ROI} \odot (\lambda_1 L_1(v, r_v) + \lambda_2 E(v)) $$
    Where:
    *   $\mathbb{1}_{ROI}$ is a binary mask (0 or 1) indicating regions of interest.
    *   $L_1(v, r_v)$ is the L1 loss between the rendered view and the ground truth.
    *   $E(v)$ is the Laplacian filter applied to the rendered view to detect high-frequency information.
    *   $\odot$ denotes element-wise multiplication.

2.  **Compute Gaussian Scores:**
    For each Gaussian $g \in G$, compute its score $S_g$ by summing a scoring function $F$ over all $N$ views:
    $$ S_g = \sum_{i=1}^N F(\nabla_g, c_{gi}, \mathbb{1}_{gi}, D_{gi}, S_{vi}, B_{gi}, z_{gi}, o_g, s_g) $$
    The global score vector is $S_G = [S_{g_1}, \dots, S_{g_M}]^T$.

3.  **Resample Gaussians:**
    Randomly select $B$ primitives from the set $G$ using $S_G$ as sampling weights. The selected Gaussians are cloned or split (implementation detail of the underlying primitive operation, typically cloning at the same position with halved opacity/energy as per parent principles, though the source emphasizes the *selection* mechanism) to form the new set.

**Scoring Function $F$:**
The function $F$ aggregates per-Gaussian and per-pixel metrics. Each parameter is median-scaled to remove outliers, multiplied by the photometric loss, and accumulated with specific weights [S1]:
$$ F = w_{\nabla} \cdot \text{scale}(\nabla_g) + w_{c} \cdot \text{scale}(c_{gi}) + w_{D} \cdot \text{scale}(D_{gi}) + w_{S} \cdot \text{scale}(S_{vi}) + w_{B} \cdot \text{scale}(B_{gi}) + w_{z} \cdot \text{scale}(z_{gi}) + w_{o} \cdot \text{scale}(o_g) + w_{s} \cdot \text{scale}(s_g) $$

**Parameters and Weights:**
*   $\nabla_g$: Magnitude of the positional gradient (Weight: 50). Acts as a 3D discontinuity detector [S1].
*   $c_{gi}$: Number of pixels covered by $g$ in view $i$ (Weight: 0.1) [S1].
*   $\mathbb{1}_{gi}$: Binary matrix indicating pixels covered by $g$ (Used in context of $c_{gi}$ or $D_{gi}$) [S1].
*   $D_{gi}$: Matrix of distances from each pixel to the center of $g$ (Weight: 50) [S1].
*   $S_{vi}$: Saliency value at the pixel location in view $i$ (Weight: 10) [S1].
*   $B_{gi}$: Blending weight of $g$ for each pixel (Weight: 50) [S1].
*   $z_{gi}$: Depth of $g$ in view $i$ (Weight: 5) [S1].
*   $o_g$: Opacity of $g$ (Weight: 100) [S1].
*   $s_g$: Scale of $g$ (Weight: 25) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def steerable_densification(Gaussians, Views, Rendered_Views, Budget_B):
    """
    Performs score-based densification.
    Gaussians: List of Gaussian objects with attributes:
               pos, scale, opacity, rotation, features
               Also requires accumulated positional gradient magnitude |grad_p|
    Views: List of camera view objects
    Rendered_Views: List of rendered image tensors (H x W)
    Budget_B: Integer, number of new Gaussians to add
    """
    M = len(Gaussians)
    N = len(Views)
    
    # Step 1: Compute Saliency Matrices
    # Initialize list of saliency matrices for each view
    saliency_matrices = []
    
    for i in range(N):
        v = Views[i]
        r_v = Rendered_Views[i]
        
        # Compute L1 Loss
        # Note: Ground truth image must be available. 
        # Assuming a function get_ground_truth(v) exists or is passed.
        gt_v = get_ground_truth(v) 
        l1_loss = abs(r_v - gt_v)
        
        # Compute Laplacian Filter E(v)
        # Standard 3x3 Laplacian kernel
        laplacian_kernel = [[0, 1, 0], [1, -4, 1], [0, 1, 0]]
        E_v = apply_convolution(r_v, laplacian_kernel)
        
        # Combine with ROI mask (assuming get_roi_mask(v) returns binary mask)
        roi_mask = get_roi_mask(v)
        
        # S_v = 1_ROI * (0.5 * L1 + 0.5 * E)
        lambda1 = 0.5
        lambda2 = 0.5
        S_v = roi_mask * (lambda1 * l1_loss + lambda2 * E_v)
        
        saliency_matrices.append(S_v)

    # Step 2: Compute Gaussian Scores
    scores = [0.0] * M
    
    # Pre-calculate median scales for robustness
    # Collect all values for each parameter across all Gaussians and views
    # to compute medians for scaling.
    
    # Helper to median-scale a value
    def median_scale(value, all_values):
        median_val = median(all_values)
        if median_val == 0:
            return 0
        return value / median_val

    # We need to gather all raw values first to compute medians
    # This requires two passes or storing raw values. 
    # For pseudocode clarity, we assume we have lists of raw values.
    
    raw_grads = [abs(g.grad_p) for g in Gaussians]
    raw_opacities = [g.opacity for g in Gaussians]
    raw_scales = [g.scale_norm for g in Gaussians] # Norm of scale vector
    
    # For view-dependent params, we need flattened lists
    raw_c = []
    raw_D = []
    raw_B = []
    raw_z = []
    raw_S = []
    
    for i in range(N):
        S_v = saliency_matrices[i]
        for j, g in enumerate(Gaussians):
            # Project Gaussian to view i to get pixel coverage
            # This is the "auxiliary rendering pass" mentioned in sources
            # Returns: coverage_count, distance_matrix, blending_weights, depth
            c_ij, D_ij, B_ij, z_ij = project_gaussian_to_view(g, Views[i])
            
            raw_c.append(c_ij)
            raw_D.append(D_ij) # Flattened
            raw_B.append(B_ij) # Flattened
            raw_z.append(z_ij) # Flattened
            
            # S_v is HxW. We need to sample S_v at the pixel locations covered by g
            # Assuming get_pixel_indices returns list of (u, v)
            pixel_indices = get_pixel_indices_for_gaussian(g, Views[i])
            S_values = [S_v[u][v] for u, v in pixel_indices]
            raw_S.extend(S_values)

    # Compute Medians
    med_grad = median(raw_grads)
    med_c = median(raw_c)
    med_D = median(raw_D)
    med_B = median(raw_B)
    med_z = median(raw_z)
    med_S = median(raw_S)
    med_o = median(raw_opacities)
    med_s = median(raw_scales)

    # Weights
    w_grad = 50
    w_c = 0.1
    w_D = 50
    w_S = 10
    w_B = 50
    w_z = 5
    w_o = 100
    w_s = 25

    # Compute scores for each Gaussian
    for j, g in enumerate(Gaussians):
        score = 0.0
        
        # Global attributes
        grad_scaled = median_scale(abs(g.grad_p), raw_grads)
        o_scaled = median_scale(g.opacity, raw_opacities)
        s_scaled = median_scale(g.scale_norm, raw_scales)
        
        # Accumulate view-dependent scores
        view_score_sum = 0.0
        
        for i in range(N):
            # Re-project or retrieve cached projection data
            c_ij, D_ij, B_ij, z_ij = project_gaussian_to_view(g, Views[i])
            pixel_indices = get_pixel_indices_for_gaussian(g, Views[i])
            S_values = [saliency_matrices[i][u][v] for u, v in pixel_indices]
            
            # Scale view-dependent values
            # Note: D_ij, B_ij, z_ij, S_values are arrays/matrices
            # We aggregate them (e.g., sum or mean) before scaling? 
            # The source says "summing a scoring function F over all N views".
            # F takes the parameters. It implies F is computed per view and summed.
            # The scaling is done on the parameters.
            
            # To be precise, we scale the raw values then aggregate? 
            # Or aggregate then scale? Source: "restrict each parameter's range using median scaling... followed by multiplication... then accumulated"
            # This implies scaling individual values.
            
            # For array parameters, we might scale the whole array by the global median
            # or compute a scalar metric (like mean distance) then scale.
            # Given the weights are scalars, F likely outputs a scalar per view.
            # Let's assume we aggregate the array parameters into a scalar metric per view 
            # (e.g., mean distance, sum of blending weights) before scaling, 
            # OR scale the arrays and then aggregate.
            # The source lists D_gi, B_gi, z_gi as matrices/vectors.
            # F( ..., D_gi, ... ) suggests D_gi is an input to F.
            # If F sums them, it's ambiguous. 
            # However, standard practice in such scoring is to reduce spatial dimensions.
            # Let's assume we take the mean of the spatial values for the score contribution.
            
            mean_D = mean(D_ij)
            mean_B = mean(B_ij)
            mean_z = mean(z_ij)
            mean_S = mean(S_values)
            
            # Scale
            s_D = median_scale(mean_D, raw_D)
            s_B = median_scale(mean_B, raw_B)
            s_z = median_scale(mean_z, raw_z)
            s_S = median_scale(mean_S, raw_S)
            s_c = median_scale(c_ij, raw_c)
            
            # Compute F for this view
            # Note: The source says "multiplication with i's photometric loss".
            # This likely means the entire F output for view i is multiplied by the loss of view i.
            # Or the parameters are multiplied? "multiplication with i's photometric loss before accumulation"
            # Usually, loss guides the importance. Let's multiply the final F_i by Loss_i.
            
            F_i = (w_grad * grad_scaled + 
                   w_c * s_c + 
                   w_D * s_D + 
                   w_S * s_S + 
                   w_B * s_B + 
                   w_z * s_z + 
                   w_o * o_scaled + 
                   w_s * s_scaled)
            
            # Multiply by photometric loss of view i
            # L1 loss of view i (scalar, e.g., mean L1)
            loss_i = mean(l1_loss_view_i) # From Step 1
            view_score_sum += F_i * loss_i
            
        scores[j] = view_score_sum

    # Step 3: Resample
    # Select B Gaussians with probability proportional to scores
    selected_indices = weighted_random_sample(Gaussians, scores, Budget_B)
    
    # Add new Gaussians based on selected ones
    # Implementation of cloning/splitting is inherited from parent principles
    new_gaussians = []
    for idx in selected_indices:
        g = Gaussians[idx]
        # Clone g
        g_new = clone_gaussian(g)
        # Halve opacity/energy as per parent principles
        g_new.opacity /= 2.0
        new_gaussians.append(g_new)
        
    return Gaussians + new_gaussians
```

**Edge Cases and Pitfalls:**
*   **Zero Median:** If the median of a parameter is zero, scaling is undefined. The pseudocode handles this by returning 0.
*   **High Opacity:** The source notes that high-opacity Gaussians (opacities > 1) can boost modeling of opaque surfaces and should be handled, potentially by conversion to clamped Gaussians [S1].
*   **Frequency:** Densification occurs every 500 iterations. Frequent densification with loss guidance can cause repeated duplication of misplaced Gaussians [S1].

## Operating characteristics
*   **Time Complexity:** Adds an extra $O(N \times \text{width} \times \text{height})$ step for computing all $N$ saliency matrices $S_v$, plus an auxiliary rendering pass to obtain $S_G$ [S1].
*   **Memory Complexity:** Requires storing $N$ saliency matrices of size $H \times W$ and intermediate projection data for all Gaussians across all views during the scoring phase.
*   **Performance:** Allows for high-quality radiance fields with limited resources (fewer Gaussians) by steering densification towards salient areas [S1].
*   **Strengths:** Interpretable, customizable sampling; reduces wasteful Gaussians compared to pure gradient-based methods; handles high-frequency details via Laplacian filter.
*   **Limitations:** Higher computational overhead per densification step due to saliency computation and auxiliary rendering pass; requires careful tuning of weights.

## Background
Steerable Densification with Sampling was introduced in "Taming 3DGS: High-Quality Radiance Fields with Limited Resources" (2024) [S1]. It addresses the issue of wasteful densification in standard 3D Gaussian Splatting by incorporating image loss and saliency into the selection process, allowing for a reduced densification frequency.

## Sources
[S1] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
