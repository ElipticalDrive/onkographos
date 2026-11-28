# NeRF Baseline

Also called: Original NeRF, Kerbl et al. baseline, Mip-NeRF precursor baseline

## Overview
The NeRF Baseline refers to the original training configuration and implementation of Neural Radiance Fields proposed by Mildenhall et al. [S2, unofficial]. It serves as a foundational reference for differentiable volume rendering, establishing the standard protocol for ray sampling, multi-resolution hashing (in later adaptations), and optimization loops. It is distinguished from its siblings, such as [DBARF](11583-dbarf.md), by its reliance on uniform or stratified sampling without explicit frequency-aware alignment or adaptive frequency modulation.

## How it works
The method optimizes a continuous volumetric scene representation parameterized by a multilayer perceptron (MLP) that maps 3D coordinates and viewing directions to volume density and view-dependent color.

**Inputs and Outputs**
- **Inputs:** A set of $N$ calibrated images with known camera poses $(R_i, t_i)$ and intrinsics.
- **Outputs:** A rendered image for a given camera pose, and the optimized MLP weights.

**Data Structures**
The scene is represented by an MLP with weights $W$. The input to the MLP is a 6D vector $(x, y, z, \theta, \phi)$, where $(x, y, z)$ is the 3D position and $(\theta, \phi)$ are spherical coordinates for the viewing direction.
- **Position Encoding:** In the original baseline, raw coordinates are used. In many modern reproductions (and as implied by the comparison context in [S1]), positional encoding or hash encoding may be applied, but the core "NeRF Baseline" definition relies on the MLP structure itself.
- **Density ($\sigma$):** Output by the MLP at intermediate layers.
- **Color ($c$):** Output by the MLP at the final layer.

**Numbered Steps**
1. **Ray Generation:** For each pixel in the target image, cast a ray $r(t) = o + td$, where $o$ is the camera origin and $d$ is the direction.
2. **Sampling:** Sample $N$ points along the ray within the near ($t_n$) and far ($t_f$) clipping planes.
   - **Coarse Phase:** Sample $N_c$ points. The sources specify $N_c = 64$ [S2, unofficial].
   - **Fine Phase:** Sample $N_f$ points. The sources specify $N_f = 128$ [S2, unofficial].
   - Sampling strategy is typically stratified or uniform within segments defined by the coarse pass.
3. **Volume Rendering:** Compute the expected color $C(r)$ and opacity $\alpha(t_i)$ for each sample point $t_i$.
   - Density $\sigma_i$ and color $c_i$ are queried from the MLP.
   - Transmittance $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, where $\delta_j = t_{j+1} - t_j$.
   - Weight $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$.
   - Rendered Color: $C(r) = \sum_{i=1}^{N} w_i c_i$.
4. **Loss Computation:** Compute the L2 loss between the rendered color and the ground truth pixel color.
5. **Optimization:** Update MLP weights using gradient descent.
   - Batch size: 2,048 rays per batch [S2, unofficial].
   - Resolution: 800 × 600 pixels [S2, unofficial].
   - Total steps: 200,000 solver steps [S2, unofficial].

**Formulas**
The volume rendering integral is discretized as:
$$ C(r) = \sum_{i=1}^{N} T_i \left(1 - e^{-\sigma_i \delta_i}\right) c_i $$
where $T_i = e^{-\sum_{j=1}^{i-1} \sigma_j \delta_j}$.

**Parameters**
- Coarse samples: 64 [S2, unofficial].
- Fine samples: 128 [S2, unofficial].
- Batch size: 2,048 rays [S2, unofficial].
- Training steps: 200,000 [S2, unofficial].
- Resolution: 800 × 600 [S2, unofficial].

**Edge Cases and Pitfalls**
- **Precision:** Floating-point precision in transmittance calculation can lead to underflow for deep scenes.
- **Boundaries:** Clipping planes must be carefully set to avoid sampling empty space or outside the scene bounds.
- **Degenerate Input:** Camera poses with zero baseline or identical views provide no geometric information for depth estimation.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_baseline_training(images, poses, steps=200000, batch_size=2048, 
                           coarse_samples=64, fine_samples=128):
    # Initialize MLP weights W
    W = initialize_mlp()
    
    for step in range(steps):
        # Sample a batch of rays
        rays = sample_rays(images, poses, batch_size)
        
        # Coarse pass
        t_coarse = stratified_sample(rays.t_n, rays.t_f, coarse_samples)
        sigma_coarse, color_coarse = query_mlp(W, rays.o, rays.d, t_coarse)
        
        # Compute weights for fine sampling
        weights_coarse = compute_weights(sigma_coarse, t_coarse)
        
        # Fine pass (importance sampling based on coarse weights)
        t_fine = importance_sample(t_coarse, weights_coarse, fine_samples)
        sigma_fine, color_fine = query_mlp(W, rays.o, rays.d, t_fine)
        
        # Render
        rendered_color = volume_render(t_fine, sigma_fine, color_fine)
        
        # Compute Loss
        loss = l2_loss(rendered_color, rays.gt_color)
        
        # Update
        W = optimize(W, loss)
        
    return W
```

## Subtopics

### Switch-NeRF++
A large-scale variant using a Mixture of Experts and hash encoding. It is 5x faster than Mega-NeRF and outperforms InstantNGP in efficiency [S1].

### Mega-NeRF
A large-scale NeRF method used as a baseline for comparison in [S1]. It uses a foreground/background network with hash encoding.

### INGP (InstantNGP)
Adapted to a large-scale setting in [S1]. It uses hash encoding and is comparable in training time to Switch-NeRF++ but with different architectural choices.

## Operating characteristics
- **Time Complexity:** Training time is approximately 0.40 seconds per iteration [S2, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Performance:** Rendering time is 32 seconds per frame [S2, unofficial].
- **Scaling:** The method scales to large scenes when combined with hash encoding (as in Mega-NeRF or INGP), but the original baseline is computationally intensive for large datasets [S1].
- **Strengths:** High-quality novel view synthesis; well-understood optimization landscape.
- **Limitations:** Slow training and rendering times compared to modern hash-based or grid-based methods [S1].
- **When to use:** As a baseline for comparison in research [S1].
- **When not to use:** When real-time rendering or fast training is required.

## Background
The NeRF Baseline originates from the paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" by Mildenhall et al. [S2, unofficial]. It was introduced as a method to synthesize novel views of complex scenes by optimizing a continuous volumetric scene function. The baseline configuration described in [S2, unofficial] is often used as a reference point for evaluating new methods in neural rendering.

## Sources
[S1] Learning Heterogeneous Mixture of Scene Experts for Large-Scale Neural Radiance Fields., https://doi.org/10.1109/tpami.2025.3603305, fetched 2026-10-26
[S2] Fast Training of Neural Lumigraph Representations using Meta Learning, https://doi.org/10.48550/arxiv.2106.14942, fetched 2026-10-26 (unofficial)
