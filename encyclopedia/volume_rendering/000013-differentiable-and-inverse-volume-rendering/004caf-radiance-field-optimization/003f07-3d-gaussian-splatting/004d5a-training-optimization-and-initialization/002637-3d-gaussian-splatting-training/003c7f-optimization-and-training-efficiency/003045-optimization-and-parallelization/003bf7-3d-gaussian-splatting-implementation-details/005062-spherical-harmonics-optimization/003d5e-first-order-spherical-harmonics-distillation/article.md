# First-order Spherical Harmonics Distillation

Also called: Mobile-GS SH Distillation, SH Knowledge Distillation, SH Order Reduction via Distillation, SH degree reduction, Spherical Harmonics degree reduction

## Overview
First-order Spherical Harmonics Distillation reduces the view-dependent color representation of 3D Gaussians from third-order to first-order spherical harmonics (SH) to decrease parameter count and rendering computation [S1][S2, unofficial]. It achieves this by training a "student" model with first-order SH coefficients to approximate the radiance output of a pretrained "teacher" model using third-order SH [S2, unofficial]. Unlike standard degree reduction which may simply truncate coefficients, this method uses knowledge distillation to preserve visual fidelity despite the significant reduction in representational capacity [S2, unofficial]. It is distinct from [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md) which compresses coefficients via codebooks, and [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md) which optimizes training gradients rather than representation size.

## How it works
**Inputs and Outputs:**
*   **Input:** A pretrained teacher model with third-order SH coefficients ($\mathbf{C}^{tea}$) and a student model initialized with first-order SH coefficients ($\mathbf{C}$).
*   **Output:** Optimized first-order SH coefficients for the student model.

**Data Structures:**
*   **Teacher Coefficients ($\mathbf{C}^{tea}$):** For each Gaussian, 3 color channels $\times$ 16 coefficients (degree 3).
*   **Student Coefficients ($\mathbf{C}$):** For each Gaussian, 3 color channels $\times$ 4 coefficients (degree 1).
*   **Rendered Pixel Color ($\mathbf{C}_p$):** The RGB color of pixel $p$ rendered by the student.
*   **Teacher Rendered Pixel Color ($\mathbf{C}_p^{tea}$):** The RGB color of pixel $p$ rendered by the teacher.
*   **Depth Maps ($\hat{\mathbf{D}}_p, \hat{\mathbf{D}}_p^{tea}$):** Depth values for pixel $p$ from student and teacher respectively.

**Parameters:**
*   $\epsilon$: Stability constant for depth distillation, set to $1e-8$ [S2, unofficial].
*   $P$: Set of pixels in the training image batch.

**Steps:**
1.  **Initialize Student:** Initialize the student model's SH coefficients to first-order (degree 1). The spatial attributes (position, scale, rotation, opacity) can be copied from the teacher or re-optimized [S2, unofficial].
2.  **Render Teacher:** Render the training image using the teacher model to obtain $\mathbf{C}_p^{tea}$ and $\hat{\mathbf{D}}_p^{tea}$ for all pixels $p \in P$ [S2, unofficial].
3.  **Render Student:** Render the training image using the student model to obtain $\mathbf{C}_p$ and $\hat{\mathbf{D}}_p$ for all pixels $p \in P$ [S2, unofficial].
4.  **Compute Distillation Loss:** Calculate the color distillation loss $\mathcal{L}_{\text{dstill}}$ between the student and teacher rendered colors [S2, unofficial]:
    $$ \mathcal{L}_{\text{dstill}} = \frac{1}{|P|} \sum_{p \in P} \|\mathbf{C}_p^{tea} - \mathbf{C}_p\| $$
    Note: The norm $\|\cdot\|$ is typically the L2 norm (Euclidean distance) for RGB vectors, though the source notation is generic [S2, unofficial].
5.  **Compute Depth Distillation Loss:** Calculate the scale-invariant depth distillation loss $\mathcal{L}_{\text{depth}}$ to enforce geometric consistency [S2, unofficial]:
    $$ \mathcal{L}_{\text{depth}}(\hat{\mathbf{D}}, \hat{\mathbf{D}}^{tea}) = \frac{1}{|P|} \sum_{p \in P} \left( \log(\hat{\mathbf{D}}_p) - \log(\hat{\mathbf{D}}_p^{tea}) \right)^2 - \frac{1}{|P|^2} \left( \sum_{p \in P} \left( \log(\hat{\mathbf{D}}_p) - \log(\hat{\mathbf{D}}_p^{tea}) \right) \right)^2 $$
    where $\hat{\mathbf{D}}_p^{tea} = \mathbf{D}^{tea}_p + \epsilon$ and $\epsilon = 1e-8$ [S2, unofficial]. The second term is the mean of the log-differences, effectively making the loss scale-invariant [S2, unofficial].
