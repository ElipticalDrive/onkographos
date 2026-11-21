# First-order Spherical Harmonics Distillation

Also called: Mobile-GS SH Distillation, SH Knowledge Distillation, SH Order Reduction via Distillation, SH degree reduction, Spherical Harmonics degree reduction

## Overview
First-order Spherical Harmonics Distillation reduces the spherical harmonics (SH) degree of 3D Gaussian elements from third-order to first-order to decrease parameter count and computation cost, enabling real-time rendering on mobile devices [S2, unofficial][S1]. It differs from standard SH degree reduction by employing a distillation strategy where a student model with first-order SH approximates the directional radiance of a pretrained third-order teacher model, rather than simply truncating coefficients [S2, unofficial]. This approach minimizes the quality loss associated with aggressive compression, achieving a 61% reduction in parameters and 52% reduction in computation with only a 0.48 dB PSNR drop [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pretrained teacher 3D Gaussian Splatting model using third-order SH (16 coefficients per color channel, $3 \times 16$ total) and a set of training images with corresponding camera poses.
*   **Output:** A student 3D Gaussian Splatting model using first-order SH (4 coefficients per color channel, $3 \times 4$ total) with optimized parameters.

**Data Structures:**
The student model maintains the standard 3DGS Gaussian attributes:
*   $\boldsymbol{\mu}$: 3D mean position (float32, 3 values).
*   $\boldsymbol{\Sigma}$: Covariance matrix (float32, 6 values due to symmetry).
*   $\alpha$: Opacity (float32, 1 value).
*   $\mathbf{c}$: SH coefficients for first-order (float32, 12 values: 4 per RGB channel).
*   $\mathbf{s}$: Scale (float32, 3 values).

**Steps:**
1.  **Initialization:** Initialize the student model with the same Gaussian primitives (means, covariances, scales, opacities) as the teacher model. Initialize the student's first-order SH coefficients $\mathbf{c}$ to approximate the teacher's third-order SH coefficients $\mathbf{c}^{tea}$ via a linear projection or random initialization [S2, unofficial].
2.  **Rendering:** Render the scene using the student model to obtain pixel colors $\mathbf{C}_p$ and depths $\hat{\mathbf{D}}_p$ for each pixel $p$ in the training set $P$. The rendering follows the standard 3DGS equation [Parent].
3.  **Teacher Rendering:** Render the scene using the frozen teacher model to obtain teacher pixel colors $\mathbf{C}_p^{tea}$ and depths $\hat{\mathbf{D}}_p^{tea}$.
4.  **Distillation Loss Calculation:** Compute the distillation loss $\mathcal{L}_{\text{dstill}}$ and scale-invariant depth distillation loss $\mathcal{L}_{\text{depth}}$.
5.  **Optimization:** Update the student model parameters (primarily $\mathbf{c}$, and potentially $\alpha$ and $\boldsymbol{\mu}$ if joint optimization is used) by minimizing the combined loss.

**Formulas:**
The color distillation loss is calculated as [S2, unofficial]:
$$ \mathcal{L}_{\text{dstill}} = \frac{1}{|P|} \sum_{p \in P} \|\mathbf{C}_p^{tea} - \mathbf{C}_p\| $$
where $P$ is the set of pixels, $\mathbf{C}_p^{tea}$ is the rendered pixel color from the teacher, and $\mathbf{C}_p$ is the rendered pixel color from the student.

The scale-invariant depth distillation loss is [S2, unofficial]:
$$ \mathcal{L}_{\text{depth}}(\mathbf{D}, \mathbf{D}^{tea}) = \frac{1}{|P|} \sum_{p \in P} \left( \log(\hat{\mathbf{D}}_p) - \log(\hat{\mathbf{D}}_p^{tea}) \right)^2 - \frac{1}{|P|^2} \left( \sum_{p \in P} \left( \log(\hat{\mathbf{D}}_p) - \log(\hat{\mathbf{D}}_p^{tea}) \right) \right)^2 $$
where $\hat{\mathbf{D}}_p^{tea} = \mathbf{D}^{tea} + \epsilon$ and $\epsilon = 1e-8$ is added for training stability [S2, unofficial].

**Parameters:**
*   Target SH degree: 1 [S1].
*   Epsilon $\epsilon$: $1e-8$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def first_order_sh_distillation(teacher_model, student_model, training_data, num_iterations):
    """
    Distill third-order SH from teacher to first-order SH in student.
    """
    # Initialize student SH coefficients to first-order approximation of teacher
    # This can be done by projecting teacher's 16 coefficients to 4 coefficients
    # or initializing with zeros and letting optimization handle it.
    # For implementation, we assume student_model.sh_coeffs is initialized.
    
    optimizer = torch.optim.Adam(student_model.parameters(), lr=1e-3)
    
    for iteration in range(num_iterations):
        optimizer.zero_grad()
        
        # Get a batch of training images and camera poses
        images, cameras = training_data.get_batch()
        
        # Render student scene
        student_outputs = student_model.render(cameras)
        student_colors = student_outputs['colors'] # Shape: (H, W, 3)
        student_depths = student_outputs['depths'] # Shape: (H, W, 1)
        
        # Render teacher scene (frozen)
        with torch.no_grad():
            teacher_outputs = teacher_model.render(cameras)
            teacher_colors = teacher_outputs['colors']
            teacher_depths = teacher_outputs['depths']
            
        # Calculate Color Distillation Loss
        # L_dstill = mean(||C_tea - C_stu||)
        color_loss = torch.mean(torch.abs(teacher_colors - student_colors))
        
        # Calculate Depth Distillation Loss
        # L_depth = scale-invariant log loss
        epsilon = 1e-8
        teacher_depths_safe = teacher_depths + epsilon
        student_depths_safe = student_depths + epsilon
        
        log_diff = torch.log(student_depths_safe) - torch.log(teacher_depths_safe)
        
        # Mean of log differences
        mean_log_diff = torch.mean(log_diff)
        
        # Variance-like term: mean of squared diff - squared mean diff
        depth_loss = torch.mean(log_diff ** 2) - (mean_log_diff ** 2)
        
        # Total Loss
        total_loss = color_loss + depth_loss
        
        # Backpropagate
        total_loss.backward()
        optimizer.step()
        
    return student_model
```

**Edge Cases and Pitfalls:**
*   **Depth Reliability:** The teacher's depth may not always be reliable, so strict L1/L2 depth losses are avoided in favor of scale-invariant loss [S2, unofficial].
*   **Initialization:** Poor initialization of first-order SH coefficients may lead to slow convergence or suboptimal local minima. Projecting teacher coefficients to first-order is a common initialization strategy [S2, unofficial].
*   **Precision:** The epsilon value ($1e-8$) is critical for numerical stability in the log-depth loss [S2, unofficial].

## Subtopics

### SH Degree Reduction (General)
Reduces SH degree from 3 to 2, 2 to 1, or 3 to 1 without distillation. Less effective than distillation for maintaining quality at low degrees [S1].

### Spherical Harmonics Vector Quantization
Further compresses the distilled SH features using neural vector quantization, reducing storage further [S2, unofficial].

### Contribution-based Pruning
Removes Gaussian points based on their contribution to the image, often used in conjunction with SH distillation for higher compression [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering is faster due to fewer SH coefficients (4 vs 16 per channel). The distillation training time is comparable to standard 3DGS training but requires rendering from the teacher model at each step, increasing training time [S2, unofficial].
*   **Memory Complexity:** Reduces SH storage by 61% (from 16 to 4 coefficients per channel) [S1].
*   **Performance:** On Mip-NeRF360, Tanks&Temples, and Deep Blending datasets, SH distillation from degree 3 to 1 results in a 0.48 dB PSNR drop compared to the original 3DGS [S1].
*   **Strengths:** Significant reduction in model size and rendering time, suitable for mobile devices [S2, unofficial].
*   **Limitations:** Quality loss compared to full third-order SH [S1].

## Background
First-order Spherical Harmonics Distillation was introduced in Mobile-GS (2026) as part of a suite of techniques to enable real-time 3D Gaussian Splatting on mobile devices [S2, unofficial]. It builds on the observation that third-order SH introduces a significant storage burden [S2, unofficial]. The method is related to LightGaussian, which uses distillation to project third-order SH to second-order [S2, unofficial].

## Sources
[S1] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S2] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29 (unofficial)
