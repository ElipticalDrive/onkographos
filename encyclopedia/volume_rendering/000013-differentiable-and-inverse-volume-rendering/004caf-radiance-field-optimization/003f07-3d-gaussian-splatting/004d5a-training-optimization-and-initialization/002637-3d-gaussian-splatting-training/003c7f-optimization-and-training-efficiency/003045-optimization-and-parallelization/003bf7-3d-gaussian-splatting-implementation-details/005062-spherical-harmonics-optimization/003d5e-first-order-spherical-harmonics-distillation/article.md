# First-order Spherical Harmonics Distillation

Also called: Mobile-GS SH Distillation, SH Knowledge Distillation, SH Order Reduction via Distillation, SH degree reduction, Spherical Harmonics degree reduction

## Overview
First-order Spherical Harmonics Distillation reduces the parameter count and computational cost of 3D Gaussian Splatting by compressing the view-dependent color representation from third-order to first-order spherical harmonics (SH) [S1][S2, unofficial]. It achieves this by training a "student" model with first-order SH coefficients to approximate the radiance of a pretrained "teacher" model that uses third-order SH [S2, unofficial]. Unlike standard degree reduction which may suffer from significant quality loss, this method uses knowledge distillation to preserve rendering fidelity while enabling faster rendering and lower storage usage, making it suitable for mobile and edge devices [S2, unofficial]. It builds on [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A pretrained teacher model with third-order SH coefficients ($L=3$, 16 coefficients per color channel) and a student model initialized with first-order SH coefficients ($L=1$, 4 coefficients per color channel).
*   **Output:** Optimized first-order SH coefficients for the student model.

**Data Structures:**
*   **Teacher SH Coefficients:** For each Gaussian, a vector of size $3 \times 16$ (RGB $\times$ 16 SH coefficients).
*   **Student SH Coefficients:** For each Gaussian, a vector of size $3 \times 4$ (RGB $\times$ 4 SH coefficients).
*   **Rendered Color ($\mathbf{C}$):** The output color of a pixel after alpha compositing the splatted Gaussians.
*   **Rendered Depth ($\mathbf{D}$):** The accumulated depth of a pixel.

**Parameters:**
*   $\epsilon = 10^{-8}$: A small constant added to depth values for numerical stability [S2, unofficial].
*   Target SH Degree: 1 (first-order) [S1][S2, unofficial].

**Steps:**
1.  **Initialize Teacher:** Train or load a 3D Gaussian Splatting model using third-order SH ($L=3$) as the teacher model [S2, unofficial].
2.  **Initialize Student:** Initialize a student model with the same Gaussian positions, opacities, scales, rotations, and base colors, but with first-order SH coefficients ($L=1$) initialized to zero or copied from the teacher's first-order components [G].
3.  **Render:** Render images from both the teacher and student models for a batch of camera viewpoints.
    *   Compute teacher rendered color $\mathbf{C}_p^{tea}$ and depth $\mathbf{D}_p^{tea}$ for each pixel $p$.
    *   Compute student rendered color $\mathbf{C}_p$ and depth $\mathbf{D}_p$ for each pixel $p$.
4.  **Compute Distillation Loss:** Calculate the color distillation loss $\mathcal{L}_{\text{distill}}$ between the teacher and student rendered colors [S2, unofficial]:
    $$ \mathcal{L}_{\text{distill}} = \frac{1}{|P|} \sum_{p \in P} \|\mathbf{C}_p^{tea} - \mathbf{C}_p\|_2^2 $$
    where $P$ is the set of pixels in the rendered image [S2, unofficial].
5.  **Compute Depth Distillation Loss:** Calculate the scale-invariant depth distillation loss $\mathcal{L}_{\text{depth}}$ to enforce geometric consistency [S2, unofficial]:
    $$ \mathcal{L}_{\text{depth}} = \frac{1}{|P|} \sum_{p \in P} \left( \log(\hat{\mathbf{D}}_p) - \log(\hat{\mathbf{D}}_p^{tea}) \right)^2 - \frac{1}{|P|^2} \left( \sum_{p \in P} \left( \log(\hat{\mathbf{D}}_p) - \log(\hat{\mathbf{D}}_p^{tea}) \right) \right)^2 $$
    where $\hat{\mathbf{D}}_p = \mathbf{D}_p + \epsilon$ and $\hat{\mathbf{D}}_p^{tea} = \mathbf{D}_p^{tea} + \epsilon$, with $\epsilon = 10^{-8}$ [S2, unofficial].
6.  **Optimize:** Update the student model's first-order SH coefficients (and potentially other attributes) by minimizing the combined loss:
    $$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{distill}} + \lambda_{\text{depth}} \mathcal{L}_{\text{depth}} $$
    where $\lambda_{\text{depth}}$ is a weighting factor for the depth loss [G].
