# Spatially Constrained Optimization

Also called: Opacity Loss with Re-initialization

## Overview
Spatially Constrained Optimization is a post-processing constraint workflow that applies opacity loss to ensure full coverage of a target object, followed by the re-initialization of boundary primitives to clean up artifacts [S1, unofficial]. It solves the problem of incomplete object rendering and boundary artifacts that arise when optimizing for foreground-only appearance, distinguishing itself from standard pruning methods by retaining and reusing transparent primitives rather than removing them [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussian primitives $\mathcal{G} = \{g_1, \dots, g_N\}$ with parameters $\Theta$ (position, scale, rotation, opacity $\nu$, color), a target image $I^{\text{target}}$, and a target alpha mask $I_\alpha^{\text{target}}$.
*   **Output:** Optimized parameters $\Theta$ where foreground primitives have high opacity and background primitives are handled via re-initialization.

**Data Structures:**
*   **Primitives:** Each primitive $i$ has an opacity value $\nu_i$ (or $\sigma(\nu_i)$ after sigmoid activation).
*   **Thresholds:** `prune_threshold` (default 0.3) and `prune_iterations` (default 50).

**Steps:**
1.  **Compute Loss:** Calculate the composite loss $\mathcal{L}$ which penalizes differences in color where the target alpha is non-zero, and differences in alpha everywhere [S1, unofficial]:
    $$ \mathcal{L} = \| (I_\alpha^{\text{target}} > 0) \odot (I - I^{\text{target}}) \|_2^2 + \lambda_{\alpha} \| I_\alpha - I_\alpha^{\text{target}} \|_2^2 $$
    Where:
    *   $I$ is the rendered color image.
    *   $I_\alpha = 1 - T_N$ is the rendered alpha image, with $T_N$ being the accumulated transmittance [S1, unofficial].
    *   $\odot$ denotes element-wise multiplication.
    *   $\lambda_{\alpha}$ is a weighting hyperparameter.
2.  **Optimize:** Update parameters $\Theta$ using gradient descent to minimize $\mathcal{L}$ [S1, unofficial].
3.  **Identify Low-Opacity Primitives:** After optimization steps, identify primitives where the activated opacity $\sigma(\nu_i)$ is below the `prune_threshold` [S1, unofficial].
4.  **Re-initialize:** Instead of removing these primitives, re-initialize their parameters randomly using the Structure-aware initialization method [S1, unofficial]. This involves sampling new positions, scales, and rotations to cover the canvas, ensuring full coverage [S1, unofficial].
5.  **Iterate:** Repeat steps 1-4 every `prune_iterations` (e.g., 50 iterations), excluding an initial warmup period and final iterations to prevent destabilization [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spatially_constrained_optimization(primitives, target_image, target_alpha, 
                                      lambda_alpha=1.0, prune_threshold=0.3, 
                                      prune_iterations=50, warmup_iters=1000, 
                                      total_iters=10000):
    """
    Optimizes primitives with spatial constraints and re-initialization.
    """
    for i in range(total_iters):
        # 1. Forward pass and Loss computation
        rendered_color, rendered_alpha = render(primitives, target_image)
        
        # Mask for color loss: only where target alpha is present
        color_mask = (target_alpha > 0).float()
        
        # Loss components
        color_loss = torch.sum((color_mask * (rendered_color - target_image))**2)
        alpha_loss = torch.sum((rendered_alpha - target_alpha)**2)
        
        total_loss = color_loss + lambda_alpha * alpha_loss
        
        # 2. Backward pass and update
        total_loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
        # 3. Re-initialization check
        if i > warmup_iters and i < total_iters - 1000: # Exclude final iterations
            if i % prune_iterations == 0:
                reinitialize_low_opacity_primitives(primitives, prune_threshold)

def reinitialize_low_opacity_primitives(primitives, threshold):
    """
    Re-initializes primitives with opacity below threshold.
    """
    for prim in primitives:
        # Compute activated opacity
        opacity = sigmoid(prim.opacity_param)
        
        if opacity < threshold:
            # Re-initialize using Structure-aware method
            # This typically involves random sampling within the scene bounds
            # or based on the canvas structure to ensure coverage
            prim.position = random_uniform_sample(scene_bounds)
            prim.scale = random_uniform_sample(scale_range)
            prim.rotation = random_uniform_rotation()
            prim.opacity_param = initial_opacity_value # Reset opacity parameter
```

**Parameters:**
*   `prune_threshold`: Threshold for opacity below which primitives are re-initialized. Typical value: 0.3 [S1, unofficial].
*   `prune_iterations`: Frequency of re-initialization. Typical value: every 50 iterations [S1, unofficial].
*   `lambda_alpha`: Weight for the alpha loss term. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Destabilization:** Re-initialization is excluded during an initial warmup period and final iterations to prevent destabilization of the optimization [S1, unofficial].
*   **Boundary Artifacts:** Using opacity loss alone leaves low-opacity primitives on the boundary of the object; adding re-initialization produces the best qualitative results [S1, unofficial].

## Subtopics

### Structure-aware Re-initialization
A mechanism to handle transparent primitives in spatially constrained rendering by re-initializing them randomly instead of pruning them, to prevent destabilization and ensure coverage [S1, unofficial]. It differs from standard pruning (e.g., [34]) by re-initializing transparent primitives rather than removing them, maintaining the primitive count and potentially aiding coverage in complex regions [S1, unofficial]. It uses a `prune_threshold` of 0.3 and occurs every 50 iterations, excluding warmup and final iterations [S1, unofficial].

### Foreground-only Rendering
A method to render only foreground by optimizing primitive opacity to match target alpha, enabling applications like appearance editing [S1, unofficial]. It uses the loss $\mathcal{L} = \| (I_\alpha^{\text{target}} > 0) \odot (I - I^{\text{target}}) \|_2^2 + \lambda_{\alpha} \| I_\alpha - I_\alpha^{\text{target}} \|_2^2$ and re-initializes transparent primitives instead of pruning them [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Adds overhead due to periodic re-initialization and loss computation. The re-initialization step is $O(N)$ where $N$ is the number of primitives.
*   **Memory Complexity:** Maintains the primitive count, so memory usage is similar to the base Gaussian Splatting method, unlike pruning which reduces it.
*   **Performance:** On an RTX 3090, the custom CUDA implementation achieves 1360ms forward / 2337ms backward per iteration at 512x512 resolution with 6.4 GB VRAM [S1, unofficial].
*   **Strengths:** Produces best qualitative results for foreground-only rendering by cleaning up boundary artifacts [S1, unofficial]. Enables downstream applications like appearance editing [S1, unofficial].
*   **Limitations:** Does not reduce memory usage as effectively as pruning methods. Re-initialization can destabilize optimization if not properly scheduled [S1, unofficial].

## Background
Originated in the DiffBMP paper (2026) as a modification of triangle mesh rendering work [52] [S1, unofficial]. It was developed to support rendering only the foreground, enabling downstream applications such as appearance editing [S1, unofficial]. The method combines opacity loss with a re-initialization mechanism to handle transparent primitives, differing from standard pruning approaches [S1, unofficial].

## Sources
[S1] DiffBMP: Differentiable Rendering with Bitmap Primitives, https://doi.org/10.48550/arXiv.2602.22625, fetched 2026-10-27 (unofficial)
