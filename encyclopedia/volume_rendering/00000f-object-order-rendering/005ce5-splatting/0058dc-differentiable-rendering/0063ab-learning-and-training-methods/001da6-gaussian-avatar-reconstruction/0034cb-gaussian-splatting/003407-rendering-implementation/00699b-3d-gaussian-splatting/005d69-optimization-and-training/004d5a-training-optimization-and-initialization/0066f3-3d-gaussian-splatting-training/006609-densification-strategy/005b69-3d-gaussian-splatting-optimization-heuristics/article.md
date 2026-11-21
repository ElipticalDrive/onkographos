# 3D Gaussian Splatting Optimization Heuristics

Also called: 3DGS densification, 3DGS pruning

## Overview
This method defines the heuristic-driven densification and pruning strategies used during the training of 3D Gaussian Splatting (3DGS) to dynamically adjust the number and distribution of Gaussian primitives [S1, unofficial]. It solves the problem of static representation capacity by allowing the model to increase density in under-reconstructed regions (via cloning/splitting) and remove redundant primitives (via pruning) based on gradient magnitude, opacity, and scale [S1, unofficial]. It is distinguished from sibling methods like [3DGS-MCMC](7852-3dgs-mcmc.md) or [Adaptive Gaussian Refinement](11448-adaptive-gaussian-refinement.md) by its reliance on simple, periodic heuristic thresholds rather than complex probabilistic sampling or continuous adaptive learning rates [S1, unofficial].

## How it works
The method operates on a set of 3D Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$, where each $g_k$ is defined by mean position $\mu_k$, covariance $\Sigma_k$ (parameterized by scale $s_k$ and rotation $q_k$), opacity $\alpha_k$, and spherical harmonic coefficients $\mathbf{c}_k$ [Parent]. The optimization uses the Adam optimizer with an L1 loss between rendered and target images, often combined with a Structural Similarity Index Measure (SSIM) loss [S1, unofficial].

### Data Structures
*   **Gaussian Buffer:** A contiguous array of Gaussian primitives. Each primitive stores:
    *   $\mu_k \in \mathbb{R}^3$: Position.
    *   $s_k \in \mathbb{R}^3$: Scale (log-space or linear, depending on implementation).
    *   $q_k \in \mathbb{R}^4$: Rotation (quaternion).
    *   $\alpha_k \in \mathbb{R}$: Opacity.
    *   $\mathbf{c}_k$: Spherical Harmonic coefficients (view-dependent color).
    *   $\tau_k \in \mathbb{R}$: Accumulated positional gradient magnitude (buffer for heuristic evaluation).
    *   `active`: Boolean flag indicating if the Gaussian is currently valid (used for pruning).

### Parameters
*   `densify_interval`: Number of iterations between densification steps. Typical value: 100 [Parent] or 400 [S1, unofficial].
*   `prune_interval`: Number of iterations between pruning steps. Often aligned with `densify_interval` or set to 3000 for alpha reset [S1, unofficial].
*   `alpha_reset_interval`: Interval for resetting opacities to zero. Typical value: 3000 [S1, unofficial].
*   $\epsilon_{\text{trans}}$: Transparency threshold for pruning. Gaussians with $\alpha_k < \epsilon_{\text{trans}}$ are removed. Value: Not given in the sources.
*   $s_{\text{max}}$: Maximum scale threshold for pruning. Gaussians with $\max(s_k) > s_{\text{max}}$ are removed. Value: Not given in the sources.
*   $N_{\text{clone}}$: Number of clones per split/clone operation. Typical value: 2 [S1, unofficial].

### Steps
1.  **Optimization Step:** Perform standard gradient descent update for $\mu_k, s_k, q_k, \alpha_k, \mathbf{c}_k$ using Adam optimizer [S1, unofficial].
2.  **Gradient Accumulation:** Accumulate the L2 norm of the positional gradient for each Gaussian across visible views:
    $$ \tau_k = \sum_{v \in \text{views}} \left\| \frac{\partial L}{\partial \mu_k} \right\|_2 $$
    [S1, unofficial][Parent].
3.  **Periodic Densification (every `densify_interval` iterations):**
    *   **Cloning (Under-reconstruction):** For each Gaussian $g_k$ with high positional gradient $\tau_k > \tau_{\text{high}}$:
        *   Create $N_{\text{clone}}$ clones of $g_k$.
        *   The clones are positioned along the positional gradient direction to better cover the geometry [S1, unofficial].
        *   The original $g_k$ is removed or retained depending on implementation specifics (typically replaced by children) [S1, unofficial].
    *   **Splitting (Over-reconstruction):** For each Gaussian $g_k$ with high positional gradient $\tau_k > \tau_{\text{high}}$ and large scale:
        *   Split $g_k$ into $N_{\text{clone}}$ children.
        *   Children means $\mu_{child}$ are sampled from $\mathcal{N}(\mu_k, \Sigma_k)$ [Parent].
        *   Scales are reduced: $s_{child} = s_k \cdot \frac{5}{4} \cdot \frac{1}{\sqrt{N_{\text{clone}}}}$ [Parent].
        *   Opacity $\alpha_{child}$ is halved to preserve total opacity mass [Parent].
