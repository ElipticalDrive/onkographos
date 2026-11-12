# Spatially Constrained Optimization

Also called: Opacity Loss with Re-initialization

## Overview
Spatially Constrained Optimization is a post-processing constraint workflow that applies an opacity loss to force full opacity rendering of target objects, followed by the re-initialization of boundary primitives that retain low opacity to clean up artifacts [S1, unofficial]. It sets itself apart from standard pruning methods by preserving transparent primitives via re-initialization rather than removal, ensuring full coverage and stabilizing optimization in complex regions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of primitives $\mathcal{G}$ with parameters $\Theta$, a target image $I^{\text{target}}$, and a target alpha mask $I_\alpha^{\text{target}}$.
*   **Output:** Optimized primitives $\Theta$ with enforced spatial constraints (full opacity for foreground, re-initialized boundary primitives).

**Data Structures:**
*   **Primitives:** Each primitive $i$ has opacity $\sigma(\nu_i)$, position, scale, rotation, and color.
*   **Buffers:** Standard rasterization buffers for forward/backward passes.

**Steps:**
1.  **Apply Opacity Loss:** During optimization, apply a loss function that penalizes deviations from the target alpha and target image, specifically weighting the image loss only where the target alpha is non-zero. This forces background primitives to become transparent and foreground primitives to become opaque [S1, unofficial].
    $$ \mathcal{L} = \| (I_\alpha^{\text{target}} > 0) \odot (I - I^{\text{target}}) \|_2^2 + \lambda_\alpha \| I_\alpha - I_\alpha^{\text{target}} \|_2^2 $$
    Where:
    *   $I_\alpha = 1 - T_N$ is the rendered alpha from Eq. (4) of the source [S1, unofficial].
    *   $I$ is the rendered image.
    *   $I^{\text{target}}$ is the target image.
    *   $I_\alpha^{\text{target}}$ is the target alpha mask.
    *   $\lambda_\alpha$ is a weighting hyperparameter.
    *   $\odot$ denotes element-wise multiplication.

2.  **Identify Low-Opacity Primitives:** Periodically (e.g., every `prune_iterations`), evaluate the opacity $\sigma(\nu_i)$ of each primitive. Identify primitives where $\sigma(\nu_i) < \text{prune threshold}$ [S1, unofficial]. These are typically located on the boundaries of the object or in regions where coverage is incomplete.

3.  **Re-initialize Boundary Primitives:** Instead of pruning (removing) these low-opacity primitives, re-initialize them randomly using the Structure-aware initialization method [S1, unofficial]. This involves:
    *   Marking the primitive for re-initialization.
    *   Sampling new parameters (position, scale, etc.) from a distribution defined by the Structure-aware initialization (Sec. 3.2.2 of the source) [S1, unofficial].
    *   Retaining the primitive in the set $\mathcal{G}$ to continue optimization.

