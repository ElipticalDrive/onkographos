# Progressive Training Strategy

Also called: Coarse-to-Fine Training, Progressive Growing, Progressive Growing Strategy, Progressive Growing of GANs, Progressive training (Growing)

## Overview
Progressive training is a method that gradually increases the resolution of the rendered output during optimization to stabilize convergence and prevent geometric artifacts. It solves the problem of complex loss landscapes and sparse initialization errors that occur when training directly at high resolution [S1, unofficial][S2]. It differs from standard full-resolution training by deferring the fitting of fine features until the global geometry is established [S2], and differs from simple two-stage pipelines by potentially using continuous or multi-stage resolution scaling rather than a single abrupt switch [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Training dataset (images/points), total iteration budget $T_{total}$, initial resolution scale $s_{start}$, target resolution scale $s_{end}=1.0$.
*   **Output:** Trained model parameters (neural network weights or Gaussian attributes) optimized for the target resolution.

**Data Structures:**
*   **Resolution Schedule:** A mapping from iteration $t$ to output image dimensions $(H_t, W_t)$.
*   **Model State:** The generator/discriminator architecture or Gaussian point cloud state, which may grow in capacity or remain fixed but render at varying scales.

**Steps:**
1.  **Initialize:** Set current resolution scale $s = s_{start}$ (e.g., 0.3 or low-res dimensions). Initialize model parameters.
2.  **Iterate:** For each training iteration $t$ from 1 to $T_{total}$:
    a.  **Determine Resolution:** Calculate current scale $s_t$.
        *   If using **Progressive Growing** (StyleNeRF):
            *   Stage 1 ($t \le T_1$): Render at fixed low resolution.
            *   Stage 2 ($T_1 < t \le T_2$): Linearly interpolate resolution from low to high.
            *   Stage 3 ($t > T_2$): Render at full target resolution.
        *   If using **Cosine Scaling** (EAGLES):
            *   If $t \le 0.7 \cdot T_{total}$: $s_t = 0.3 + 0.7 \cdot \frac{1}{2}(1 + \cos(\pi \frac{t}{0.7 T_{total}}))$.
            *   If $t > 0.7 \cdot T_{total}$: $s_t = 1.0$.
    b.  **Render:** Generate image at resolution determined by $s_t$.
    c.  **Compute Loss:** Calculate loss against ground truth images at the current resolution.
    d.  **Update:** Perform gradient descent to update model parameters.
    e.  **Stabilize (Progressive Growing only):** If adding new layers/resolution levels, linearly interpolate the output with the previous low-resolution output to stabilize training due to random initialization of new parameters [S1, unofficial].
3.  **Finalize:** Continue training at full resolution until $T_{total}$ is reached.

**Formulas:**
*   **Linear Interpolation (StyleNeRF Stage 2):**
    $$ s_t = s_{low} + (s_{high} - s_{low}) \cdot \frac{t - T_1}{T_2 - T_1} $$
    Where $s_{low}$ is the initial low resolution scale, $s_{high}=1.0$, $T_1$ is the start of Stage 2, and $T_2$ is the end of Stage 2 [S1, unofficial].
*   **Cosine Schedule (EAGLES):**
    $$ s_t = s_{start} + (s_{end} - s_{start}) \cdot \frac{1}{2} \left( 1 + \cos \left( \pi \frac{t}{T_{scale}} \right) \right) $$
    Where $T_{scale} = 0.7 \cdot T_{total}$ [S2].
*   **Output Interpolation (Stabilization):**
    $$ I_{out} = \alpha \cdot I_{new} + (1 - \alpha) \cdot I_{low} $$
    Where $\alpha$ is the interpolation factor corresponding to the progress of layer/resolution addition [S1, unofficial].

**Parameters:**
*   **$T_1$ (StyleNeRF):** Start of Stage 2. Default: 500,000 images [S1, unofficial].
*   **$T_2$ (StyleNeRF):** End of Stage 2. Default: 5,000,000 images [S1, unofficial].
*   **$T_3$ (StyleNeRF):** End of Stage 3. Default: 25,000,000 images [S1, unofficial].
*   **$s_{start}$ (EAGLES):** Initial scale factor. Default: 0.3 [S2].
*   **$T_{scale}$ (EAGLES):** Duration of scaling phase. Default: 70% of total iterations [S2].
*   **Batch Size:** 64 images per batch [S1, unofficial].
*   **Total Iterations (EAGLES):** 30,000 iterations (configurable based on budget) [S2].

**Edge Cases and Pitfalls:**
*   **Geometry Failure:** Directly training at the highest resolution causes the model to fail to capture object geometry because the rendering equations are approximations [S1, unofficial].
*   **Floating Artifacts:** Standard full-resolution training forces Gaussians to fit fine features early with sparse SfM initialization, leading to suboptimal optimization and floating artifacts [S2].
*   **Small Datasets:** If the dataset is too small (e.g., MetFaces), progressive training may be unstable; fine-tuning from a pretrained high-resolution model is recommended instead [S1, unofficial].
*   **New Layer Initialization:** Parameters of newly added layers are random; without linear interpolation with low-resolution output, training becomes unstable [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_training(model, dataset, total_iterations, strategy="cosine"):
    """
    model: The neural network or Gaussian system to train.
    dataset: Iterable of ground truth images.
    total_iterations: Total number of training steps.
    strategy: "cosine" (EAGLES) or "linear" (StyleNeRF).
    """
    if strategy == "cosine":
        # EAGLES configuration
        scale_start = 0.3
        scale_end = 1.0
        scale_duration = int(0.7 * total_iterations)
        
        for t in range(1, total_iterations + 1):
            # Calculate current resolution scale
            if t <= scale_duration:
                # Cosine annealing from 0.3 to 1.0
                progress = t / scale_duration
                scale = scale_start + (scale_end - scale_start) * 0.5 * (1 + math.cos(math.pi * progress))
            else:
                scale = scale_end
            
            # Determine current image dimensions based on scale
            current_height = int(base_height * scale)
            current_width = int(base_width * scale)
            
            # Get batch of ground truth images
            gt_images = get_next_batch(dataset, batch_size=64)
            
            # Resize ground truth to current resolution if necessary
            # Or render model at current resolution
            rendered_images = model.render(resolution=(current_height, current_width))
            
            # Compute loss
            loss = compute_loss(rendered_images, gt_images)
            
            # Update model
            model.update(loss)
            
            # Optional: Densification/Pruning steps specific to Gaussians
            if t % 175 == 0:
                model.densify()
            if t % 2500 == 0:
                model.reset_opacity()
            if t % 5000 == 0:
                model.prune(0.15)

    elif strategy == "linear":
        # StyleNeRF configuration
        T1 = 500000
        T2 = 5000000
        T3 = 25000000
        low_res_scale = 0.1 # Example low res, e.g., 32x32 relative to 256x256
        high_res_scale = 1.0
        
        for t in range(1, total_iterations + 1):
            if t <= T1:
                # Stage 1: Low resolution
                scale = low_res_scale
            elif t <= T2:
                # Stage 2: Linear interpolation
                progress = (t - T1) / (T2 - T1)
                scale = low_res_scale + (high_res_scale - low_res_scale) * progress
            else:
                # Stage 3: Full resolution
                scale = high_res_scale
            
            current_height = int(base_height * scale)
            current_width = int(base_width * scale)
            
            gt_images = get_next_batch(dataset, batch_size=64)
            
            # Progressive Growing Stabilization
            # If layers were added recently, interpolate with low-res output
            if t <= T2:
                rendered_images = model.render(resolution=(current_height, current_width))
                low_res_rendered = model.render(resolution=(int(base_height * low_res_scale), int(base_width * low_res_scale)))
                # Interpolate to stabilize new layers
                final_render = interpolate(rendered_images, low_res_rendered, progress)
            else:
                final_render = model.render(resolution=(current_height, current_width))
            
            loss = compute_loss(final_render, gt_images)
            model.update(loss)
```

## Subtopics

### Progressive Growing
Starts with a shallow network and fades in layers for higher resolutions. Requires linear interpolation with low-resolution output to stabilize random new parameters. Chosen for speed advantage at low resolutions [S1, unofficial].

### Progressive Up-sampling
Inserts upsample operations during training instead of growing layers. Does not require fading in layers linearly. Works similarly in visual quality but may be slower at low resolutions [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time is distributed across resolutions. The majority of iterations (e.g., 70% in EAGLES, or up to $T_2$ in StyleNeRF) occur at lower resolutions, which are computationally cheaper per iteration. However, the total iteration count may be higher to compensate for the coarse phases.
*   **Memory Complexity:** Lower resolution rendering requires less memory per iteration. The model architecture size may grow in Progressive Growing, but up-sampling keeps it fixed.
*   **Performance:** Prevents failure to capture geometry and reduces floating artifacts compared to direct high-resolution training [S1, unofficial][S2].
*   **Strengths:** Stabilizes optimization in complex loss landscapes; handles sparse SfM initialization better; avoids early fitting to noise/fine details [S2].
*   **Limitations:** Requires careful scheduling of resolution transitions; small datasets may not support stable progressive training [S1, unofficial].
*   **When to use:** High-resolution synthesis, scenes with sparse initialization, or when direct training leads to artifacts.
*   **When not to use:** Very small datasets (use fine-tuning instead) [S1, unofficial].

## Background
The strategy is inspired by Karras et al. (2017) "Progressive Growing of GANs for Improved Quality, Stability, and Variation" [S1, unofficial]. It was adapted for NeRF-based methods (StyleNeRF) and 3D Gaussian Splatting variants (EAGLES) to address the specific challenges of volume rendering and differentiable rendering, where direct high-resolution training leads to geometric inaccuracies due to approximation errors in the rendering equations [S1, unofficial][S2].

## Sources
[S1] StyleNeRF: A Style-based 3D-Aware Generator for High-resolution Image Synthesis, https://doi.org/10.48550/arxiv.2110.08985, fetched 2026-10-26 (unofficial)
[S2] EAGLES: Efficient Accelerated 3D Gaussians with Lightweight EncodingS, https://doi.org/10.1007/978-3-031-73036-8_4, fetched 2026-10-27
