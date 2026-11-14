# Atomized Proliferation

Also called: Atom Gaussian Densification

## Overview
Atomized Proliferation is a training optimization strategy for 3D Gaussian Splatting that enhances geometric precision by constraining fine-detail Gaussians into uniform "Atom Gaussians" and prioritizing their proliferation to align with scene geometry, followed by pruning to merge large smooth surfaces [S1, unofficial]. Unlike standard 3D Gaussian Splatting, which struggles to stabilize densification and often results in noisy meshes or blurry artifacts, Atomized Proliferation uses a warm-up strategy that aggressively densifies initially and then merges, leading to a declining and stabilizing trend in Gaussian count [S1, unofficial]. It builds on [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of Gaussians $G = \{(\mu, \Sigma, c, \alpha)\}$, where $\mu$ is mean position, $\Sigma$ is covariance, $c$ is color, and $\alpha$ is opacity; input SfM point cloud distances $d$; hyperparameters $\tau_c, \tau_s, \varepsilon, S, t_a, t_w$.
*   **Outputs:** Updated Gaussian set with refined geometry and optimized parameters.

**Data Structures:**
*   **Atom Scale ($S$):** A scalar value representing the maximum allowed minimum covariance norm for atomization. Calculated as the $i$-th percentile of ordered distances from the input SfM points: $S = P_i(d)$. Typically, $i=1$ (first percentile) is used for general scenes, but $i=10$ for object-centered datasets like DTU [S1, unofficial].
*   **Atom Gaussian:** A Gaussian designated as an Atom Gaussian if its minimum covariance norm $||\Sigma||_{min} \le S$. These are isotropic spheroids with uniform size $s = \sqrt[3]{S}$ [S1, unofficial].

**Parameters:**
*   **Clone Threshold ($\tau_c$):** Threshold for cloning based on loss gradient. Typical value: $0.002$ [S1, unofficial].
*   **Split Threshold ($\tau_s$):** Base threshold for splitting. Typical value: $0.002$ [S1, unofficial].
*   **Prune Threshold ($\varepsilon$):** Opacity threshold for pruning. Typical value: $0.005$ (or $0.5$ for DTU dataset) [S1, unofficial].
*   **Atomized Proliferation Duration ($t_a$):** Iteration limit for atomization. Typical value: $7000$ [S1, unofficial].
*   **Warm-Up Duration ($t_w$):** Iteration limit for the warm-up phase. Typical value: $7000$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Calculate Atom Scale $S$ from SfM points. Initialize iteration counter $i=0$.
2.  **Iteration Loop:** For each iteration $i$ from $0$ to total training iterations:
    a.  **Atom Scale Decay:** If $i < t_a$, update the effective Atom Scale $S_i$ using geometric progression: $S_i = S_{i-1} \cdot r^{i-1}$, where $r = \sqrt[t_a]{p}$ and $p$ is the final proportion to the initial $S$ [S1, unofficial]. [G: If $i=0$, $S_0 = S$.]
    b.  **Process Each Gaussian:** For each Gaussian $g \in G$:
        i.  **Pruning:** If $\alpha < \varepsilon$ or $||\Sigma||_{max}$ is excessively large (defined by `IsTooLarge`), remove $g$ from $G$.
        ii. **Cloning:** If loss gradient $\nabla_p L \ge \tau_c$, clone $g$. This allows larger Gaussians to be cloned to bridge geometry gaps [S1, unofficial].
        iii. **Splitting:** If $\nabla_p L \ge \min(\frac{i}{t_w}\tau_s, \tau_s)$ AND $||\Sigma||_{max} > S$, split $g$. The threshold $\frac{i}{t_w}\tau_s$ linearly increases during the warm-up phase ($i < t_w$) to enhance the probability of division into smaller Atom Gaussians [S1, unofficial].
        iv. **Atomization:** If $||\Sigma||_{min} \le S$ AND $i < t_a$, designate $g$ as an Atom Gaussian. Set its size to $s = \sqrt[3]{S}$. Fix its scale parameters (no backprop optimization). Update scale via geometric progression $S_n = S_{n-1} \cdot r^{n-1}$ [S1, unofficial].
    c.  **Opacity Resetting:** Increase the frequency of opacity resetting to merge Gaussians depicting large, simple surfaces, rather than eliminating noisy ones near the camera [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def atomized_proliferation(Gaussians, SfM_distances, tau_c, tau_s, epsilon, ta, tw, total_iterations):
    # Calculate initial Atom Scale S from SfM points (e.g., 1st percentile)
    S_initial = percentile(SfM_distances, 1)
    
    # Define geometric progression factor r
    # p is the final proportion to initial S (e.g., if S decreases to 0.1*S_initial, p=0.1)
    # r = p^(1/ta)
    p = 0.1  # Example value, source does not specify exact p, only that r = ta^(1/p) is incorrect notation in source, 
             # source says r = ta^(1/p) but context implies r = p^(1/ta) or similar geometric decay. 
             # Source: "r = ta^(1/p)" is likely a typo for "r = p^(1/ta)" or "r = (final_S/initial_S)^(1/ta)".
             # We use the formula Sn = Sn-1 * r^(n-1) where r is constant.
    # Note: Source text "r = ta^(1/p)" is ambiguous. Standard geometric decay is r = (final/initial)^(1/ta).
    # We will assume r is a constant decay factor derived from ta and a target proportion.
    # For implementation, r must be defined. Let's assume r is provided or calculated as (S_final/S_initial)^(1/ta).
    # Since source doesn't give S_final, we note this as a parameter to be set.
    r = 0.99  # Placeholder for geometric decay factor, typically < 1
    
    for i in range(total_iterations):
        # Update Atom Scale for atomization check
        if i < ta:
            # Geometric progression for Atom Scale
            # S_i = S_initial * r^i
            current_S = S_initial * (r ** i)
        else:
            current_S = S_initial # Or keep last value
            
        new_Gaussians = []
        clones = []
        splits = []
        
        for g in Gaussians:
            mu, Sigma, c, alpha = g
            
            # 1. Pruning
            if alpha < epsilon or is_too_large(Sigma):
                continue # Prune
            
            # 2. Cloning
            if g.loss_gradient >= tau_c:
                clones.append(g)
                
            # 3. Splitting
            # Dynamic threshold during warm-up
            if i < tw:
                split_threshold = (i / tw) * tau_s
            else:
                split_threshold = tau_s
                
            if g.loss_gradient >= split_threshold and max_norm(Sigma) > current_S:
                splits.append(g)
                
            # 4. Atomization
            if min_norm(Sigma) <= current_S and i < ta:
                # Designate as Atom Gaussian
                # Fix scale, set size s = cube_root(current_S)
                # Update scale via geometric progression in subsequent steps
                g.is_atom = True
                g.scale_fixed = True
                g.size = current_S ** (1/3)
                # Note: The source says scales are updated via geometric progression.
                # This implies the 'size' or 'scale' parameter evolves.
                # We store the current S for this Gaussian to decay it.
                g.atom_scale_history = current_S
                
            new_Gaussians.append(g)
            
        # Apply Clones
        for g in clones:
            # Clone logic: create new Gaussian with similar properties
            new_g = clone_gaussian(g)
            new_Gaussians.append(new_g)
            
        # Apply Splits
        for g in splits:
            # Split logic: create two new Gaussians
            new_g1, new_g2 = split_gaussian(g)
            new_Gaussians.append(new_g1)
            new_Gaussians.append(new_g2)
            
        # Update Atom Scales for existing Atom Gaussians
        for g in new_Gaussians:
            if g.is_atom and i < ta:
                # Decay scale
                g.size = g.size * r
                
        Gaussians = new_Gaussians
        
        # Opacity Resetting (Frequency increased)
        reset_opacities(Gaussians, frequency_factor=2.0) # [G: Specific reset logic depends on base 3DGS implementation]
        
    return Gaussians

def is_too_large(Sigma):
    # Check if covariance is excessively large
    # Typically based on max eigenvalue or trace
    return max_eigenvalue(Sigma) > threshold # [G: Threshold not specified in source]

def max_norm(Sigma):
    return max_eigenvalue(Sigma)

def min_norm(Sigma):
    return min_eigenvalue(Sigma)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The geometric progression for Atom Scale requires careful handling of floating-point precision to avoid underflow over many iterations [S1, unofficial].
*   **Boundaries:** The `IsTooLarge` function is not explicitly defined in the source. Implementers must define a threshold for covariance size [G].
*   **Degenerate Input:** If SfM points are sparse, the Atom Scale $S$ may be too large, leading to insufficient atomization. Conversely, if too dense, $S$ may be too small, causing excessive fragmentation [S1, unofficial].
*   **Dataset Specifics:** For object-centered datasets (e.g., DTU), the Prune Threshold $\varepsilon$ should be increased to $0.5$ and Atom Scale $S$ to the 10th percentile $P_{10}(d)$ to avoid over-pruning [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time is comparable to or slightly faster than standard 3DGS. On Mip-NeRF360, AtomGS averages 0.28 hours vs. 3DGS's 0.40 hours [S1, unofficial].
*   **Memory Complexity:** Model size is reduced. On Mip-NeRF360, AtomGS results in 749MB vs. 3DGS's 869MB [S1, unofficial].
*   **Performance:** Achieves superior geometric quality and faster convergence by stabilizing densification. The Gaussian count shows a declining and stabilizing trend, unlike the fluctuating count in standard 3DGS [S1, unofficial].
*   **Strengths:** Enhances geometric precision, reduces noisy meshes, and stabilizes training.
*   **Limitations:** Requires careful tuning of hyperparameters ($\tau_c, \tau_s, \varepsilon, S, t_a, t_w$) and may struggle with very sparse SfM data [S1, unofficial].

## Background
Atomized Proliferation was introduced in the paper "AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field" by Liu, Xu, Hu, Chen, and Feng [S1, unofficial]. It addresses issues in standard 3D Gaussian Splatting where Gaussians may not adhere to underlying geometric structures, leading to noisy meshes and blurry artifacts. The method was developed to improve geometric precision and convergence speed [S1, unofficial].

## Sources
[S1] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
