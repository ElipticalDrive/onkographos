# Complexity-Aware Adaptive Density Control

Also called: Complexity-Aware Density Control, Complexity-Aware Level of Details, Complexity-Guided Relocation

## Overview
Complexity-Aware Adaptive Density Control is a training-time strategy for 3D Gaussian Splatting that replaces opacity-only heuristics with a composite score incorporating image complexity, scale, and opacity to guide Gaussian relocation and pruning [S1, unofficial]. It solves the problem where standard methods concentrate primitives in already-resolved high-opacity regions while neglecting unresolved areas, by using complexity as a surrogate for reconstruction error to bias sampling toward difficult regions [S1, unofficial]. It differs from its siblings by explicitly modeling the spatial footprint (scale) and local image entropy (complexity) to ensure uniform degradation across different Levels of Detail (LoD), whereas siblings like [Multi-View Density Control Criterion](5663-multi-view-density-control-criterion.md) rely on multi-view consistency or [Frequency-Aware Adaptive Splat Spawning](19904-frequency-aware-adaptive-splat-spawning.md) relies on frequency domain metrics [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Set of 3D Gaussians $G = \{g_j\}$, where each $g_j$ has mean $\mu_j$, opacity $\alpha_j$, scale $s(g_j)$, and rotation.
- Camera views $\Pi_i$ with associated image planes.
- Complexity map $C_i^{2D}(u)$ for each view, computed via delentropy [S1, unofficial].
- Training budget factor $\tau \in (0, 1]$, where $\tau=1$ is full capacity and $\tau<1$ is a reduced budget [S1, unofficial].
- Hyperparameters: EMA learning rate $\eta \in (0, 1]$, complexity dropout probability $\delta = 0.2$, scale weight $\lambda = 2$ [S1, unofficial].

**Outputs:**
- Updated complexity field $C_j$ for each Gaussian.
- Normalized complexity $\tilde{C}_j \in [0, 1]$.
- Relocation probability distribution $p(g_j)$ for densification.
- Importance score $S(g_j)$ for pruning/survival selection.

### Data Structures
- **Complexity Field:** A scalar value $C_j$ stored per Gaussian. Initialized to 0 or a small constant.
- **Normalized Complexity:** $\tilde{C}_j$ is computed per training step or periodically by min-max normalization of $C_j$ across all Gaussians: $\tilde{C}_j = \frac{C_j - \min(C)}{\max(C) - \min(C)}$ [S1, unofficial].

### Steps
1. **Complexity Update:** After rendering a view $\Pi_i$ at full capacity ($\tau=1$), project the mean $\mu_j$ of every contributing Gaussian into the image plane to obtain pixel coordinate $u = \pi_i(\mu_j)$ [S1, unofficial]. Only update $C_j$ if $\mu_j$ projects within the view's frustum [S1, unofficial]. Update using exponential moving average:
   $$ C_j \leftarrow (1 - \eta) C_j + \eta C_i^{2D}(u) $$
   where $C_i^{2D}(u)$ is the delentropy-based complexity value at pixel $u$ [S1, unofficial].

2. **Normalization:** Compute $\tilde{C}_j$ for all $j$ by min-max normalization of the current $C_j$ values across the entire set of Gaussians [S1, unofficial].

3. **Relocation Sampling (Densification):** When selecting a source Gaussian $g_j$ to spawn new primitives, sample $g_j$ with probability:
   $$ p(g_j) = \alpha_j \cdot \exp(\tilde{C}_j') $$
   where $\tilde{C}_j'$ is the complexity value after applying dropout [S1, unofficial]. With probability $\delta = 0.2$, set $\tilde{C}_j' = 0$ (complexity dropout) to prevent collapse onto fixed complex regions; otherwise $\tilde{C}_j' = \tilde{C}_j$ [S1, unofficial].

4. **Importance Scoring (Pruning/LoD):** For a given budget $\tau$, compute the importance score $S(g_j)$ to determine which Gaussians survive:
   $$ S(g_j) = \alpha_j + \lambda \cdot \frac{s(g_j)}{\exp(\tilde{C}_j)} $$
   where $s(g_j)$ is the max-normalized scale of the Gaussian and $\lambda = 2$ [S1, unofficial]. Gaussians with the highest scores are retained up to the budget limit [S1, unofficial].

5. **Variance Loss Minimization:** To ensure uniform degradation, compute the residual $\Delta_i(u)$ between the full-capacity error $E(I_i^{full})$ and the budgeted error $E(I_i^{\tau})$ at each pixel $u$ in the image domain $\Omega$ [S1, unofficial]. Minimize the spatial variance of this residual:
   $$ L_{var} = \text{Var}_{u \in \Omega} \Delta_i(u) $$
   The total training loss combines the standard photometric losses (L1 + D-SSIM) on both full and budgeted renders with this variance term [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def update_complexity(gaussians, view, eta):
    """
    Update the complexity field C_j for Gaussians contributing to the current view.
    """
    # Project Gaussian means to image plane
    # Returns list of (gaussian_index, pixel_u, pixel_v) for valid projections
    projections = project_means_to_image(gaussians, view.camera)
    
    # Get complexity map for the current view (computed via delentropy)
    complexity_map = compute_delentropy_complexity(view.image)
    
    for idx, u, v in projections:
        # Check if projection is within frustum/bounds
        if is_within_frustum(u, v, view):
            c_2d = complexity_map[v, u]
            # Exponential Moving Average update
            gaussians[idx].C = (1 - eta) * gaussians[idx].C + eta * c_2d

def normalize_complexity(gaussians):
    """
    Compute min-max normalized complexity C_tilde for all Gaussians.
    """
    C_values = np.array([g.C for g in gaussians])
    min_c = np.min(C_values)
    max_c = np.max(C_values)
    
    # Avoid division by zero if all complexities are identical
    if max_c == min_c:
        return np.zeros_like(C_values)
        
    C_tilde = (C_values - min_c) / (max_c - min_c)
    for i, g in enumerate(gaussians):
        g.C_tilde = C_tilde[i]

def sample_relocation_source(gaussians, delta=0.2):
    """
    Select a Gaussian for densification based on opacity and complexity.
    """
    n = len(gaussians)
    weights = np.zeros(n)
    
    for i, g in enumerate(gaussians):
        # Apply complexity dropout
        if np.random.rand() < delta:
            c_tilde_effective = 0.0
        else:
            c_tilde_effective = g.C_tilde
            
        # Probability proportional to alpha * exp(C_tilde)
        weights[i] = g.alpha * np.exp(c_tilde_effective)
        
    # Normalize to sum to 1
    weights_sum = np.sum(weights)
    if weights_sum == 0:
        return np.random.randint(n)
    probs = weights / weights_sum
    
    # Sample index
    return np.random.choice(n, p=probs)

def compute_importance_score(gaussians, lambda_scale=2.0):
    """
    Compute importance score S(g_j) for pruning/LoD selection.
    """
    scores = np.zeros(len(gaussians))
    
    # Pre-calculate max-normalized scale if not already done
    scales = np.array([g.scale for g in gaussians])
    max_scale = np.max(scales)
    if max_scale == 0:
        max_scale = 1.0
    normalized_scales = scales / max_scale
    
    for i, g in enumerate(gaussians):
        # S(g_j) = alpha_j + lambda * s(g_j) / exp(C_tilde_j)
        denom = np.exp(g.C_tilde)
        scores[i] = g.alpha + lambda_scale * (normalized_scales[i] / denom)
        
    return scores

def compute_variance_loss(full_render, budgeted_render, tau):
    """
    Compute L_var = Var(Delta_i(u)) where Delta = E(I_full) - E(I_tau).
    Assumes E(I) is the error map (e.g., L1 error) for the respective render.
    """
    # Compute error maps (assuming full_render and budgeted_render are error tensors)
    # If inputs are images, compute L1 error against ground truth first.
    # Here we assume E(I) is passed as the error map.
    delta_map = full_render - budgeted_render
    
    # Variance over all pixels in the image domain Omega
    variance = np.var(delta_map)
    return variance
```

### Parameters
- $\eta \in (0, 1]$: Learning rate for the exponential moving average of complexity [S1, unofficial].
- $\delta = 0.2$: Complexity dropout probability [S1, unofficial].
- $\lambda = 2$: Fixed scalar weight for the scale term in the importance score [S1, unofficial].
- $\tau$: Training budget factor. $\tau=1$ for full capacity, $\tau<1$ for reduced LoD [S1, unofficial].

### Edge Cases and Pitfalls
- **Division by Zero:** If all Gaussians have identical complexity, min-max normalization results in a zero denominator. The pseudocode handles this by returning zeros [S1, unofficial].
- **Collapse:** Without complexity dropout ($\delta=0$), the method may collapse onto fixed complex regions. The default $\delta=0.2$ prevents this [S1, unofficial].
- **Frustum Culling:** Complexity updates must only occur for Gaussians whose centers project within the current view's frustum; otherwise, out-of-view Gaussians receive stale or incorrect complexity updates [S1, unofficial].
- **Scale Normalization:** The scale $s(g_j)$ must be max-normalized across the current set of Gaussians to ensure consistent weighting relative to the current scene geometry [S1, unofficial].

## Subtopics

### Complexity-Aware Density Control
The core mechanism of biasing relocation probabilities using the exponential of normalized complexity. It differs from opacity-only relocation by ensuring new primitives are spawned in high-complexity areas even if current opacity is low. Choose when reconstruction error is spatially uneven. See [Splitting Criteria and Prioritization](16226-splitting-criteria-and-prioritization.md).

### Complexity-Aware Level of Details
The application of the composite importance score $S(g_j)$ to prune Gaussians under a budget $\tau$. It differs from simple opacity pruning by preserving large primitives in low-complexity regions, ensuring uniform degradation. Choose when variable LoD support is required. See [Splitting Criteria and Prioritization](16226-splitting-criteria-and-prioritization.md).

## Operating characteristics
- **Time Complexity:** Complexity update is $O(N)$ per view, where $N$ is the number of contributing Gaussians. Normalization is $O(N)$. Relocation sampling is $O(N)$ for weight computation and $O(1)$ for sampling. Importance scoring is $O(N)$. The variance loss computation is $O(P)$ where $P$ is the number of pixels.
- **Memory Complexity:** Requires storing an additional scalar $C_j$ and $\tilde{C}_j$ per Gaussian, increasing memory by $O(N)$.
- **Performance:** At full capacity, performance is on par with standard 3DGS. At lower budgets (e.g., 10%, 1%), it significantly outperforms baselines like OctreeGS and FlexGS in PSNR, SSIM, and LPIPS, maintaining higher quality with fewer primitives [S1, unofficial].
- **Strengths:** Ensures uniform degradation across LoD levels; prevents collapse on complex regions; effectively utilizes limited primitive budgets.
- **Limitations:** Requires computation of delentropy-based complexity maps, which adds overhead; sensitive to the choice of $\delta$ and $\lambda$; requires varying $\tau$ during training to be effective [S1, unofficial].

## Background
The method originates from the KoalaGS paper, which addresses the limitations of opacity-based density control in 3D Gaussian Splatting [S1, unofficial]. Standard methods fail to resolve untrained regions because they rely on accumulated opacity, which is low in unresolved areas. By using complexity (delentropy) as a surrogate for reconstruction error, the method biases optimization towards difficult regions [S1, unofficial]. It was developed to provide continuous Level of Detail (LoD) support, allowing efficient rendering at various budgets while maintaining visual quality [S1, unofficial].

## Sources
[S1] KoalaGS: Complexity-Aware Continuous Level of Detail for Gaussian Splatting, https://arxiv.org/abs/2610.28615, fetched 2026-10-26 (unofficial)