4.  **Periodic Pruning (every `prune_interval` iterations):**
    *   Remove any Gaussian $g_k$ where $\alpha_k < \epsilon_{\text{trans}}$ [S1, unofficial].
    *   Remove any Gaussian $g_k$ where $\max(s_k) > s_{\text{max}}$ [S1, unofficial].
5.  **Alpha Reset (every `alpha_reset_interval` iterations):**
    *   Set $\alpha_k = 0$ for all Gaussians [S1, unofficial].
    *   This forces the optimizer to re-evaluate the importance of each Gaussian, allowing the pruning step to remove irrelevant primitives that were previously maintained by non-zero opacity [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_gaussians(gaussians, views, loss_fn, optimizer, iteration):
    # Standard optimization step
    for g in gaussians:
        if g.active:
            # Render and compute loss
            rendered_image = render(gaussians, views)
            loss = loss_fn(rendered_image, target_image)
            # Backpropagate
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()
            
            # Accumulate positional gradient magnitude
            g.grad_accum += torch.norm(g.grad_position)

    # Densification and Pruning
    if iteration % densify_interval == 0:
        densify(gaussians)
        
    if iteration % prune_interval == 0:
        prune(gaussians)
        
    if iteration % alpha_reset_interval == 0:
        reset_alpha(gaussians)

def densify(gaussians):
    high_grad_threshold = 0.01 # Placeholder value
    for g in gaussians:
        if not g.active: continue
        if g.grad_accum > high_grad_threshold:
            if g.scale_max > scale_threshold:
                # Splitting: Over-reconstruction
                children = split_gaussian(g)
                gaussians.extend(children)
                g.active = False
            else:
                # Cloning: Under-reconstruction
                clones = clone_gaussian(g)
                gaussians.extend(clones)
                g.active = False
    # Compact buffer or mark inactive
    gaussians = [g for g in gaussians if g.active]

def prune(gaussians):
    for g in gaussians:
        if not g.active: continue
        if g.opacity < epsilon_trans or g.scale_max > scale_threshold:
            g.active = False
    gaussians = [g for g in gaussians if g.active]

def reset_alpha(gaussians):
    for g in gaussians:
        if g.active:
            g.opacity = 0.0
```

### Edge Cases and Pitfalls
*   **Precision:** Gradient accumulation must be normalized by the number of views to avoid bias towards frequently viewed Gaussians [Parent].
*   **Boundaries:** Pruning must be careful not to remove Gaussians that are temporarily low-opacity but critical for geometry [S1, unofficial]. The alpha reset helps mitigate this by forcing re-evaluation [S1, unofficial].
*   **Degenerate Input:** If the scene has no geometry, all Gaussians may be pruned. A minimum number of Gaussians should be maintained [G].

## Operating characteristics
*   **Time Complexity:** Densification and pruning are $O(N)$ operations, where $N$ is the number of Gaussians. The overhead is minimal compared to the rendering and optimization steps [S1, unofficial].
*   **Memory Complexity:** Memory usage scales with the number of Gaussians. Pruning reduces memory footprint, while densification increases it. Buffer compaction is required to maintain contiguous memory access [Parent].
*   **Measured Performance:** The method enables high-quality novel view synthesis with fewer primitives than static methods, reducing memory usage while maintaining visual fidelity [S1, unofficial].
*   **Strengths:** Simple to implement, effective at adapting to scene complexity, avoids "observable blobs" from isotropic Gaussians by using anisotropic splits [S1, unofficial].
*   **Limitations:** Heuristic thresholds require tuning. The alpha reset can cause temporary visual artifacts if not handled smoothly [S1, unofficial].
*   **When to Use:** When training 3DGS models and dynamic adaptation of primitive density is required.
*   **When Not to Use:** When real-time inference is the primary concern and training-time optimization is not needed.

## Background
The method originates from the 3D Gaussian Splatting paper by Kerbl et al. (2023) [S1, unofficial]. It was developed to address the limitations of static point cloud representations in neural radiance fields. The use of anisotropic Gaussians and heuristic-based densification/pruning was a key innovation that enabled real-time rendering of complex scenes [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting | Guest Lecture for "Computer Graphics in the AI Era", https://www.youtube.com/watch?v=MBVmQSA24Yk, fetched 2026-10-27 (unofficial)