7.  **Iterate:** Repeat steps 3-6 until convergence.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def first_order_sh_distillation(teacher_model, student_model, camera_poses, num_iterations):
    """
    Distills third-order SH from a teacher model to a first-order student model.
    """
    epsilon = 1e-8
    lambda_depth = 1.0  # [G] Typical weight for depth loss; not specified in sources
    
    for iteration in range(num_iterations):
        # Select a random camera pose
        pose = random.choice(camera_poses)
        
        # Render from teacher and student
        # Returns: color (H, W, 3), depth (H, W, 1)
        teacher_color, teacher_depth = teacher_model.render(pose)
        student_color, student_depth = student_model.render(pose)
        
        # Compute Color Distillation Loss
        # L_distill = mean squared error between teacher and student colors
        color_loss = mean_squared_error(teacher_color, student_color)
        
        # Compute Scale-Invariant Depth Distillation Loss
        # Add epsilon for stability
        teacher_depth_safe = teacher_depth + epsilon
        student_depth_safe = student_depth + epsilon
        
        log_teacher_depth = log(teacher_depth_safe)
        log_student_depth = log(student_depth_safe)
        
        # Term 1: Mean of squared differences
        term1 = mean(square(log_student_depth - log_teacher_depth))
        
        # Term 2: Square of mean differences
        term2 = square(mean(log_student_depth - log_teacher_depth))
        
        depth_loss = term1 - term2
        
        # Total Loss
        total_loss = color_loss + lambda_depth * depth_loss
        
        # Backpropagate and update student model parameters
        student_model.zero_grad()
        total_loss.backward()
        student_model.optimizer.step()
        
    return student_model
```

**Edge Cases and Pitfalls:**
*   **Depth Reliability:** The teacher model's depth may not always be reliable, so strict L1/L2 depth losses are avoided in favor of scale-invariant loss [S2, unofficial].
*   **Precision:** The addition of $\epsilon = 10^{-8}$ is critical for training stability when taking the logarithm of depth values [S2, unofficial].
*   **Degenerate Input:** If the teacher model is poorly trained, the distillation may propagate errors to the student model [G].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Reduces SH storage from 16 coefficients per channel to 4 coefficients per channel, a 75% reduction in SH parameters per Gaussian [S1][S2, unofficial].
*   **Computation:** Reduces rendering computation by 52% when distilling directly from degree 3 to 1 [S1].

**Measured Performance:**
*   **Degree 3→2:** -21 elements, -36% parameters, -35% computation, -0.17 dB PSNR [S1].
*   **Degree 2→1:** -15 elements, -25% parameters, -17% computation, -0.31 dB PSNR [S1].
*   **Degree 3→1:** -36 elements, -61% parameters, -52% computation, -0.48 dB PSNR [S1].
*   **Overall:** When combined with other compression methods (like VQ and pruning), SH distillation contributes to a 51.6× overall compression with a 0.73 dB PSNR drop [S1].

**Strengths and Limitations:**
*   **Strengths:** Significantly reduces storage burden and increases rendering speed, making it suitable for mobile devices [S2, unofficial]. Maintains higher fidelity than naive degree reduction due to distillation [S2, unofficial].
*   **Limitations:** Involves a two-stage training process (teacher then student), increasing total training time [G]. There is a trade-off between compression and quality, with a 0.48 dB PSNR drop for 3→1 distillation [S1].

**When to Use:**
*   Use when deploying 3DGS on resource-constrained devices (mobile, edge) where rendering speed and memory footprint are critical [S2, unofficial].
*   Use when a high-quality teacher model is available for distillation [S2, unofficial].

**When Not to Use:**
*   Use when maximum possible rendering quality is required and computational resources are abundant [G].
*   Use when a single-stage training pipeline is preferred [G].

## Background
First-order Spherical Harmonics Distillation was introduced in the context of Mobile-GS, a framework for real-time Gaussian Splatting on mobile devices [S2, unofficial]. It addresses the high parameter count and storage burden of the original 3D Gaussian Splatting, which uses third-order SH [S2, unofficial]. The method is part of a broader effort to compress 3DGS for edge hardware, alongside Neural Vector Quantization and Contribution-based Pruning [S2, unofficial]. It builds on the observation that higher-order SH coefficients often change minimally during training, leading to redundancy [S1][S2, unofficial].

## Sources
[S1] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S2] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29 (unofficial)
