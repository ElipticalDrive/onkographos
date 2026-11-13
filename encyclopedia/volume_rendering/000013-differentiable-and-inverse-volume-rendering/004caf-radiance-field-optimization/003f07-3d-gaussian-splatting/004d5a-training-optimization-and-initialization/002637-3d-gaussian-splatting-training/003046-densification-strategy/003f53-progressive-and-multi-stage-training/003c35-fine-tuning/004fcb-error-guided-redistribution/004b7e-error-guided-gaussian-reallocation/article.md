# Error-Guided Gaussian Reallocation

Also called: EGR

## Overview
Error-Guided Gaussian Reallocation (EGR) is a training-stage module that redistributes a fixed budget of Gaussian primitives toward regions with persistent coarse-scale reconstruction errors, without increasing the total number of Gaussians [S1]. It refines post-densification Gaussian allocation by moving low-contribution (LC) Gaussians to high-error (HE) regions within a fixed Gaussian set, using contribution- and error-aware selection [S1]. Unlike methods using explicit rollback or separate re-correction modules, EGR uses adaptive relocation count, parameter reset, and continued optimization as its stabilization strategy [S1].

## How it works
EGR operates after densification is complete, once the Gaussian count is fixed [S1]. It identifies pairs of Gaussians: one with low contribution (LC) and one with high error (HE), and relocates the LC Gaussian to the position of the HE Gaussian [S1].

### Inputs and Outputs
- **Inputs**: The current set of $N$ Gaussians, their parameters (position, scale, orientation, opacity, SH coefficients, ACO offsets), optimizer states, and the training dataset (images and camera poses).
- **Outputs**: The same set of $N$ Gaussians with updated positions, scales, orientations, colors, opacities, and reset optimizer states for the relocated Gaussians.

### Data Structures
The Gaussian set is stored as an array of $N$ elements. Each Gaussian $i$ contains:
- `pos`: 3D position vector $(x, y, z)$.
- `scale`: 3D scale vector $(s_x, s_y, s_z)$.
- `rot`: Quaternion or rotation matrix representing orientation.
- `opacity`: Scalar opacity value $\alpha_i$.
- `sh_coeffs`: Spherical Harmonics coefficients.
- `aco_offset`: Adaptive-Scale Color Offset (if using ACO).
- `optimizer_state`: State variables for the optimizer (e.g., momentum, variance).

### Steps
1. **Compute Contribution Metrics**:
   For each view $v$ in the training set $V$, compute the normalized blending weight $\tilde{w}_i(v)$ for each Gaussian $i$:
   $$
   \tilde{w}_i(v) = \frac{w_i(v)}{\sum_{i'} w_{i'}(v) + \epsilon}
   $$
   where $w_i(v)$ is the blending weight of Gaussian $i$ in view $v$, $i'$ indexes projected Gaussians in view $v$, and $\epsilon$ is a small positive constant [S1].

   Compute the mean contribution $C_i^{mean}$ and maximum contribution $C_i^{max}$ for each Gaussian:
   $$
   C_i^{mean} = \frac{1}{|V|} \sum_{v} \tilde{w}_i(v), \quad C_i^{max} = \max_{v} \tilde{w}_i(v)
   $$
   where $|V|$ is the number of views used for contribution measurement [S1].

2. **Compute Gradient Magnitude**:
   Compute the $L_2$ loss for each view $v$:
   $$
   L_2(v) = \frac{1}{3H_v W_v} \sum_{h,w,\ell} (\hat{I}_{h,w,\ell}(v) - I_{h,w,\ell}(v))^2
   $$
   where $H_v, W_v$ are image height and width for view $v$, $\hat{I}$ is the rendered image, and $I$ is the ground truth image [S1].

   Compute the normalized mean gradient magnitude $G_i^{mean}$ for each Gaussian:
   $$
   G_i^{mean} = \frac{1}{|V|} \sum_v \frac{\nabla_{\theta_i} L_2(v)}{\sum_{i'} \nabla_{\theta'_i} L_2(v) + \epsilon}
   $$
   where $\nabla_{\theta_i} L_2(v)$ is the parameter gradient of Gaussian $i$ from this loss, and the denominator normalizes gradient magnitude within each view [S1].

3. **Robust Z-Score Normalization**:
   Apply robust z-score normalization to reduce sensitivity to extreme values for each measure $x \in \{C_i^{mean}, C_i^{max}, G_i^{mean}\}$:
   $$
   z(x) = \frac{x - \text{median}(x)}{\text{median}(|x - \text{median}(x)|) + \epsilon}
   $$
   where $\text{median}(x)$ is the median of measure $x$ over all Gaussians [S1].

