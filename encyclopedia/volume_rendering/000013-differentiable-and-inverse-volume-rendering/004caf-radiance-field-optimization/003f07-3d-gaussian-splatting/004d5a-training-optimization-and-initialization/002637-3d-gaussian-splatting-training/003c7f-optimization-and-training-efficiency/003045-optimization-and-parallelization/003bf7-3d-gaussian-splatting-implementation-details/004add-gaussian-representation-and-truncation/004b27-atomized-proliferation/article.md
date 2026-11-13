# Atomized Proliferation

Also called: Atom Gaussian Densification

## Overview
Atomized Proliferation is a training optimization strategy for 3D Gaussian Splatting that enhances geometric precision by constraining Gaussians representing fine details into "Atom Gaussians" (isotropic spheroids with uniform size) and prioritizing their proliferation to align with scene geometry, followed by pruning to merge large smooth surfaces [S1, unofficial]. Unlike standard 3DGS which struggles to stabilize densification, AtomGS uses a warm-up strategy that aggressively densifies initially, then merges, leading to a declining and stabilizing trend in Gaussian count [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of Gaussians $G = \{(\mu, \Sigma, c, \alpha)\}$, where $\mu$ is the mean position, $\Sigma$ is the covariance matrix, $c$ is the color, and $\alpha$ is the opacity. Input SfM point cloud distances $d$.
*   **Output:** Updated set of Gaussians with refined geometry and optimized parameters.

**Data Structures and Parameters:**
*   **Atom Scale ($S$):** The maximum allowed minimum covariance norm for atomization. Calculated as the $i$-th percentile of ordered distances from input SfM points: $S = P_i(d)$. Default: $i=1$ (first percentile) [S1, unofficial].
*   **Clone Threshold ($\tau_c$):** Threshold for cloning based on loss gradient. Default: $0.002$ [S1, unofficial].
*   **Split Threshold ($\tau_s$):** Base threshold for splitting. Default: $0.002$ [S1, unofficial].
*   **Prune Threshold ($\epsilon$):** Opacity threshold for pruning. Default: $0.005$ (or $0.5$ for DTU dataset) [S1, unofficial].
*   **Atomized Proliferation Duration ($t_a$):** Iteration limit for atomization. Default: $7000$ [S1, unofficial].
*   **Warm-Up Duration ($t_w$):** Iteration limit for the warm-up phase. Default: $7000$ [S1, unofficial].
*   **Current Iteration ($i$):** The current training step index [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Calculate $S$ from SfM points. Set $t_a$ and $t_w$. Initialize Gaussian set [S1, unofficial].
2.  **Iteration Loop:** For each training step $i$ from $0$ to total iterations:
    3.  **Atom Scale Update:** If $i < t_a$, update the atom scale $S$ using geometric progression: $S_n = S_{n-1} \cdot r^{n-1}$, where $r = t_a^{\sqrt{p}}$ and $p$ is the final proportion to the initial $S$. Note: The source formula for $r$ is given as $r = t_a^{\sqrt{p}}$ in the text description but the algorithm description says $r = t_a^{\sqrt{p}}$ is not quite right, it says $r = t_a^{\sqrt{p}}$? Let's re-read carefully. "where $r = t_a^{\sqrt{p}}$ and $t_a$ represents the total number of atomization iterations, and $p$ is the final proportion to the initial $S$." This seems to imply $S$ decreases. The text says "atom scale decreases over iterations". The formula $S_n = S_{n-1} \cdot r^{n-1}$ suggests exponential decay if $r < 1$. The source text says $r = t_a^{\sqrt{p}}$. This is ambiguous. However, the key constraint is $||\Sigma||_{min} \le S$. We will use the condition check directly [S1, unofficial].
    4.  **Process Each Gaussian:** For each Gaussian $(\mu, \Sigma, c, \alpha)$ in the set:
        5.  **Pruning:** If $\alpha < \epsilon$ OR `IsTooLarge($\Sigma$)` is true, prune the Gaussian. `IsTooLarge` is not explicitly defined in the notes but implies a threshold on covariance size [S1, unofficial].
        6.  **Cloning:** If the loss gradient $\nabla_p L \ge \tau_c$, clone the Gaussian. This allows larger Gaussians to be cloned to bridge geometry gaps [S1, unofficial].
        7.  **Splitting:** If $\nabla_p L \ge \min(\frac{i}{t_w}\tau_s, \tau_s)$ AND $||\Sigma||_{max} > S$, split the Gaussian. The threshold $\min(\frac{i}{t_w}\tau_s, \tau_s)$ implements a linear warm-up, increasing the threshold during the warm-up phase $i < t_w$ to enhance the probability of division into smaller Atom Gaussians [S1, unofficial].
        8.  **Atomization:** If $||\Sigma||_{min} \le S$ AND $i < t_a$, atomize the Gaussian.
            *   Designate as Atom Gaussian.
            *   Set size $s = \sqrt[3]{S}$ (isotropic spheroid, $s_1=s_2=s_3$).
            *   Fix scale: No backprop optimization for scale.
            *   Update scale via geometric progression: $S_n = S_{n-1} \cdot r^{n-1}$ where $r$ is derived from $t_a$ and $p$. The source states $r = t_a^{\sqrt{p}}$ which seems dimensionally inconsistent or typoed, but the intent is a decreasing scale. We implement the condition $||\Sigma||_{min} \le S$ and the fixed scale constraint [S1, unofficial].
    9.  **Opacity Resetting:** Increase frequency of opacity resetting to merge Gaussians depicting large, simple surfaces, rather than eliminating noisy ones near the camera [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def atomized_proliferation(Gaussians, SfM_distances, tau_c, tau_s, epsilon, t_a, t_w):
    """
    Gaussians: List of objects with attributes mu, Sigma, c, alpha, grad_L
    SfM_distances: List of distances from SfM points
    tau_c: Clone threshold
    tau_s: Split threshold
    epsilon: Prune threshold
    t_a: Atomization duration
    t_w: Warm-up duration
    """
    # Calculate Atom Scale S from SfM points
    # S = P_i(d), typically i=1 (first percentile)
    S = percentile(SfM_distances, 1)
    
    # Initialize scale progression parameters
    # r = t_a^(sqrt(p)) is ambiguous in source, but S decreases.
    # We track current S.
    current_S = S
    
    for i in range(total_iterations):
        # Update Atom Scale if within atomization duration
        if i < t_a:
            # Geometric progression to decrease S
            # Source: S_n = S_{n-1} * r^{n-1}, r = t_a^sqrt(p)
            # This formula is unclear. We assume S decreases over time.
            # A common interpretation for "proportion p" might be related to 
            # the target scale. However, without exact p, we use the condition check.
            # Let's assume a simple decay or use the provided formula if p is known.
            # For implementation, we rely on the condition ||Sigma||_min <= S.
            # We will update S based on the geometric progression if p is defined.
            # If p is not defined, we might keep S constant or decay it.
            # The source says "atom scale decreases over iterations".
            # Let's assume a decay factor.
            # Note: The source formula r = t_a^sqrt(p) is likely a typo for r = p^(1/t_a) or similar.
            # Given the ambiguity, we will use the condition check primarily.
            # [G] Assuming a simple decay for S if p is not provided:
            # current_S = current_S * decay_factor
            # However, the source explicitly mentions geometric progression.
            # We will implement the check ||Sigma||_min <= S.
            pass 

        new_gaussians = []
        
        for g in Gaussians:
            # Pruning
            if g.alpha < epsilon or is_too_large(g.Sigma):
                continue # Prune
            
            # Cloning
            if g.grad_L >= tau_c:
                new_gaussians.append(clone(g))
            
            # Splitting
            # Linear warm-up for split threshold
            if i < t_w:
                split_threshold = (i / t_w) * tau_s
            else:
                split_threshold = tau_s
            
            if g.grad_L >= split_threshold and max_norm(g.Sigma) > current_S:
                new_gaussians.append(split(g))
            
            # Atomization
            if min_norm(g.Sigma) <= current_S and i < t_a:
                atomized_g = atomize(g, current_S)
                new_gaussians.append(atomized_g)
            else:
                new_gaussians.append(g)
        
        Gaussians = new_gaussians
        
        # Opacity resetting to merge large surfaces
        if should_reset_opacity(i):
            reset_opacities(Gaussians)
            
        # Update current_S for next iteration if needed
        if i < t_a:
            # Update S based on geometric progression
            # r = t_a^sqrt(p) is ambiguous. 
            # Let's assume p is a target proportion, e.g., 0.1.
            # r = p^(1/t_a) would make S decrease.
            # Source: r = t_a^sqrt(p). This is likely wrong.
            # We will skip explicit S update if p is unknown, relying on the condition.
            # [G] Assuming a standard geometric decay for S:
            # current_S = current_S * (0.1 ** (1/t_a))
            pass

    return Gaussians

def atomize(g, S):
    """
    Convert Gaussian to Atom Gaussian with fixed isotropic scale.
    """
    # Size s = cube_root(S)
    s = S ** (1/3)
    # Create isotropic covariance
    # Sigma is diagonal with s, s, s
    new_Sigma = create_isotropic_covariance(s)
    # Fix scale: no gradient update for scale
    g.Sigma = new_Sigma
    g.is_atom = True
    return g

def is_too_large(Sigma):
    """
    Check if covariance is excessively large.
    Not defined in source. [G] Assume max norm > some large threshold.
    """
    return max_norm(Sigma) > LARGE_THRESHOLD
```

**Edge Cases and Pitfalls:**
*   **Ambiguous Scale Update Formula:** The source formula $r = t_a^{\sqrt{p}}$ is dimensionally inconsistent and likely contains a typo. Implementers should verify the intended geometric progression for $S$ [S1, unofficial].
*   **DTU Dataset:** For object-centered datasets like DTU, the Prune Threshold $\epsilon$ should be set to $0.5$ and Atom Scale $S$ to $P_{10}(d)$ [S1, unofficial].
*   **Scale Learning Rate:** For object-centered datasets, set scale learning rate $\eta_s = 0$ to maximize geometric accuracy [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time is comparable to or slightly faster than standard 3DGS. On Mip-NeRF360, AtomGS averages 0.28 hours vs 3DGS 0.40 hours [S1, unofficial].
*   **Memory Complexity:** Model size is reduced. On Mip-NeRF360, AtomGS results in 749MB vs 3DGS 869MB [S1, unofficial].
*   **Strengths:** Enhances geometric precision, stabilizes densification, reduces model size, and improves convergence speed [S1, unofficial].
*   **Limitations:** Requires careful tuning of thresholds ($\tau_c, \tau_s, \epsilon$) and scale parameters ($S, t_a, t_w$). The ambiguity in the scale update formula may lead to implementation errors [S1, unofficial].
*   **When to Use:** When high-fidelity geometry and fine details are critical, and model size/computation time are concerns. Particularly effective for scenes with intricate details [S1, unofficial].

## Background
Atomized Proliferation is part of the AtomGS method, proposed by Liu, Xu, Hu, Chen, and Feng. It addresses issues in standard 3DGS where Gaussians may not adhere to underlying geometric structures, leading to noisy meshes and blurry artifacts [S1, unofficial]. The method was presented in "AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field" [S1, unofficial].

## Sources
[S1] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
