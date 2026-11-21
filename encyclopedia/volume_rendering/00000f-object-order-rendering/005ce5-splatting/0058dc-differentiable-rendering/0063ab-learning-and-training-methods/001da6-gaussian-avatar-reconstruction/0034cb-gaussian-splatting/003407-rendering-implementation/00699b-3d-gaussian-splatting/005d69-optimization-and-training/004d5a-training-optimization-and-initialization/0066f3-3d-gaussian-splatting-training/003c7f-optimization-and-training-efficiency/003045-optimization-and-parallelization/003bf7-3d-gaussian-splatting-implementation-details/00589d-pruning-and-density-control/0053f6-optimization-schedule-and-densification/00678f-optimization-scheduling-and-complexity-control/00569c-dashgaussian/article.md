# DashGaussian

Also called: Computational resource allocation for 3DGS

## Overview
DashGaussian is a scheduling scheme for 3D Gaussian Splatting (3DGS) that accelerates optimization by reformulating the process as a frequency component fitting task [S1]. It cooperatively allocates rendering resolution and primitive count to strip redundant complexity, distinguishing itself from vanilla 3DGS and other backbones by producing a concave-up primitive growth curve that spans the entire optimization process [S1]. Unlike methods that peak in primitive count at half-iteration or use concave-down curves, DashGaussian synchronizes primitive growth with increasing resolution to fit high-frequency details only after low-frequency structures are established [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of training views, a base 3DGS backbone (e.g., Taming-3DGS, Mip-Splatting), and an optimization timeline $T$.
*   **Output:** Optimized Gaussian primitives (position, scale, rotation, opacity, color) and a rendered scene with quality preserved or improved over the baseline [S1].

**Data Structures**
The method operates on the standard 3DGS primitive representation defined in [Parent]:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$
*   Scale $\mathbf{s}_i \in \mathbb{R}^3$
*   Rotation $\mathbf{q}_i \in \mathbb{R}^4$ (quaternion)
*   Opacity $\alpha_i \in [0, 1]$
*   Color coefficients $\mathbf{c}_i$ (Spherical Harmonics)

The core state tracked is the **optimization complexity**, defined as the product of rendering resolution and primitive count, which dominates time costs [S1].

**Formulas**
The rendering equation for a pixel $u$ with $P$ primitives is [S1]:
$$ C(u) = \sum_{i=1}^P T_i(u) \cdot w_i(u) \cdot c_i $$
where:
*   $T_i(u) = \prod_{j=1}^{i-1} (1 - w_j(u))$ is the transmittance.
*   $w_i(u) = o_i G_{i2D}(u)$ is the weight, with $o_i$ being opacity and $G_{i2D}(u)$ being the 2D Gaussian splatted to the image plane.
*   $c_i$ is the color coefficient.

**Parameters**
*   **Target Optimization Time:** 200 seconds when applied to the Taming-3DGS backbone [S1].
*   **Resolution Downsampling Factor $r(i)$:** A continuous scalar modulated in practice as $\lfloor r(i) \rfloor$ to encourage faster primitive growth relative to the unmodulated case [S1].

**Steps**
1.  **Frequency Formulation:** Reformulate 3DGS optimization as progressively fitting the Gaussians to higher levels of frequency components in the training views. This is based on the observation that increasing resolution is equivalent to fitting residual high-frequency components [S1].
2.  **Dynamic Resolution Scheduling:** Apply a scene-adaptive dynamic rendering resolution scheme.
    *   Start with a lower resolution to reduce optimization complexity.
    *   Gradually increase resolution.
    *   **Pitfall Avoidance:** To prevent 3D aliasing and artifacts when downsampling, apply a low-pass filter before downsampling to avoid disturbing recoverable low-frequency components [S1].
3.  **Primitive Number Scheduling:** Schedule the growth of primitives to synchronize with the rendering resolution.
    *   Couple the primitive count with the resolution via a numerical constraint.
    *   Produce a **concave-up** growth curve for primitives that spans the entire optimization process [S1].
    *   This contrasts with existing works that use concave-down curves or curves peaking at half-iteration [S1].
4.  **Momentum-Based Budgeting:** Use a momentum-based primitive budgeting method to adaptively estimate the maximum number of primitives in the growth curve. This avoids reliance on dataset priors [S1].
5.  **Optimization Loop:** Execute the standard 3DGS optimization loop (forward rendering, gradient back-propagation, primitive optimizing) using the scheduled resolution and primitive count. This reduces per-iteration elapsed time for all three major operations [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dash_gaussian_optimization(training_views, backbone, total_iterations):
    # Initialize primitives using backbone's initialization
    primitives = backbone.initialize_primitives(training_views)
    
    # Initialize momentum for primitive budgeting
    momentum = 0.0
    momentum_decay = 0.9  # [G] Typical momentum decay value
    
    for i in range(total_iterations):
        # 1. Calculate dynamic resolution
        # r(i) is a continuous scalar derived from the frequency formulation
        # In practice, modulate with floor function
        continuous_r = calculate_resolution_factor(i, total_iterations)
        resolution_factor = floor(continuous_r)
        
        # 2. Apply anti-aliasing downsampling if resolution is reduced
        # Low-pass filter is applied before downsampling to prevent aliasing
        current_resolution = apply_low_pass_filter_and_downsample(
            training_views, resolution_factor
        )
        
        # 3. Estimate primitive budget using momentum-based method
        # This adaptively estimates the max primitives in the concave-up curve
        current_primitive_count = momentum_based_budget(
            primitives, momentum, momentum_decay
        )
        
        # 4. Update momentum based on current primitive usage/growth
        momentum = momentum_decay * momentum + (1 - momentum_decay) * current_primitive_count
        
        # 5. Render and Optimize
        # Forward rendering with current resolution and primitive count
        rendered_images = render_primitives(primitives, current_resolution)
        
        # Compute loss against training views
        loss = compute_loss(rendered_images, current_resolution)
        
        # Back-propagation and optimization step
        gradients = compute_gradients(loss, primitives)
        primitives = backbone.optimizer_step(primitives, gradients)
        
        # 6. Densification/Pruning according to backbone's schedule
        # DashGaussian's scheduler influences the *rate* and *timing* via the constraints
        primitives = backbone.densify_and_prune(primitives, i)
        
    return primitives
```

**Subtopics**
*   **Error-Driven 2D Gaussian Training:** Focuses on 2D projection errors; DashGaussian focuses on frequency-based resource allocation. [Link](13985-error-driven-2d-gaussian-training.md)
*   **Training Configuration and Hyperparameters:** General hyperparameter tuning; DashGaussian provides specific scheduling logic for resolution and primitives. [Link](16385-training-configuration-and-hyperparameters.md)

## Operating characteristics
*   **Time Complexity:** The optimization complexity is defined as rendering resolution $\times$ primitive count. DashGaussian reduces the average optimization time by 45.7% across tested backbones [S1]. Specifically, it reduces the optimization time of a Taming-3DGS model by 41.3% [S1].
*   **Memory Complexity:** By using fewer primitives (scene-adaptively), it reduces memory usage compared to vanilla 3DGS which might over-densify [S1].
*   **Performance:** Achieves SOTA rendering quality among accelerated methods, often improving PSNR/SSIM while reducing time [S1].
*   **Strengths:** Plug-and-play integration with various backbones (3DGS, Mip-Splatting, Revising-3DGS, Taming-3DGS); avoids dataset priors; reduces per-iteration time for forward, backward, and optimization steps [S1].
*   **Limitations:** Requires careful implementation of the low-pass filter for anti-aliasing; relies on the assumption that frequency fitting correlates well with resolution scaling [S1].
*   **When to use:** When training time is a bottleneck and rendering quality must be preserved or improved. Ideal for consumer-grade GPUs where 200-second optimization is a target [S1].
*   **When not to use:** If the scene has extremely high-frequency details that require immediate high-resolution fitting from the start, the initial low-resolution phase might delay convergence (though the concave-up curve mitigates this) [S1].

## Background
DashGaussian was proposed by Youyu Chen, Junjun Jiang, Kui Jiang, Xiao Tang, Zhihao Li, Xianming Liu, and Yinyu Nie from Harbin Institute of Technology and Huawei Noah’s Ark Lab [S1]. It was published in CVPR 2025 (arXiv:2503.18402v2, March 26, 2025) [S1]. The work addresses the high computational cost of 3DGS optimization, which is dominated by rendering resolution and primitive count [S1]. It builds on the observation that naive resolution reduction causes aliasing, and proposes a frequency-based scheduling approach to balance computational redundancy and fitting quality [S1].

## Sources
[S1] DashGaussian: Optimizing 3D Gaussian Splatting in 200 Seconds, https://doi.org/10.1109/cvpr52734.2025.01041, fetched 2026-10-31