6.  **Optimize:** Update the student's first-order SH coefficients (and potentially other attributes) by minimizing the combined loss. The source emphasizes that strict L1/L2 depth losses are not used because teacher depth may be unreliable and slight differences are expected [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def first_order_sh_distillation(teacher_model, student_model, images, intrinsics, extrinsics, epsilon=1e-8):
    """
    Distills third-order SH from teacher to first-order SH in student.
    """
    # 1. Render Teacher
    teacher_colors, teacher_depths = render_scene(teacher_model, images, intrinsics, extrinsics)
    
    # 2. Render Student
    student_colors, student_depths = render_scene(student_model, images, intrinsics, extrinsics)
    
    # 3. Compute Color Distillation Loss
    # L_dstill = Mean L2 distance between teacher and student pixel colors
    color_diff = teacher_colors - student_colors
    l_dstill = mean(square(color_diff)) # Assuming L2 norm squared inside mean
    
    # 4. Compute Scale-Invariant Depth Distillation Loss
    # Add epsilon for stability
    teacher_depth_log = log(teacher_depths + epsilon)
    student_depth_log = log(student_depths + epsilon)
    
    diff = student_depth_log - teacher_depth_log
    mean_diff = mean(diff)
    
    # L_depth = Mean squared diff - (Mean diff)^2
    l_depth = mean(square(diff)) - square(mean_diff)
    
    # 5. Update Student
    # Combine losses (weights not specified in source, assume equal or tuned)
    total_loss = l_dstill + l_depth
    
    # Backpropagate and update student_model.sh_coefficients (first-order)
    optimize(student_model, total_loss)
    
    return student_model
```

**Edge Cases and Pitfalls:**
*   **Depth Reliability:** The teacher's depth map may not be perfectly accurate. Using a scale-invariant loss prevents the student from overfitting to potentially noisy absolute depth values [S2, unofficial].
*   **Precision:** The epsilon value ($1e-8$) is critical for numerical stability in the log-depth calculation [S2, unofficial].
*   **Representation Gap:** Reducing from degree 3 (16 coeffs) to degree 1 (4 coeffs) is a large reduction. Distillation is necessary because simple truncation or projection would result in significant quality loss [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Reducing SH degree from 3 to 1 reduces the SH parameter count per Gaussian by 61% (from 48 to 12 coefficients for RGB) [S1].
*   **Computation:** Rendering computation is reduced by 52% when moving from degree 3 to degree 1 [S1].

**Measured Performance:**
*   **Degree 3→2:** Reduces parameters by 36% and computation by 35%, with a PSNR drop of 0.17 dB [S1].
*   **Degree 2→1:** Reduces parameters by 25% and computation by 17%, with a PSNR drop of 0.31 dB [S1].
*   **Degree 3→1:** Reduces parameters by 61% and computation by 52%, with a PSNR drop of 0.48 dB [S1].
*   **Overall:** When combined with other compression methods (like VQ), SH distillation contributes to a total model compression of 51.6× with a total PSNR drop of 0.73 dB [S1].

**Strengths and Limitations:**
*   **Strengths:** Significantly lowers storage burden and increases rendering speed, making it suitable for mobile devices and edge hardware [S1][S2, unofficial].
*   **Limitations:** Introduces a quality trade-off, with a 0.48 dB PSNR drop for full 3→1 distillation [S1]. It requires a pretrained teacher model, adding an extra training step [S2, unofficial].

**When to Use:**
*   Use when deploying 3DGS on resource-constrained devices (mobile, embedded) where rendering speed and memory footprint are critical [S2, unofficial].
*   Use when a slight decrease in PSNR is acceptable for significant compression gains [S1].

**When Not to Use:**
*   Use when maximum visual fidelity is required and computational resources are abundant (standard desktop rendering).
*   Use when a pretrained teacher model is not available.

## Background
First-order Spherical Harmonics Distillation was introduced in the context of **Mobile-GS**, a framework designed for real-time Gaussian Splatting on mobile devices [S2, unofficial]. The method addresses the high parameter count and storage burden of the original 3D Gaussian Splatting (3DGS) which uses third-order SH [S2, unofficial]. It builds on the concept of knowledge distillation, where a smaller student model learns from a larger teacher model. While LightGaussian used distillation to project third-order SH to second-order, Mobile-GS extends this to first-order for even greater compression [S2, unofficial]. The method is part of a broader suite of compression techniques including Neural Vector Quantization and Contribution-based Pruning [S2, unofficial].

## Sources
[S1] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S2] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29 (unofficial)
