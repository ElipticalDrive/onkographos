# Atomized Proliferation

Also called: Atom Gaussian Densification

## Overview
Atomized Proliferation is a training strategy for 3D Gaussian Splatting that enhances geometric precision by constraining Gaussians representing fine details into "Atom Gaussians" (isotropic spheroids with uniform size) and prioritizing their proliferation to align with scene geometry, followed by pruning to merge large smooth surfaces [S1, unofficial]. It addresses the instability of standard densification and the tendency of Gaussians to form noisy, elongated structures that do not adhere to underlying geometric structures, leading to blurry artifacts and slower convergence [S1, unofficial]. Unlike standard 3DGS, which struggles to stabilize densification, AtomGS uses a warm-up strategy that aggressively densifies initially, then merges, leading to a declining and stabilizing trend in Gaussian count [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of Gaussians $\mathcal{G} = \{(\mu_i, \Sigma_i, c_i, \alpha_i)\}$, where $\mu$ is mean position, $\Sigma$ is covariance, $c$ is color, and $\alpha$ is opacity; input SfM point cloud distances $d$; hyperparameters $\tau_c, \tau_s, \varepsilon, S, t_a, t_w$.
*   **Output:** Refined set of Gaussians with improved geometric alignment and reduced redundancy.

**Data Structures:**
*   **Gaussian Set:** List of tuples $(\mu, \Sigma, c, \alpha)$.
*   **Atom Scale $S$:** A scalar value derived from SfM points.
*   **Iteration Counter $i$:** Current training iteration.

**Parameters:**
*   **Clone Threshold ($\tau_c$):** Threshold for cloning based on loss gradient. Typical value: $0.002$ [S1, unofficial].
*   **Split Threshold ($\tau_s$):** Base threshold for splitting. Typical value: $0.002$ [S1, unofficial].
*   **Prune Threshold ($\varepsilon$):** Opacity threshold for pruning. Typical value: $0.005$ (or $0.5$ for DTU dataset) [S1, unofficial].
*   **Atom Scale ($S$):** Maximum allowed minimum covariance norm for atomization. Defined as the $i$-th percentile of distances from input SfM points, $S = P_i(d)$. Typical setting: $P_1(d)$ (1st percentile) [S1, unofficial].
*   **Atomized Proliferation Duration ($t_a$):** Iteration limit for atomization. Typical value: $7000$ [S1, unofficial].
*   **Warm-Up Duration ($t_w$):** Iteration limit for the warm-up phase. Typical value: $7000$ [S1, unofficial].

**Algorithm Steps:**
1.  **Calculate Atom Scale:** Compute $S$ from the input SfM points. Let $d$ be the sorted distances between SfM points. $S = P_i(d)$, where $P_i$ is the $i$-th percentile. Standard implementation uses $i=1$ [S1, unofficial].
2.  **Iterate Gaussians:** For each Gaussian $g \in \mathcal{G}$ at iteration $i$:
    a.  **Pruning:** If opacity $\alpha < \varepsilon$ OR the covariance $\Sigma$ is excessively large (determined by `IsTooLarge`), remove the Gaussian [S1, unofficial].
    b.  **Cloning:** If the loss gradient $\nabla_p L \ge \tau_c$, clone the Gaussian [S1, unofficial].
    c.  **Splitting:** If $\nabla_p L \ge \min(\frac{i}{t_w}\tau_s, \tau_s)$ AND the maximum norm of the covariance $\|\Sigma\|_{\max} > S$, split the Gaussian. This condition applies during the warm-up phase ($i < t_w$) using a linearly increasing threshold [S1, unofficial].
    d.  **Atomization:** If the minimum norm of the covariance $\|\Sigma\|_{\min} \le S$ AND $i < t_a$, atomize the Gaussian [S1, unofficial].

**Atomization Details:**
*   **Definition:** An Atom Gaussian is an isotropic spheroid with uniform size $s_1 = s_2 = s_3 = s$.
*   **Size Assignment:** When a Gaussian is atomized, its size is set to $s = \sqrt[3]{S}$ [S1, unofficial].
*   **Scale Fixing:** Once designated as an Atom Gaussian, its scale is fixed and no longer optimized through backpropagation.
*   **Geometric Progression:** The atom scale decreases over iterations to refine details. The scale at iteration $n$ is updated as $S_n = S_{n-1} \cdot r^{n-1}$, where $r = \sqrt[t_a]{p}$ and $p$ is the final proportion to the initial $S$ [S1, unofficial]. Note: The source text contains a typo "$\sqrt{S}^3$" which is interpreted as $\sqrt[3]{S}$ based on context "size set to s = (S)3" and standard cube root notation for volume-to-size conversion. The progression formula $S_n = S_{n-1} \cdot r^{n-1}$ implies a decay if $r < 1$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def atomized_proliferation(Gaussians, SfM_distances, tau_c, tau_s, epsilon, ta, tw):
    # Calculate Atom Scale S from SfM points
    # d is sorted distances
    d = sorted(SfM_distances)
    S = percentile(d, 1) # 1st percentile
    
    # Precompute rate for geometric progression if needed for scale updates
    # p is final proportion to initial S. Source says r = t^(1/a) where ta is iterations.
    # The text says "r = t^(1/a)" but also "ta represents total atomization iterations".
    # Assuming p=1 for simplicity if not specified, or r is derived from target decay.
    # Source: "r = t^(1/a)" is likely a typo for "r = p^(1/ta)".
    # We assume a default p=1 (no decay) or p<1 if specified. 
    # Given "S_n = S_{n-1} * r^{n-1}", if r=1, S is constant.
    # Let's assume r is a parameter or fixed. Source doesn't specify p value.
    # [G] Default p=1.0 if not specified, meaning S remains constant at S.
    p = 1.0
    r = p ** (1.0 / ta)
    
    current_atom_scale = S
    
    for i, gaussian in enumerate(Gaussians):
        mu, Sigma, c, alpha = gaussian
        grad = get_loss_gradient(gaussian) # External function
        
        # 1. Pruning
        if alpha < epsilon or is_too_large(Sigma):
            remove_gaussian(gaussian)
            continue
            
        # 2. Cloning
        if grad >= tau_c:
            clone_gaussian(gaussian)
            
        # 3. Splitting
        # Dynamic threshold: min((i/tw)*tau_s, tau_s)
        dynamic_threshold = min((i / tw) * tau_s, tau_s)
        max_cov_norm = get_max_covariance_norm(Sigma)
        
        if grad >= dynamic_threshold and max_cov_norm > S:
            split_gaussian(gaussian)
            
        # 4. Atomization
        min_cov_norm = get_min_covariance_norm(Sigma)
        
        if min_cov_norm <= S and i < ta:
            atomize_gaussian(gaussian, current_atom_scale)
            
        # Update atom scale for next iteration if atomization is active
        # The source implies S decreases.
        if i < ta:
            current_atom_scale = current_atom_scale * r

def atomize_gaussian(gaussian, scale_s):
    mu, Sigma, c, alpha = gaussian
    # Set size s = cube root of S
    s = scale_s ** (1.0 / 3.0)
    # Make isotropic
    Sigma = isotropic_covariance(s)
    # Fix scale (no backprop)
    gaussian.scale_requires_grad = False
    gaussian.covariance = Sigma
```

**Edge Cases and Pitfalls:**
*   **DTU Dataset:** For object-centered datasets like DTU, the Prune Threshold $\varepsilon$ should be increased to $0.5$ to avoid excessive pruning of valid geometry [S1, unofficial].
*   **Scale Learning Rate:** For object-centered datasets, set scale learning rate $\eta_s = 0$ to maximize geometric accuracy [S1, unofficial].
*   **Precision:** The calculation of covariance norms ($\|\Sigma\|_{\max}, \|\Sigma\|_{\min}$) must be robust to numerical instability in small Gaussians.
*   **Warm-up Phase:** The linear warm-up of the split threshold ensures aggressive densification early on. If $t_w$ is too short, the threshold rises too quickly, preventing sufficient atomization.

## Operating characteristics
*   **Time Complexity:** Training time is comparable to or slightly faster than standard 3DGS. On Mip-NeRF360, AtomGS averages 0.28 hours vs 3DGS's 0.40 hours [S1, unofficial].
*   **Memory Complexity:** Model size is reduced. On Mip-NeRF360, AtomGS results in 749MB vs 3DGS's 869MB [S1, unofficial].
*   **Scaling:** The method stabilizes Gaussian count, leading to a declining and stabilizing trend, which improves memory efficiency over time compared to the fluctuating count in standard 3DGS [S1, unofficial].
*   **Strengths:** Enhances geometric precision, reduces noisy meshes and blurry artifacts, stabilizes densification, and compactly models smooth surfaces while retaining fine details [S1, unofficial].
*   **Limitations:** Requires careful tuning of $\varepsilon$ for different datasets (e.g., DTU vs Mip-NeRF360). The atomization process adds complexity to the training loop.

## Background
Atomized Proliferation is part of the AtomGS method, proposed by Liu, Xu, Hu, Chen, and Feng. It was introduced to resolve issues in standard 3D Gaussian Splatting where Gaussians do not adhere to underlying geometric structures, leading to noisy 3D meshes and slower convergence [S1, unofficial]. The method combines atomized proliferation with geometry-guided optimization [S1, unofficial].

## Sources
[S1] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
