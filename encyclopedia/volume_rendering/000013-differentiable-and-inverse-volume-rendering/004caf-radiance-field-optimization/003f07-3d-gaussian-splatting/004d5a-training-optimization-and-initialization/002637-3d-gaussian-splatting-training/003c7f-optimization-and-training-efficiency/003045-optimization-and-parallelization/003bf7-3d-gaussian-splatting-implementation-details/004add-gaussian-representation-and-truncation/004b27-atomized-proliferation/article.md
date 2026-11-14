# Atomized Proliferation

Also called: Atom Gaussian Densification

## Overview
Atomized Proliferation is a training optimization strategy for 3D Gaussian Splatting that enhances geometric precision by constraining Gaussians representing fine details into "Atom Gaussians" (isotropic spheroids with uniform size) and prioritizing their proliferation to align with scene geometry [S1, unofficial]. Unlike standard 3D Gaussian Splatting, which struggles to stabilize densification and often produces noisy meshes or blurry artifacts, Atomized Proliferation uses a warm-up strategy that aggressively densifies initially, then merges large smooth surfaces, leading to a declining and stabilizing trend in Gaussian count [S1, unofficial]. It builds on [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md) by introducing specific atomization and geometry-guided pruning mechanisms.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians defined by mean $\mu$, covariance $\Sigma$, color $c$, and opacity $\alpha$; input Structure-from-Motion (SfM) points; hyperparameters $\tau_c, \tau_s, \varepsilon, S, t_a, t_w$.
*   **Outputs:** An optimized set of Gaussians where fine details are represented by fixed-size Atom Gaussians and large surfaces are merged.

**Data Structures:**
The algorithm iterates over the Gaussian set $(M, S, C, A)$ representing means, covariances, colors, and opacities [S1, unofficial].
*   **Atom Scale ($S$):** A scalar threshold derived from SfM points. $S = P_i(d)$, where $P_i$ is the $i$-th percentile of ordered distances between SfM points [S1, unofficial].
*   **Atom Gaussian:** A Gaussian designated as an Atom Gaussian if its minimum covariance norm $||\Sigma||_{min} \le S$. Atom Gaussians are isotropic spheroids with uniform size $s = \sqrt[3]{S}$ [S1, unofficial].

