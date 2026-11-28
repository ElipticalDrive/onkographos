# Bundle-Adjusted Neural Radiance Field (BARF)

Also called: BARF (Bundle-Adjusted Neural Radiance Field), Blending Artificially Realized Frequency, Bundle-Adjusting NeRF

## Overview
BARF is a method for jointly optimizing camera poses and the parameters of a Neural Radiance Field (NeRF) from a set of uncalibrated images, eliminating the need for external Structure-from-Motion (SfM) preprocessing [S1][S2, unofficial]. It distinguishes itself from other joint optimization approaches like NeRF++ [6198-nerf.md] or SCNeRF [103] by introducing a coarse-to-fine annealing schedule applied directly to the positional encoding frequencies, which stabilizes the optimization trajectory by aligning low-frequency image information before refining high-frequency details [S1][S2, unofficial][S3].

## How it works
BARF builds on [Neural Radiance Fields](4972-neural-radiance-fields.md) by treating camera poses as learnable parameters alongside the MLP weights. The core innovation is the temporal masking of positional encoding frequencies to decouple the optimization of geometry (high frequency) from pose (low frequency) [S1][S2, unofficial].

**Inputs and Outputs:**
*   **Inputs:** A set of $N$ images $\{I_i\}_{i=1}^N$ with initial, potentially inaccurate camera poses $P_i$ (extrinsics) and intrinsics $K$.
*   **Outputs:** Optimized camera poses $\hat{P}_i$, optimized NeRF MLP parameters $\theta$, and the rendered novel views.

**Data Structures:**
*   **NeRF MLP:** Standard 8-layer MLP mapping $(x, d)$ to $(\sigma, c)$ [S1].
*   **Positional Encoding (PE):** $\gamma(x) = [\sin(2^0 \pi x), \cos(2^0 \pi x), \dots, \sin(2^{L-1} \pi x), \cos(2^{L-1} \pi x)]$.
*   **Annealing Mask:** A binary mask $M_t \in \{0, 1\}^L$ for each frequency level $l \in \{0, \dots, L-1\}$ at training step $t$.

**Parameters:**
*   $L$: Number of frequency bands in PE (typically 10 in NeRF).
*   $T_{total}$: Total number of training steps.
*   $t$: Current training step.

**Algorithm Steps:**
1.  **Initialization:** Initialize NeRF MLP weights $\theta$ and camera poses $P_i$ using coarse SfM estimates or random initialization [S1].
2.  **Annealing Schedule:** For each training step $t$, compute the annealing mask $M_t$. The mask starts with only the lowest frequency band active (masking out higher frequencies) and gradually reveals higher frequencies [S2, unofficial].
    *   The mask $M_t$ is applied to the positional encoding function $\gamma$. If $M_t[l] = 0$, the $l$-th frequency component is zeroed out.
    *   This effectively acts as a low-pass filter on the input coordinates, forcing the network to learn coarse, low-frequency structures first [S3].
3.  **Ray Sampling:** For each image $i$, generate rays using the current pose $P_i$. Use naive dense ray sampling (uniform sampling along the ray) for simplicity [S1].
4.  **Volume Rendering:** Render the image $I_i$ using the standard NeRF volume rendering equation:
    $$ C(r) = \sum_{j=1}^J T_j \alpha_j c_j $$
    where $T_j = \exp(-\sum_{k=1}^{j-1} \sigma_k \delta_k)$, $\alpha_j = 1 - \exp(-\sigma_j \delta_j)$, and $\delta_j$ is the distance between samples [S1].
5.  **Loss Computation:** Compute the photometric loss between the rendered image $\hat{I}_i$ and the ground truth $I_i$:
    $$ \mathcal{L} = \frac{1}{N} \sum_{i=1}^N \| \hat{I}_i - I_i \|^2 $$
6.  **Joint Optimization:** Perform gradient descent to update both the NeRF parameters $\theta$ and the camera poses $P_i$. The gradient flows through the volume rendering equation to the poses [S2, unofficial].
7.  **Iteration:** Repeat steps 2-6 until convergence. As $t$ increases, higher frequencies are unmasked, allowing the network to refine details and the poses to align with high-frequency features [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Wide-baseline captures:** BARF struggles with wide-baseline 360-degree captures if initialized poorly, as the coarse-to-fine schedule may not recover from large initial pose errors [S2, unofficial].
*   **Rotational motion:** Standard NeRF struggles with rotational motion; BARF mitigates this but requires careful initialization [S1].
*   **Precision:** The annealing schedule must be smooth to avoid local minima. Abrupt changes in mask can destabilize training [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def barf_optimization(images, initial_poses, num_steps, num_freq_bands):
    # Initialize NeRF MLP and poses
    nerf_mlp = initialize_mlp()
    poses = initial_poses.clone()
    
    for t in range(num_steps):
        # Compute annealing mask: only frequencies below threshold are active
        # Threshold increases linearly or exponentially with t
        threshold = anneal_frequency(t, num_steps, num_freq_bands)
        mask = torch.zeros(num_freq_bands)
        mask[:threshold] = 1.0
        
        # Zero out gradients for poses if we want to freeze them initially?
        # BARF optimizes poses jointly, but the mask affects the input to MLP.
        
        total_loss = 0
        for i, image in enumerate(images):
            # Generate rays from current pose
            rays = generate_rays(image.shape, poses[i])
            
            # Sample points along rays
            points_3d = sample_points(rays, num_samples)
            
            # Apply positional encoding with mask
            # gamma(x) is the standard NeRF PE
            pe_input = apply_pe_mask(points_3d, mask)
            
            # Query NeRF
            sigma, color = nerf_mlp(pe_input, rays.directions)
            
            # Volume render
            rendered_color = volume_render(sigma, color, rays)
            
            # Compute loss
            loss = mse_loss(rendered_color, image)
            total_loss += loss
            
        # Backpropagate to update both MLP and poses
        total_loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
        # Update annealing schedule for next step
        # The mask is recomputed in the next iteration based on t+1
```

## Operating characteristics
*   **Time Complexity:** Similar to standard NeRF per iteration, but requires more iterations to converge due to the joint optimization of poses [S1].
*   **Memory Complexity:** Standard NeRF memory usage plus storage for pose parameters [S1].
*   **Performance:** On LLFF forward-facing scenes with unknown poses, BARF outperformed COLMAP-registered baseline NeRF by 0.45 PSNR and NeRF++ by 1.49 PSNR on average [S1].
*   **Strengths:** Eliminates the need for accurate SfM preprocessing; robust to moderate pose errors; smooth optimization trajectory [S1][S2, unofficial].
*   **Limitations:** Struggles with wide-baseline 360-degree captures; sensitive to initialization; slower convergence than fixed-pose NeRF [S2, unofficial].
*   **When to use:** When camera poses are unknown or inaccurate, and the scene is not extremely wide-baseline.
*   **When not to use:** For wide-baseline 360-degree captures where other methods like GNeRF [82] or SCNeRF [103] may be more suitable.

## Background
BARF was introduced by Lin et al. in April 2021 [S1]. It was developed concurrently with NeRF++ [6198-nerf.md] and SCNeRF [103] as a solution to the problem of joint pose and reconstruction optimization. The method draws inspiration from image alignment theory, using low-pass filtering to align low-frequency image information for pose optimization [S3]. It is part of the broader family of differentiable and inverse volume rendering methods that seek to bypass traditional SfM pipelines [S2, unofficial].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S3] Advancement in Neural Radiance Fields Propulsion for SLAM: A Novel Upgrade, https://www.atlantis-press.com/proceedings/icaic-24/126003439, fetched 2026-10-28
