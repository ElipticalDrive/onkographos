# First-order Spherical Harmonics Distillation

Also called: Mobile-GS SH Distillation, SH Knowledge Distillation, SH Order Reduction via Distillation, SH degree reduction, Spherical Harmonics degree reduction

## Overview
First-order Spherical Harmonics (SH) Distillation is a compression technique that reduces the SH degree of 3D Gaussian elements from the standard third-order (degree 3) to first-order (degree 1) to decrease parameter count and rendering computation cost [S1][S2, unofficial]. Unlike standard degree reduction which simply truncates coefficients, this method employs a distillation strategy where a "student" model with first-order SH approximates the directional radiance of a pretrained "teacher" model [S2, unofficial]. It sets itself apart from siblings like [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md) (which quantizes coefficients) and [Compact View-dependent Color](12904-compact-view-dependent-color.md) (which uses neural fields) by explicitly training a lower-degree SH representation to mimic the output of a higher-degree reference [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A pretrained teacher 3D Gaussian model with third-order SH coefficients ($\mathbf{c}_i^{tea}$, degree 3, 16 coefficients per color channel) and a student model initialized with first-order SH coefficients ($\mathbf{c}_i$, degree 1, 4 coefficients per color channel) [S2, unofficial].
*   **Output:** Optimized first-order SH coefficients for the student model that approximate the teacher's appearance [S2, unofficial].

**Data Structures:**
The student Gaussian $i$ stores SH coefficients $\mathbf{c}_i \in \mathbb{R}^{3 \times 4}$ (RGB $\times$ 4 first-order coefficients) [S2, unofficial]. The teacher stores $\mathbf{c}_i^{tea} \in \mathbb{R}^{3 \times 16}$ [S2, unofficial].

**Steps:**
1.  **Initialization:** Initialize the student model's first-order SH coefficients. These can be initialized by projecting the teacher's third-order coefficients onto the first-order basis or randomly [S2, G, unofficial].
2.  **Rendering:** Render the scene using both the teacher (degree 3) and student (degree 1) models for a set of training views.
3.  **Loss Calculation:** Compute the distillation loss and optional depth distillation loss.
4.  **Optimization:** Update the student's first-order SH coefficients (and potentially other Gaussian attributes like position/opacity if joint optimization is used) to minimize the total loss.

**Formulas:**
The primary distillation loss matches the rendered pixel colors of the student to the teacher [S2, unofficial]:
$$
\mathcal{L}_{\text{distill}} = \frac{1}{|P|} \sum_{p \in P} \|\mathbf{C}_p^{tea} - \mathbf{C}_p\|
$$
where $P$ is the set of pixels, $\mathbf{C}_p^{tea}$ is the rendered pixel color from the teacher, and $\mathbf{C}_p$ is the rendered pixel color from the student [S2, unofficial].

To improve geometric consistency, a scale-invariant depth distillation loss is applied [S2, unofficial]:
$$
\mathcal{L}_{\text{depth}}(\mathbf{D}, \mathbf{D}^{tea}) = \frac{1}{|P|} \sum_{p \in P} \left( \log(\hat{\mathbf{D}}_p) - \log(\hat{\mathbf{D}}_p^{tea}) \right)^2 - \frac{1}{|P|^2} \left( \sum_{p \in P} \left( \log(\hat{\mathbf{D}}_p) - \log(\hat{\mathbf{D}}_p^{tea}) \right) \right)^2
$$
where $\hat{\mathbf{D}}_p^{tea} = \mathbf{D}^{tea} + \epsilon$ and $\epsilon = 10^{-8}$ for stability [S2, unofficial]. The term $\mathbf{D}^{tea}$ is the depth rendered by the teacher, and $\mathbf{D}$ is the depth rendered by the student. The second term subtracts the mean difference to make it scale-invariant [S2, unofficial].

**Parameters:**
*   Target SH degree: 1 (first-order) [S1][S2, unofficial].
*   $\epsilon$: $10^{-8}$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_first_order_distillation(teacher_model, student_model, views, optimizer, num_iterations):
    """
    Distills third-order SH from teacher to first-order SH in student.
    """
    for iteration in range(num_iterations):
        # 1. Sample a view
        view = sample_view(views)
        
        # 2. Render teacher (Degree 3)
        teacher_color, teacher_depth = render_gaussians(teacher_model, view)
        
        # 3. Render student (Degree 1)
        student_color, student_depth = render_gaussians(student_model, view)
        
        # 4. Compute Distillation Loss (Color)
        # L_dstill = Mean(||C_tea - C_stu||)
        # Assuming L2 norm per pixel, then mean over pixels
        color_diff = teacher_color - student_color
        l_distill = mean(color_diff ** 2)
        
        # 5. Compute Depth Distillation Loss (Scale-invariant)
        epsilon = 1e-8
        log_tea_depth = log(teacher_depth + epsilon)
        log_stu_depth = log(student_depth + epsilon)
        
        diff_log = log_tea_depth - log_stu_depth
        mean_diff = mean(diff_log)
        
        # L_depth = mean((diff - mean(diff))^2)
        l_depth = mean((diff_log - mean_diff) ** 2)
        
        # 6. Total Loss
        # Note: Weights for depth loss are not specified in sources [G: assume 1.0 or tuneable]
        total_loss = l_distill + l_depth
        
        # 7. Backpropagate and update student SH coefficients
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
```

## Subtopics

### L3GS with Spherical Harmonics Distillation
A variant where splats are organized into layers, but each splat contains both a compact (first-order/distilled) and full (third-order) version of the SH representation, while other shared features (opacity, location) remain consistent [S3]. The dimension (compact or full) of the SH is added as a component to the utility function, allowing the scheduler to choose from layered splats with either full or compact SH [S3]. Initially, compact SH versions are loaded to allow more splats within the same bandwidth; over time, compact SH is replaced by full SH to enhance visual quality [S3].

## Operating characteristics
*   **Compression:** Reducing SH degree from 3 to 1 reduces parameters by 61% and computation by 52% [S1].
*   **Quality Trade-off:** The PSNR drop for 3→1 distillation is approximately 0.48 dB when combined with other compression methods (like pruning and VQ) [S1]. Standalone 3→1 reduction results in a 0.48 dB drop [S1].
*   **Performance:** Enables faster rendering speeds and lower storage usage, making it suitable for mobile devices and edge hardware [S2, unofficial][S1].
*   **Limitations:** The quality drop is more significant than degree 3→2 reduction (-0.17 dB) or 2→1 reduction (-0.31 dB) individually, but the cumulative effect with other compression techniques is managed within bounded quality costs [S1].

## Background
First-order Spherical Harmonics Distillation was introduced in the context of Mobile-GS (Mobile Gaussian Splatting) to enable real-time 3DGS on mobile devices [S2, unofficial]. It addresses the high storage and computational burden of the original 3DGS which uses third-order SH [S2, unofficial]. The method builds on the concept of knowledge distillation, applying it specifically to the view-dependent color representation in 3DGS [S2, unofficial].

## Sources
[S1] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S2] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29 (unofficial)
[S3] L3GS: Layered 3D Gaussian Splats for Efficient 3D Scene Delivery, https://doi.org/10.1145/3680207.3723485, fetched 2026-10-27