4. **Compute LC and HE Scores**:
   Compute the LC score $s_i^{LC}$ for each Gaussian. High score indicates low contribution:
   $$
   s_i^{LC} = \exp(-z(C_i^{mean}) - z(C_i^{max}) - z(G_i^{mean}))
   $$
   Gaussians with small mean contribution, small maximum contribution, and small gradient magnitude receive high scores [S1].

   Compute the HE score $s_i^{HE}$ for each Gaussian. High score indicates high error-weighted contribution. First, compute the error-weighted contribution $C_i^{HE}$ (not explicitly defined in the provided text, but implied as the basis for ranking). Then:
   $$
   s_i^{HE} = \frac{1}{N-1} \sum_{j=1}^N \mathbb{1}[C_j^{HE} < C_i^{HE}]
   $$
   where $N$ is the number of Gaussians and $\mathbb{1}[\cdot]$ is the indicator function. This is a percentile score based on the rank of $C_i^{HE}$ [S1].

5. **Select Pairs**:
   Select $M_{eff}^{LC}$ Gaussians with the highest $s_i^{LC}$ scores and $M_{eff}^{HE}$ Gaussians with the highest $s_i^{HE}$ scores. The number of pairs $M$ is determined by the smaller effective sample size:
   $$
   M = \min(M_{eff}^{LC}, M_{eff}^{HE})
   $$
   where $M_{eff}^q = \frac{1}{\sum_{j=1}^N (p_j^q)^2}$ for $q \in \{LC, HE\}$, and $p_j^q$ is the probability associated with the score distribution [S1].

6. **Relocate Gaussians**:
   For each of the $M$ selected pairs:
   - Relocate the LC Gaussian to the location of its paired HE Gaussian.
   - Copy the HE Gaussian’s scale, orientation, and base color to the relocated LC Gaussian.
   - Reset the relocated Gaussian’s higher-order SH coefficients and ACO offsets to zero.
   - Clear the optimizer state of the relocated Gaussian [S1].

7. **Rescale Opacities**:
   After relocation, rescale opacities of the paired Gaussians to preserve original transmittance according to the opacity split of 3DGS-MCMC [21] [S1].

