# Parameter-space ReSTIR

Also called: GRIS for derivatives, PGRIS, Parameter-Space Differentiable Rendering, Parameter-centric reservoirs

## Overview
Parameter-space ReSTIR is a differentiable rendering formulation that computes gradients with respect to scene parameters (e.g., textures) directly in parameter space rather than image space [S1]. It solves the memory bottleneck of standard reservoir-based differentiable rendering, where one reservoir is required per pixel per parameter, by reformulating the derivative estimation as a single integral over the parameter space [S1]. This allows the method to store only one reservoir per parameter (texel), accumulating derivative estimates from all relevant pixels, thereby reducing storage complexity from $O(\text{pixels} \times \text{parameters})$ to $O(\text{parameters})$ [S1]. It builds on [Differentiable volume rendering](43-differentiable-volume-rendering.md) by enabling efficient sample reuse across optimization iterations [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene parameters (e.g., texture maps defining density $\sigma$ or color $\mathbf{c}$), camera rays, and a loss function $L$ dependent on rendered image $\hat{C}$.
*   **Outputs:** Gradient estimates $\nabla_\theta L$ with respect to the scene parameters $\theta$.

**Data Structures:**
The core data structure is the **Parameter-space Reservoir**, which stores a single selected sample and its associated weights [S1].

**Reservoir Layout:**
The `Reservoir` struct contains five fields [S1]:
1.  `x`: The selected sample (e.g., a path or sample index).
2.  `wsum`: The sum of weights, calculated as $\sum (q/p)$, where $q$ is the target distribution and $p$ is the source distribution.
3.  `M`: The number of candidates or confidence weight.
4.  `W`: The unbiased contribution weight.
5.  `lock`: A synchronization lock to handle concurrent updates from multiple pixels [S1].

**Algorithm Steps:**
1.  **Candidate Generation:** Generate paths from the camera. To avoid wasting computation on invisible texels, candidates are generated only for texels visible from the camera [S1].
2.  **Reservoir Update:** For each pixel, compute the derivative contribution of each candidate sample with respect to the texels it intersects. Update the corresponding texel's reservoir with these candidates [S1].
    *   The update uses the reservoir resampling rule: a new candidate $x_s$ with weight $w_s$ replaces the current reservoir sample $x$ with probability $w_s / w_{sum}$ [S1].
3.  **Reuse Across Iterations:** At optimization iteration $v+1$, if a new sample is occluded or poor, the algorithm can reuse the sample stored in the reservoir from iteration $v$ [S1]. This allows obtaining unoccluded samples by leveraging previous iterations' results [S1].
4.  **Gradient Estimation:** Use the final selected sample from each texel's reservoir to compute the derivative of the loss $L$ with respect to the parameter [S1]. The weight $w$ characterizes how the change in parameter affects the loss [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Reservoir:
    def __init__(self):
        self.x = 0          # Selected sample
        self.wsum = 0.0     # Sum of weights (q / p)
        self.M = 0          # Number of candidates / confidence weight
        self.W = 0.0        # Unbiased contribution weight
        self.lock = 0       # Lock to synchronize the reservoir update

    def update(self, x_s, w_s):
        """
        Update reservoir with a new candidate sample x_s and weight w_s.
        """
        # Acquire lock for thread safety [S1]
        acquire_lock(self.lock)
        
        self.wsum += w_s
        
        # Reservoir resampling step: select new sample with probability proportional to weight
        if random() < (w_s / self.wsum):
            self.x = x_s
            
        # Update M and W as per RIS/GRIS theory [S1]
        # Note: Exact update rules for M and W depend on the specific RIS variant (e.g., one-sample RIS)
        # For one-sample RIS, M is typically incremented, and W is updated to maintain unbiasedness.
        self.M += 1
        self.W = self.W + w_s - (self.W * w_s / self.wsum) # Placeholder for exact GRIS weight update [G]
        
        release_lock(self.lock)

def parameter_space_restir(scene_params, rays, loss_fn, num_iterations):
    """
    Main loop for Parameter-space ReSTIR.
    """
    # Initialize one reservoir per parameter (texel) [S1]
    # Assuming texels are indexed by t
    reservoirs = {t: Reservoir() for t in scene_params.texels}
    
    for v in range(num_iterations):
        # 1. Generate paths from camera [S1]
        paths = generate_paths(rays, scene_params)
        
        # 2. Update reservoirs with derivative contributions [S1]
        for path in paths:
            # Identify texels intersected by the path
            intersected_texels = path.intersect_texels()
            
            for t in intersected_texels:
                # Compute derivative contribution of this path to texel t
                # w_s characterizes how the change in parameter affects the loss L [S1]
                w_s = compute_derivative_contribution(path, t, loss_fn)
                
                # Update the reservoir for texel t
                reservoirs[t].update(path.sample, w_s)
                
        # 3. Reuse samples across iterations [S1]
        # If a new sample is occluded, reuse the reservoir's stored sample from iteration v-1
        # This is handled implicitly by the reservoir update logic retaining the best sample
        # or explicitly by checking occlusion before update [S1]
        
        # 4. Compute gradients using final samples [S1]
        gradients = {}
        for t in scene_params.texels:
            final_sample = reservoirs[t].x
            # Compute gradient based on the final selected sample
            gradients[t] = compute_gradient(final_sample, t, loss_fn)
            
        # Update scene parameters using gradients
        scene_params.update(gradients)
```

**Edge Cases and Pitfalls:**
*   **Invisible Texels:** Generating candidates for texels not visible from the camera is wasteful as their contribution to the loss is zero [S1]. Visibility culling is required.
*   **Concurrency:** Multiple pixels may update the same texel's reservoir simultaneously, requiring synchronization locks [S1].
*   **Precision:** Accumulating weights in `wsum` can lead to precision issues if weights vary significantly in magnitude [G].

## Subtopics

### Parameter-space Reservoir
A data structure for storing a single selected sample and its associated weights to support reservoir resampling in parameter space [S1]. It allows for efficient merging and updating of candidate samples across iterations [S1]. It is chosen when memory usage must be minimized by sharing derivative estimates across pixels for a single parameter [S1]. See [Parameter-space ReSTIR](1373-parameter-space-restir.md).

## Operating characteristics
*   **Memory Complexity:** $O(\text{parameters})$ or $O(\text{texels})$, compared to $O(\text{pixels} \times \text{parameters})$ for per-pixel reservoirs [S1].
*   **Time Complexity:** Depends on the number of paths and texels. Candidate generation is limited to visible texels [S1].
*   **Strengths:** Significant memory reduction for inverse rendering with many parameters; enables sample reuse across optimization iterations [S1].
*   **Limitations:** Requires synchronization for concurrent updates; visibility culling is necessary to avoid wasted computation [S1].
*   **When to use:** Inverse rendering problems with large texture maps or parameter sets where memory is a bottleneck [S1].

## Background
Parameter-space ReSTIR was introduced by Chang, Sivaram, Nowrouzezahrai, Hachisuka, Ramamoorthi, and Li [S1]. It extends Generalized Resampled Importance Sampling (GRIS) [Lin et al. 2022] to differentiable rendering [S1]. It builds on the principles of ReSTIR [Talbot et al. 2005] and applies them to the estimation of derivatives in inverse rendering [S1].

## Sources
[S1] Parameter-space ReSTIR for Differentiable and Inverse Rendering, https://doi.org/10.1145/3588432.3591512, fetched 2026-10-27
