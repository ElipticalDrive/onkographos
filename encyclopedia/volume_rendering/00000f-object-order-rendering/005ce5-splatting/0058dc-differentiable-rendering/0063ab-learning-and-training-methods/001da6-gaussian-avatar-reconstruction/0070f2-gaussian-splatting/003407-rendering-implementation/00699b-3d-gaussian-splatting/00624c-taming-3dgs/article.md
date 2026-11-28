# Taming 3DGS

Also called: Budget-constrained 3DGS, Explicit Gaussian Cap, High-Quality Radiance Fields with Limited Resources, Taming3DGS

## Overview
Taming 3DGS is a resource-efficiency method for 3D Gaussian Splatting that imposes an explicit cap on the total number of Gaussians to enable high-quality rendering on consumer-grade GPUs [S1, unofficial]. It steers the densification process to spend a fixed budget where it helps most, trading peak quality for faster training and smaller model size [S2][S3, unofficial]. Unlike standard 3DGS which allows unbounded growth, this approach formulates optimization as a mixed-integer problem to maintain a constant Gaussian count [S1, unofficial].

## How it works
**Inputs:** A set of 3D Gaussians $\{G_i\}$ with attributes (position, covariance, opacity, SH coefficients), a target maximum count $N_{max}$, and a training schedule or importance metric [S2][S3, unofficial].
**Outputs:** A pruned and densified set of Gaussians $\{G'_i\}$ where $|\{G'_i\}| \le N_{max}$ [S2][S3, unofficial].

**Data Structures:**
*   **Gaussian List:** Array of size $N_{max}$ storing each Gaussian's attributes [S3, unofficial].
*   **Global Score:** A scalar value per Gaussian, computed from four factors: gradient magnitude, pixel coverage, per-view saliency, and core attributes (opacity, depth, scale) [S2].

**Steps:**
1. **Initialize:** Start with the standard 3DGS initialization [G]. Set current count $N = N_{init}$.
2. **Iterative Optimization Loop:**
   a. Perform standard differentiable rasterization and backpropagation to compute gradients [G].
   b. **Score:** Calculate the global score for each Gaussian using the four factors [S2].
   c. **Prune:** Identify the least important Gaussians based on the global score. Remove them if $N > N_{max}$ or if they fall below a threshold [S1, unofficial].
   d. **Densify:** Identify the Gaussians with the top $B$ scores, where $B$ is the desired number of new Gaussians. Split or clone these Gaussians to increase local density [S2].
   e. **Budget Enforcement:** Maintain $N \le N_{max}$ by pruning excess Gaussians after densification [S1, S3, unofficial].
3. **Schedule/Policy:** Use either a predictable schedule (fixed pruning/densification intervals) or adaptive importance guidance to decide when to apply steps 2b–2e [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def taming_3dgs_step(gaussians, N_max, B):
    # 1. Compute global score per Gaussian
    scores = []
    for g in gaussians:
        # [G] Standard differentiable rasterization provides positional gradients
        grad = compute_gradient_norm(g)
        # [G] Projected screen-space area
        coverage = compute_screen_coverage(g)
        # [G] View-dependent saliency metric
        saliency = compute_saliency(g)
        # [G] Normalized opacity, depth, and scale
        core = compute_core_attributes(g)
        scores.append(grad + coverage + saliency + core)

    # 2. Prune least important if over budget
    if len(gaussians) > N_max:
        sorted_indices = sorted(range(len(scores)), key=lambda i: scores[i])
        prune_count = len(gaussians) - N_max
        remove_indices = set(sorted_indices[:prune_count])
        gaussians = [g for i, g in enumerate(gaussians) if i not in remove_indices]
        scores = [s for i, s in enumerate(scores) if i not in remove_indices]

    # 3. Densify top B Gaussians
    if len(gaussians) < N_max:
        sorted_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)
        top_indices = sorted_indices[:B]
        new_primitives = []
        new_scores = []
        for idx in top_indices:
            g = gaussians[idx]
            # [G] Standard 3DGS split/clone: generate child with perturbed position/scale
            child = clone_with_perturbation(g, offset=0.01)
            new_primitives.append(child)
            new_scores.append(0.0)
        gaussians.extend(new_primitives)
        scores.extend(new_scores)

    return gaussians, scores
```

**Edge Cases:**
*   **Boundaries:** $N_{max}$ must be chosen carefully; too low causes quality loss, too high defeats the purpose [S2][S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time is reduced due to fewer Gaussians to rasterize and backpropagate through [S2][S3, unofficial].
*   **Memory Complexity:** Memory usage is bounded by $O(N_{max})$, significantly lower than unconstrained 3DGS [S2][S3, unofficial].
*   **Performance:** Enables high-quality rendering on consumer-grade GPUs [S1, unofficial].
*   **Strengths:** Faster training, smaller model size, predictable resource usage [S2][S3, unofficial].
*   **Limitations:** Slight quality loss compared to unconstrained 3DGS [S1, S3, unofficial].
*   **When to use:** When hardware resources are limited or training time is a constraint [S2][S3, unofficial].
*   **When not to use:** When peak visual fidelity is required regardless of cost [S3, unofficial].

## Background
Taming 3DGS was introduced to address the resource intensity of standard 3D Gaussian Splatting [S1, unofficial]. It explores a spectrum of trade-offs from predictable schedules to adaptive importance guidance [S1, unofficial]. The method is part of a broader trend of optimizing 3DGS for efficiency, alongside works like LightGaussian and RTGS [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
[S3] Gaussian Splats Part 2: From Photos to a Trained Model, https://alphapixeldev.com/gaussian-splats-part-2-creating-gaussian-splats/, fetched 2026-10-29 (unofficial)
