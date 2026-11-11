# Hyperparameter Settings

Also called: Optimizer and Regularization Settings, Optimizer settings, Phys3DGS Hyperparameter Settings, Training Hyperparameters

## Overview
Hyperparameter settings define the numerical configuration for the optimizer, learning rates, and regularization terms used during the training of 3D Gaussian Splatting variants. These settings are critical for stabilizing the optimization process, particularly in physically-based inverse rendering where material-lighting ambiguities and geometric errors can lead to gradient overflow. This article consolidates specific hyperparameter values from [S1] and [S2, unofficial], focusing on the Adam optimizer configuration, per-parameter learning rates, and regularization strengths required to prevent instability caused by erroneous Gaussians.

## How it works
The training process relies on a differentiable rasterizer and an optimizer (typically Adam) to update Gaussian parameters (position, scale, opacity, color, etc.) and potentially auxiliary networks (MLPs). The stability of this process depends heavily on the balance between learning rates and regularization penalties.

### Inputs and Outputs
*   **Inputs:**
    *   Ground truth images (RGB, and optionally depth/ToF).
    *   Camera poses and intrinsics.
    *   Initial set of 3D Gaussians.
    *   Hyperparameter configuration (learning rates, regularization weights, optimizer constants).
*   **Outputs:**
    *   Optimized Gaussian attributes (means, covariances, opacities, spherical harmonics coefficients).
    *   Optimized material properties (base color, metallic, roughness) if performing inverse rendering.
    *   Optimized lighting coefficients (direct/indirect SH) if performing inverse rendering.

### Data Structures
The parameters are stored in GPU buffers. The specific layout depends on the variant (standard 3DGS vs. Phys3DGS).
*   **Standard 3DGS Parameters:** Position (3), Scale (3), Rotation (4, quaternion), Opacity (1), Color SH (variable, e.g., 45 for deg 2).
*   **Phys3DGS Parameters:** Adds Base Color (3), Metallic (1), Roughness (1), Normal Rotation (4, quaternion), Direct Light SH, Indirect Light SH, Visibility SH.
*   **MLP Weights:** If a deformation network is used, weights are stored in standard tensor format.

### Numbered Steps
1.  **Initialization:**
    *   Initialize Gaussian attributes based on the densification strategy.
    *   Initialize auxiliary parameters (e.g., reflectivity, MLP weights) to small values to ensure stable optimization [S1].
    *   Set initial learning rates for each parameter group.
2.  **Forward Pass:**
    *   Render the scene using the differentiable rasterizer to produce predicted images $I'$.
    *   Compute the loss $L$ between predicted images and ground truth $I$.
3.  **Backward Pass:**
    *   Compute gradients $\nabla \theta$ for all learnable parameters $\theta$.
4.  **Optimizer Update:**
    *   Update parameters using the Adam optimizer with the configured hyperparameters.
5.  **Regularization Application:**
    *   Apply smoothness regularizers to material properties (base color, metallic, roughness) to prevent high-frequency noise and instability [S2, unofficial].

### Formulas
**Adam Optimizer Update:**
For each parameter $\theta_t$ at step $t$:
$$ m_t = \beta_1 m_{t-1} + (1 - \beta_1) g_t $$
$$ v_t = \beta_2 v_{t-1} + (1 - \beta_2) g_t^2 $$
$$ \hat{m}_t = \frac{m_t}{1 - \beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1 - \beta_2^t} $$
$$ \theta_{t+1} = \theta_t - \frac{\eta}{\sqrt{\hat{v}_t} + \epsilon} \hat{m}_t $$
Where:
*   $g_t$ is the gradient.
*   $\eta$ is the learning rate for the specific parameter group.
*   $\beta_1, \beta_2$ are exponential decay rates for moment estimates.
*   $\epsilon$ is a small constant for numerical stability.

**Smoothness Regularization (Phys3DGS):**
The loss includes smoothness terms for material properties:
$$ L_{smooth} = \lambda_{smooth} \sum_{i} \text{Smoothness}(\theta_i) $$
Where $\theta_i$ represents base color, metallic, or roughness [S2, unofficial].

### Parameters with Typical Values
**Optimizer Constants (Common to [S1] and [S2, unofficial]):**
*   $\beta_1 = 0.9$
*   $\beta_2 = 0.999$
*   $\epsilon = 10^{-15}$ [S1][S2, unofficial]