8. **Resume Optimization**:
   Resume standard gradient-based optimization immediately after relocation, allowing the paired Gaussians to independently optimize their appearance parameters [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def error_guided_reallocation(gaussians, training_views, optimizer, iteration):
    """
    Perform Error-Guided Gaussian Reallocation.
    
    Args:
        gaussians: List of Gaussian objects with attributes: pos, scale, rot, opacity, sh_coeffs, aco_offset, optimizer_state.
        training_views: List of view objects with attributes: image, camera, h, w.
        optimizer: Optimizer object with method clear_state(gaussian).
        iteration: Current training iteration.
        
    Returns:
        Updated list of Gaussians.
    """
    N = len(gaussians)
    V = len(training_views)
    epsilon = 1e-8
    
    # Step 1: Compute Contribution Metrics
    w_mean = [0.0] * N
    w_max = [0.0] * N
    
    for v in training_views:
        # Compute blending weights for all Gaussians in view v
        # This requires rendering or accessing pre-computed weights
        # Assume get_blending_weights(v) returns a list of weights for each Gaussian
        weights = get_blending_weights(v, gaussians)
        total_weight = sum(weights) + epsilon
        
        for i in range(N):
            normalized_w = weights[i] / total_weight
            w_mean[i] += normalized_w
            if normalized_w > w_max[i]:
                w_max[i] = normalized_w
                
    # Average over views
    for i in range(N):
        w_mean[i] /= V
        # w_max[i] is already the max
        
    # Step 2: Compute Gradient Magnitude
    # Assume get_gradients(v, gaussians) returns list of gradient magnitudes for each Gaussian
    # And get_loss(v) returns L2 loss
    g_mean = [0.0] * N
    
    for v in training_views:
        gradients = get_gradients(v, gaussians)
        total_grad = sum(gradients) + epsilon
        
        for i in range(N):
            normalized_grad = gradients[i] / total_grad
            g_mean[i] += normalized_grad
            
    for i in range(N):
        g_mean[i] /= V
        
    # Step 3: Robust Z-Score Normalization
    def robust_z_score(values):
        median_val = median(values)
        mad = median([abs(x - median_val) for x in values])
        return [(x - median_val) / (mad + epsilon) for x in values]
    
    z_w_mean = robust_z_score(w_mean)
    z_w_max = robust_z_score(w_max)
    z_g_mean = robust_z_score(g_mean)
    
    # Step 4: Compute LC and HE Scores
    # LC Score: High score for low contribution
    lc_scores = []
    for i in range(N):
        score = exp(-z_w_mean[i] - z_w_max[i] - z_g_mean[i])
        lc_scores.append(score)
        
    # HE Score: High score for high error
    # Need to compute C_i^HE first. The text says "error-weighted contribution".
    # Assuming C_i^HE is computed similarly to w_mean but weighted by L2 loss?
    # The text says "EGR combines reconstruction error with normalized Gaussian contribution".
    # Let's assume C_i^HE is available from get_error_weighted_contribution()
    c_he = get_error_weighted_contribution(gaussians, training_views)
    
    he_scores = []
    for i in range(N):
        count = sum(1 for j in range(N) if c_he[j] < c_he[i])
        score = count / (N - 1)
        he_scores.append(score)
        
    # Step 5: Select Pairs
    # Compute effective sample sizes
    # p_j^q is probability associated with score distribution. 
    # Assuming softmax or similar normalization to get probabilities
    def get_probabilities(scores):
        max_score = max(scores)
        exp_scores = [exp(s - max_score) for s in scores]
        sum_exp = sum(exp_scores)
        return [e / sum_exp for e in exp_scores]
    
    p_lc = get_probabilities(lc_scores)
    p_he = get_probabilities(he_scores)
    
    m_eff_lc = 1.0 / sum([p**2 for p in p_lc])
    m_eff_he = 1.0 / sum([p**2 for p in p_he])
    
    M = min(m_eff_lc, m_eff_he)
    M = int(M) # Round down
    
    # Select top M LC and top M HE
    # Sort indices by score descending
    lc_indices = sorted(range(N), key=lambda i: lc_scores[i], reverse=True)[:M]
    he_indices = sorted(range(N), key=lambda i: he_scores[i], reverse=True)[:M]
    
    # Pair them up
    pairs = list(zip(lc_indices, he_indices))
    
    # Step 6: Relocate Gaussians
    for lc_idx, he_idx in pairs:
        lc_g = gaussians[lc_idx]
        he_g = gaussians[he_idx]
        
        # Relocate LC to HE position
        lc_g.pos = he_g.pos.copy()
        
        # Copy scale, orientation, base color
        lc_g.scale = he_g.scale.copy()
        lc_g.rot = he_g.rot.copy()
        lc_g.base_color = he_g.base_color.copy() # Assuming base_color is stored
        
        # Reset higher-order SH coefficients and ACO offsets to zero
        lc_g.sh_coeffs = reset_sh_coeffs(lc_g.sh_coeffs) # Zero out higher-order terms
        lc_g.aco_offset = 0.0
        
        # Clear optimizer state
        optimizer.clear_state(lc_g)
        
    # Step 7: Rescale Opacities
    # Preserve original transmittance according to opacity split of 3DGS-MCMC [21]
    for lc_idx, he_idx in pairs:
        lc_g = gaussians[lc_idx]
        he_g = gaussians[he_idx]
        
        # Opacity rescaling logic from 3DGS-MCMC
        # This is a placeholder for the specific formula from [21]
        # Assuming a function that takes two opacities and returns new opacities
        new_lc_opacity, new_he_opacity = rescale_opacities_for_transmittance(lc_g.opacity, he_g.opacity)
        
        lc_g.opacity = new_lc_opacity
        he_g.opacity = new_he_opacity
        
    return gaussians
```

### Parameters
- **Fixed primitive budget**: The total number of Gaussians remains constant during reallocation; no new Gaussians are added [S1].
- **Reallocation interval**: 1000 iterations [S1].
- **Reallocation window**: Iterations 16,000 to 25,000 [S1].
- **Number of reallocation rounds**: 10 [S1].

### Edge Cases and Pitfalls
- **Degenerate input**: If densification is still active, EGR should not run. It is explicitly disabled during densification (iterations 500-15,000) and only runs after the Gaussian count is fixed [S1].
- **Precision**: The use of robust z-score normalization helps mitigate issues with extreme values in contribution or gradient metrics [S1].
- **Boundary conditions**: The denominator in normalization formulas includes $\epsilon$ to prevent division by zero [S1].

## Operating characteristics
- **Time Complexity**: The method involves computing contributions, gradients, and scores for all Gaussians and views, which is $O(N \cdot |V|)$ per round. The selection and relocation steps are $O(N \log N)$ due to sorting. With 10 rounds, the total overhead is significant but bounded.
- **Memory Complexity**: No additional memory is required beyond storing the current Gaussian parameters and optimizer states. The method operates in-place.
- **Measured Performance**: Experiments on MS-GS with three real-scene benchmarks demonstrate improved multi-scale rendering quality, particularly at coarse scales [S1].
- **Scaling**: The method scales with the number of Gaussians and views. The adaptive relocation count $M$ helps limit the perturbation introduced in each round [S1].
- **Strengths**: Improves coarse-scale rendering quality by redirecting primitives to high-error regions. Preserves Gaussians that support coarse-scale rendering. Uses adaptive relocation count, parameter reset, and continued optimization for stabilization [S1].
- **Limitations**: Only operates after densification. Requires recomputation of statistics every round. May not address fine-scale errors as effectively as densification methods.
- **When to use**: When coarse-scale rendering quality is poor and densification has already been completed.
- **When not to use**: During densification phase (iterations 500-15,000).

## Background
Error-Guided Gaussian Reallocation (EGR) was introduced in the paper "Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting" [S1]. It is part of the ACO-EGR framework, which also includes Adaptive-Scale Color Offset (ACO) to address scale-dependent color mismatch [S1]. The method was developed to improve multi-scale rendering quality in 3D Gaussian Splatting by redistributing a fixed budget of Gaussians toward regions with persistent reconstruction errors [S1].

## Sources
[S1] Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting, https://doi.org/10.3390/math14152713, fetched 2026-11-01