**Parameters:**
*   **Prune Threshold:** The opacity threshold below which primitives are re-initialized. Typical value: 0.3 [S1, unofficial].
*   **Prune Iterations:** The frequency of re-initialization checks. Typical value: every 50 iterations [S1, unofficial].
*   **$\lambda_\alpha$:** Weight for the alpha loss term. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Destabilization:** Re-initialization is excluded during an initial warmup period and final iterations to prevent destabilizing the optimization [S1, unofficial].
*   **Boundary Artifacts:** Using opacity loss alone leaves low-opacity primitives on the object boundary; adding re-initialization produces the best qualitative results [S1, unofficial].
*   **Coverage Gaps:** Without structure-aware initialization or uniform noise canvas, primitives may prioritize covering areas distinct from a blending color, leaving parts of the canvas uncovered [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spatially_constrained_optimization(primitives, target_image, target_alpha, 
                                      prune_threshold=0.3, prune_iterations=50, 
                                      warmup_iterations=1000, total_iterations=5000, 
                                      lambda_alpha=1.0):
    """
    Perform spatially constrained optimization with re-initialization.
    """
    for iteration in range(total_iterations):
        # 1. Forward pass and loss computation
        rendered_image, rendered_alpha = forward_render(primitives)
        
        # Compute opacity loss
        # Mask for foreground: where target alpha is non-zero
        foreground_mask = (target_alpha > 0).float()
        
        # Image loss weighted by foreground mask
        image_loss = torch.sum((foreground_mask * (rendered_image - target_image)) ** 2)
        
        # Alpha loss
        alpha_loss = torch.sum((rendered_alpha - target_alpha) ** 2)
        
        total_loss = image_loss + lambda_alpha * alpha_loss
        
        # 2. Backward pass
        total_loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
        # 3. Periodic Re-initialization
        if iteration > warmup_iterations and iteration < total_iterations - 100: # Exclude final iterations
            if iteration % prune_iterations == 0:
                reinitialize_low_opacity_primitives(primitives, prune_threshold)

def reinitialize_low_opacity_primitives(primitives, threshold):
    """
    Re-initialize primitives with opacity below threshold.
    """
    for prim in primitives:
        if prim.opacity < threshold:
            # Re-initialize using Structure-aware initialization
            # This involves sampling new parameters from a specific distribution
            # defined in Sec 3.2.2 of the source paper
            new_params = structure_aware_initialization(prim)
            prim.update(new_params)
```

## Subtopics

### Structure-aware Re-initialization
A mechanism to handle transparent primitives in spatially constrained rendering by re-initializing them randomly instead of pruning them, to prevent destabilization and ensure coverage [S1, unofficial]. It differs from standard pruning (e.g., [34]) by re-initializing transparent primitives rather than removing them, maintaining the primitive count and potentially aiding coverage in complex regions [S1, unofficial].
*   **Definition:** Check opacity: For each primitive $i$, compute opacity $\sigma(\nu_i)$. If $\sigma(\nu_i) <$ prune threshold, mark for re-initialization [S1, unofficial].
*   **Step:** Re-initialize: Instead of removing the primitive, re-initialize it randomly using the Structure-aware initialization method (Sec. 3.2.2 of main paper) [S1, unofficial].
*   **Parameter:** Prune threshold: Threshold for opacity below which primitives are re-initialized. Typical value: 0.3 [S1, unofficial]. Prune iterations: Frequency of re-initialization. Typical value: every 50 iterations [S1, unofficial].
*   **Pitfall:** Re-initialization is excluded during an initial warmup period and final iterations to prevent destabilization of the optimization [S1, unofficial].

### Foreground-only Rendering
A method to render only foreground by optimizing primitive opacity to match target alpha, enabling applications like appearance editing [S1, unofficial].
*   **Definition:** Optimize $\Theta$ using loss: $\mathcal{L} = \| (I_{\alpha}^{target} > 0) \odot (I - I^{target}) \|_2^2 + \lambda_{\alpha} \| I_{\alpha} - I_{\alpha}^{target} \|_2^2$. Symbols: $I_{\alpha} = 1 - T_N \text{ from Eq. (4), } I^{target} \text{ is target image, } I_{\alpha}^{target} \text{ is target alpha, } \lambda_{\alpha} \text{ is weight.}$ [S1, unofficial].
*   **Step:** Re-initialize transparent primitives instead of pruning them for further applications [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Adds a periodic re-initialization step every `prune_iterations` (e.g., 50 iterations). The re-initialization involves sampling new parameters, which is $O(N)$ where $N$ is the number of primitives below the threshold. The loss computation is $O(N)$ per iteration.
*   **Memory Complexity:** Maintains the same number of primitives as the initial set, as re-initialized primitives are not removed. This can lead to higher VRAM usage compared to pruning methods if the initial primitive count is high [S1, unofficial].
*   **Measured Performance:** On an RTX 3090, per-iteration forward/backward runtime and VRAM usage vary by resolution and tile size. For 1024x1024 resolution with 16x16 tile size, forward runtime is 5514ms, backward is 9811ms, and VRAM is 12.0 GB [S1, unofficial]. For 512x512 with 16x16 tile, forward is 1360ms, backward is 2337ms, VRAM is 6.4 GB [S1, unofficial].
*   **Strengths:** Produces the best qualitative results by cleaning up boundary artifacts and ensuring full coverage [S1, unofficial]. Enables downstream applications like appearance editing by isolating foreground [S1, unofficial].
*   **Limitations:** Higher VRAM usage due to retained primitives. Requires careful tuning of warmup and final iteration exclusions to avoid destabilization [S1, unofficial].
*   **When to use:** When full opacity rendering of foreground objects is required and boundary artifacts are a concern. When downstream applications like appearance editing are needed.
*   **When not to use:** When memory is constrained and pruning is preferred to reduce primitive count. When real-time performance is critical and the overhead of re-initialization is unacceptable.

## Background
Originated in the DiffBMP paper (2026) by the authors of "DiffBMP: Differentiable Rendering with Bitmap Primitives" [S1, unofficial]. The idea is modified from [52]'s work for triangle mesh rendering [24], but differs by sampling gradients only once per iteration instead of five times [S1, unofficial]. It is used in differentiable rendering with bitmap primitives to support rendering only the foreground, enabling downstream applications such as appearance editing [S1, unofficial].

## Sources
[S1] DiffBMP: Differentiable Rendering with Bitmap Primitives, https://doi.org/10.48550/arXiv.2602.22625, fetched 2026-10-27 (unofficial)