**Learning Rates (Phys3DGS [S2, unofficial]):**
*   Position: $1.6 \times 10^{-4}$
*   Color SH: $2.5 \times 10^{-3}$
*   Scale: $5 \times 10^{-3}$
*   Opacity: $5 \times 10^{-2}$
*   Base Color: $1 \times 10^{-2}$
*   Metallic: $1 \times 10^{-2}$
*   Roughness: $1 \times 10^{-2}$
*   Normal Rotation: $1 \times 10^{-3}$
*   Direct Light SH (deg=0): $2.5 \times 10^{-3}$
*   Direct Light SH (deg>0): $1 \times 10^{-4}$
*   Indirect Light SH (deg=0): $2.5 \times 10^{-4}$
*   Indirect Light SH (deg>0): $1 \times 10^{-3}$
*   Visibility SH: $0.1$

**Learning Rates (ToRF [S1]):**
*   MLP Learning Rate: Starts at $\gamma_{MLP} = 0.0008$, decays exponentially to $1.6 \times 10^{-6}$ over 60,000 iterations.
*   Reflectivity Learning Rate: $\gamma_r = 0.00016$.
*   Reflectivity Initialization: $0.1$.

**Regularization Strengths (Phys3DGS [S2, unofficial]):**
*   $\lambda_{smooth}$ (Base Color): $0.06$
*   $\lambda_{smooth}$ (Metallic): $0.02$
*   $\lambda_{smooth}$ (Roughness): $0.02$
*   $\lambda_o$ (Other regularizers): $1.0$
*   $\lambda_{sc}$ (Other regularizers): $1.0$
*   $\lambda_{sr}$ (Other regularizers): $1.0$

**Loss Weights (ToRF [S1]):**
*   $\alpha = 5$ (for real-world scenes), $\alpha = 1$ (for synthetic scenes).
*   $\beta = 0.0008$.

### Edge Cases and Pitfalls
*   **Gradient Overflow:** If a single Gaussian behaves abnormally (e.g., driven to the opposite side of the surface, large scale, or extreme opacity), it can cause instability. Erroneous Gaussians interfere with visibility, occluding global illumination. The optimizer may compensate by increasing local illumination intensity or decreasing opacity to extreme values, leading to gradient overflow in SH coefficients or opacity values [S2, unofficial].
*   **Material-Lighting Ambiguity:** In inverse rendering, optimization of material and lighting is ambiguous. Explicit computation of rescale factors for base color (using median pixel-wise ratios) is recommended for synthetic scenes with ground truth, but omitted for real captures [S2, unofficial].
*   **MLP Initialization:** MLP weights should be initialized from a normal distribution with a standard deviation of $10^{-5}$ to ensure stable optimization by starting Gaussian deformations from small values [S1].

## Subtopics

### [Conditional 3DGS Optimization and Rendering](11459-conditional-3dgs-optimization-and-rendering.md)
Adapts 3DGS training for 7D data (3D spatial, 1D temporal, 3D directional) by converting conditional representations into standard 3DGS formats. Uses temporal cues for adaptive density control and modified pruning thresholds.

## Operating characteristics
*   **Time Complexity:** The hyperparameter settings themselves do not change the asymptotic time complexity of the optimization step, which is dominated by the rasterization and gradient computation. However, improper settings (e.g., too high learning rates) can lead to divergence, requiring restarts and increasing effective training time.
*   **Memory Complexity:** No additional memory is required for hyperparameters themselves. However, Phys3DGS requires storing additional attributes (base color, metallic, roughness, lighting SH) per Gaussian, increasing memory usage compared to standard 3DGS.
*   **Strengths:** Carefully tuned learning rates and regularization prevent instability in complex inverse rendering tasks. The use of Adam with specific $\epsilon$ ensures numerical stability for small gradients.
*   **Limitations:** The settings are specific to certain variants (Phys3DGS, ToRF). Generalizing these exact values to other 3DGS variants may require re-tuning. The high learning rate for opacity ($5 \times 10^{-2}$) may require careful monitoring to prevent premature pruning or explosion.

## Background
The hyperparameter settings described here originate from recent advancements in 3D Gaussian Splatting, particularly in the context of physically-based inverse rendering and dynamic scenes.
*   [S1] (ToRF) introduces settings for optimizing depth indirectly in dynamic radiance fields, using an MLP for deformation and specific learning rate schedules.
*   [S2, unofficial] (Phys3DGS) introduces settings for physically-based 3D Gaussian Splatting for inverse rendering, emphasizing regularization to handle material-lighting ambiguity and prevent instability from erroneous Gaussians.

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