**Parameters:**
*   **Clone Threshold ($\tau_c$):** Threshold for cloning based on loss gradient. Typical value: 0.002 [S1, unofficial].
*   **Split Threshold ($\tau_s$):** Base threshold for splitting. Typical value: 0.002 [S1, unofficial].
*   **Prune Threshold ($\varepsilon$):** Opacity threshold for pruning. Typical value: 0.005 (0.5 for DTU dataset) [S1, unofficial].
*   **Atom Scale ($S$):** Maximum allowed minimum covariance norm for atomization. Defined as $P_1(d)$ (first percentile) for general scenes, or $P_{10}(d)$ for DTU [S1, unofficial].
*   **Atomized Proliferation Duration ($t_a$):** Iteration limit for atomization. Typical value: 7000 [S1, unofficial].
*   **Warm-Up Duration ($t_w$):** Iteration limit for the warm-up phase. Typical value: 7000 [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Calculate Atom Scale $S$ from SfM points. Initialize Gaussian set.
2.  **Iteration Loop:** For each training iteration $i$ from 1 to total iterations:
    3.  **Atomization Check:** For each Gaussian, if $||\Sigma||_{min} \le S$ and $i < t_a$, designate it as an Atom Gaussian.
    4.  **Scale Fixing:** For Atom Gaussians, fix the scale (no backpropagation). Update scale via geometric progression: $S_n = S_{n-1} \cdot r^{n-1}$, where $r = \sqrt[t_a]{p}$ and $p$ is the final proportion to initial $S$ [S1, unofficial]. *Note: The source describes $S_n$ as the scale at iteration $n$, but the exact definition of $p$ and the base of the exponent in the provided text ($r^{n-1}$ vs $r_n$) implies a decay over the atomization period.*
    5.  **Pruning:** If opacity $\alpha < \varepsilon$ or covariance $\Sigma$ is excessively large ($IsTooLarge(\Sigma)$), prune the Gaussian [S1, unofficial].
    6.  **Cloning:** If loss gradient $\nabla_p L \ge \tau_c$, clone the Gaussian [S1, unofficial].
    7.  **Splitting:** If $\nabla_p L \ge \min(\frac{i}{t_w}\tau_s, \tau_s)$ AND $||\Sigma||_{max} > S$, split the Gaussian [S1, unofficial]. This uses a linearly increasing threshold during the warm-up phase ($i < t_w$).
    8.  **Opacity Resetting:** Increase the frequency of opacity resetting to merge Gaussians depicting large, simple surfaces [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def atomized_proliferation(gaussians, sfm_points, tau_c, tau_s, epsilon, S, ta, tw):
    """
    gaussians: List of Gaussian objects with attributes mu, Sigma, c, alpha, grad
    sfm_points: List of 3D points from SfM
    tau_c: Clone threshold
    tau_s: Split threshold
    epsilon: Prune threshold
    S: Atom scale threshold
    ta: Atomization duration
    tw: Warm-up duration
    """
    # Calculate Atom Scale S from SfM points
    distances = compute_pairwise_distances(sfm_points)
    S = percentile(distances, 1) # P1(d)
    
    for i in range(1, total_iterations + 1):
        new_gaussians = []
        
        for g in gaussians:
            # 1. Atomization
            if norm_min(g.Sigma) <= S and i < ta:
                g.is_atom = True
                # Fix scale and update via geometric progression
                # r = p^(1/ta) where p is final proportion
                # S_n = S_{n-1} * r^(n-1)
                # Note: Exact implementation of p and progression index depends on specific codebase
                update_atom_scale(g, i, ta)
            
            # 2. Pruning
            if g.alpha < epsilon or is_too_large(g.Sigma):
                continue # Prune
            
            # 3. Cloning
            if g.grad >= tau_c:
                clone_gaussian(g)
                # Cloned gaussians are added to the set
            
            # 4. Splitting
            # Dynamic threshold during warm-up
            if i < tw:
                split_threshold = min((i / tw) * tau_s, tau_s)
            else:
                split_threshold = tau_s
                
            if g.grad >= split_threshold and norm_max(g.Sigma) > S:
                split_gaussian(g)
                # Split gaussians are added to the set
            
            # 5. Keep original if not pruned
            new_gaussians.append(g)
            
        # 6. Opacity Resetting (Frequency increased)
        reset_opacitys(new_gaussians, frequency_factor=i)
        
        gaussians = new_gaussians
        
    return gaussians

def norm_min(Sigma):
    # Return the minimum eigenvalue or singular value of Sigma
    return min(eigvals(Sigma))

def norm_max(Sigma):
    # Return the maximum eigenvalue or singular value of Sigma
    return max(eigvals(Sigma))

def is_too_large(Sigma):
    # Heuristic for excessively large covariance
    # Not explicitly defined in sources, but implied as a pruning condition
    # [G] Assuming a threshold based on trace or max eigenvalue
    return norm_max(Sigma) > some_large_threshold

def update_atom_scale(g, i, ta):
    # Geometric progression for atom scale
    # S_n = S_{n-1} * r^(n-1)
    # r = p^(1/ta)
    # [G] p is not explicitly defined, assuming p=1 for no decay or p<1 for decay
    # The source says "decreases over iterations", so p < 1 likely
    # However, without p, we cannot implement exactly.
    # We will assume a simple decay factor r < 1
    r = 0.99 # Placeholder for p^(1/ta)
    g.scale *= r
```

**Edge Cases and Pitfalls:**
*   **Precision:** The calculation of $S$ from SfM points requires robust distance ordering.
*   **Boundaries:** The warm-up phase ($i < t_w$) requires careful handling of the linear threshold to avoid premature splitting or cloning.
*   **Degenerate Input:** If SfM points are sparse, $S$ may be too large, leading to insufficient atomization. If SfM points are noisy, $S$ may be too small, leading to excessive fragmentation.

## Operating characteristics
*   **Time Complexity:** Similar to standard 3D Gaussian Splatting, but with additional checks for atomization and geometric progression updates. The warm-up phase may increase initial computational cost due to aggressive densification.
*   **Memory Complexity:** The final model size is reported as 749MB on Mip-NeRF360, compared to 869MB for standard 3DGS, indicating a reduction in Gaussian count through merging [S1, unofficial].
*   **Measured Performance:** On Mip-NeRF360, average training time is 0.28 hours vs 0.40 hours for 3DGS [S1, unofficial].
*   **Strengths:** Enhances geometric precision, reduces noisy meshes, stabilizes Gaussian count, and improves training efficiency.
*   **Limitations:** Requires SfM points for initialization. The geometric progression for atom scale requires careful tuning of $p$.
*   **When to use:** When high-fidelity geometric reconstruction is required, especially for scenes with intricate details.
*   **When not to use:** When SfM points are unavailable or of poor quality.

## Background
Atomized Proliferation was introduced in "AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field" by Liu et al. [S1, unofficial]. It addresses the issue of standard 3D Gaussian Splatting struggling to adhere to underlying geometric structures, leading to noisy meshes and blurry artifacts. The method combines atomization of fine details with geometry-guided optimization for smooth surfaces.

## Sources
[S1] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
