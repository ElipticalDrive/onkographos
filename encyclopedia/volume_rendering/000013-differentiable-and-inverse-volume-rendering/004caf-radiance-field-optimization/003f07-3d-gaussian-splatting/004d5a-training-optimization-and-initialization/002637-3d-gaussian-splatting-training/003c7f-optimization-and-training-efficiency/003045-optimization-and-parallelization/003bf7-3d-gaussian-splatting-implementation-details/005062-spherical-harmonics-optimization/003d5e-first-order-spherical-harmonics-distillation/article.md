# First-order Spherical Harmonics Distillation

Also called: Mobile-GS SH Distillation, SH Knowledge Distillation, SH Order Reduction via Distillation, SH degree reduction, Spherical Harmonics degree reduction

## Overview
First-order Spherical Harmonics (SH) Distillation reduces the directional appearance complexity of 3D Gaussian primitives by distilling a high-degree (typically third-order) teacher model into a low-degree (first-order) student model [S2, unofficial]. Unlike standard degree reduction which simply truncates coefficients, this method uses knowledge distillation to preserve visual fidelity despite the significant reduction in parameter count [S2, unofficial]. It sets itself apart from siblings like [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md) by focusing on architectural degree reduction via optimization rather than codebook compression, and from [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md) by targeting appearance representation rather than gradient computation efficiency.

## How it works
**Inputs and Outputs:**
- **Input:** A pretrained teacher 3D Gaussian model with third-order SH coefficients ($d=3$, 15 coefficients per channel) and a student model initialized with first-order SH coefficients ($d=1$, 4 coefficients per channel) [S2, unofficial].
- **Output:** Optimized first-order SH coefficients for the student model that approximate the teacher's radiance field [S2, unofficial].

**Data Structures:**
- **Teacher SH Coefficients ($\mathbf{C}^{tea}$):** For each Gaussian, a vector of size $3 \times 16$ (RGB channels $\times$ 16 coefficients for $d=3$) [S2, unofficial].
- **Student SH Coefficients ($\mathbf{C}$):** For each Gaussian, a vector of size $3 \times 4$ (RGB channels $\times$ 4 coefficients for $d=1$) [S2, unofficial].
- **Rendered Pixel Color ($\mathbf{C}_p$):** The final color output for pixel $p$ from the student renderer [S2, unofficial].
- **Rendered Teacher Pixel Color ($\mathbf{C}_p^{tea}$):** The final color output for pixel $p$ from the teacher renderer [S2, unofficial].
- **Depth Maps ($\mathbf{D}, \mathbf{D}^{tea}$):** Depth buffers for student and teacher respectively [S2, unofficial].

**Steps:**
1. **Initialization:** Initialize the student model's SH coefficients to zero or random values, keeping other Gaussian attributes (position, covariance, opacity) fixed or jointly optimized depending on the specific training protocol [S2, unofficial]. The teacher model is frozen [S2, unofficial].
2. **Rendering:** Render the current student state to obtain pixel colors $\mathbf{C}_p$ and depth $\hat{\mathbf{D}}_p$ for a batch of training views [S2, unofficial]. Render the teacher state to obtain $\mathbf{C}_p^{tea}$ and $\hat{\mathbf{D}}_p^{tea}$ [S2, unofficial].
3. **Loss Calculation:** Compute the distillation loss and depth distillation loss (see Formulas below) [S2, unofficial].
4. **Optimization:** Update the student's SH coefficients (and potentially other attributes) using gradient descent to minimize the total loss [S2, unofficial].
5. **Iteration:** Repeat steps 2-4 until convergence [S2, unofficial].

**Formulas:**
The primary distillation loss matches the rendered pixel colors between teacher and student [S2, unofficial]:
$$
\mathcal{L}_{\text{distill}} = \frac{1}{|P|} \sum_{p \in P} \|\mathbf{C}_p^{tea} - \mathbf{C}_p\|_2^2
$$
where $P$ is the set of pixels in the batch, $\mathbf{C}_p^{tea}$ is the teacher's rendered color, and $\mathbf{C}_p$ is the student's rendered color [S2, unofficial]. Note: The source [S2, unofficial] writes $\|\mathbf{C}_p^{tea} - \mathbf{C}_p\|$, which implies an $L_2$ norm in the context of color vectors, though strictly it could be $L_1$. Given standard practice and the depth loss formulation, $L_2$ is assumed for color magnitude [S2, unofficial].

The scale-invariant depth distillation loss ensures geometric consistency [S2, unofficial]:
$$
\mathcal{L}_{\text{depth}}(\mathbf{D}, \mathbf{D}^{tea}) = \frac{1}{|P|} \sum_{p \in P} \left( \log(\hat{\mathbf{D}}_p) - \log(\hat{\mathbf{D}}_p^{tea}) \right)^2 - \frac{1}{|P|^2} \left( \sum_{p \in P} \left( \log(\hat{\mathbf{D}}_p) - \log(\hat{\mathbf{D}}_p^{tea}) \right) \right)^2
$$
where $\hat{\mathbf{D}}_p^{tea} = \mathbf{D}^{tea} + \epsilon$ and $\epsilon = 10^{-8}$ [S2, unofficial]. The second term is a mean subtraction to make the loss scale-invariant [S2, unofficial].

**Parameters:**
- **Target SH Degree:** 1 (first-order) [S2, unofficial].
- **Epsilon ($\epsilon$):** $10^{-8}$ for depth stability [S2, unofficial].
- **Loss Weights:** Not specified in the provided notes [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def compute_depth_loss(student_depth, teacher_depth, epsilon=1e-8):
    """
    Computes the scale-invariant depth distillation loss.
    student_depth: Tensor of shape (B, H, W) or (N,)
    teacher_depth: Tensor of shape (B, H, W) or (N,)
    """
    # Add epsilon for stability
    teacher_depth_safe = teacher_depth + epsilon
    
    # Log differences
    log_diff = torch.log(student_depth) - torch.log(teacher_depth_safe)
    
    # Mean squared error term
    mse_term = torch.mean(log_diff ** 2)
    
    # Mean subtraction term (scale invariance)
    mean_diff = torch.mean(log_diff)
    mean_term = mean_diff ** 2
    
    loss = mse_term - mean_term
    return loss

def distill_step(student_gaussians, teacher_gaussians, images, intrinsics, extrinsics):
    """
    Performs one step of SH distillation.
    """
    # 1. Render Student
    student_colors, student_depths = render_gaussians(student_gaussians, images, intrinsics, extrinsics)
    
    # 2. Render Teacher (Frozen)
    with torch.no_grad():
        teacher_colors, teacher_depths = render_gaussians(teacher_gaussians, images, intrinsics, extrinsics)
        
    # 3. Compute Distillation Loss (Color)
    # Assuming L2 norm for color vectors
    color_loss = torch.mean((student_colors - teacher_colors) ** 2)
    
    # 4. Compute Depth Loss
    depth_loss = compute_depth_loss(student_depths, teacher_depths)
    
    # 5. Total Loss
    # Note: Weights are not specified in sources [S2, unofficial]
    total_loss = color_loss + depth_loss
    
    # 6. Backpropagate
    total_loss.backward()
    
    # 7. Optimizer Step (Assuming an optimizer instance exists)
    # optimizer.step()
    
    return total_loss.item()
```

**Edge Cases and Pitfalls:**
- **Depth Reliability:** The teacher's depth may not be perfectly reliable, hence the use of scale-invariant loss rather than strict $L_1/L_2$ depth matching [S2, unofficial].
- **Precision:** The epsilon value $10^{-8}$ is critical for numerical stability in the log-depth calculation [S2, unofficial].
- **Degenerate Input:** If the student initialization is poor, the distillation may fail to converge to a high-quality approximation [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Rendering complexity is reduced because the student model uses fewer SH coefficients ($d=1$ vs $d=3$). The number of coefficients per channel drops from 16 to 4 [S2, unofficial].
- **Memory Complexity:** Parameter count is reduced by 61% when going directly from $d=3$ to $d=1$ [S1]. Storage burden is significantly lowered [S2, unofficial].
- **Measured Performance:**
    - $d=3 \to d=2$: -36% parameters, -35% computation, -0.17 dB PSNR [S1].
    - $d=2 \to d=1$: -25% parameters, -17% computation, -0.31 dB PSNR [S1].
    - $d=3 \to d=1$: -61% parameters, -52% computation, -0.48 dB PSNR [S1].
- **Strengths:** Enables real-time rendering on mobile devices with lower storage and higher FPS [S2, unofficial].
- **Limitations:** Inherent quality loss (PSNR drop) compared to the original third-order SH [S1].
- **When to use:** When deploying 3DGS on resource-constrained hardware (e.g., mobile devices) [S2, unofficial].
- **When not to use:** When maximum visual fidelity is required and computational resources are abundant.

## Background
First-order Spherical Harmonics Distillation was introduced in the context of Mobile-GS, a framework for real-time Gaussian Splatting on mobile devices [S2, unofficial]. It addresses the high parameter count and storage burden of the original 3DGS method which uses third-order SH [S2, unofficial]. The method builds on the concept of knowledge distillation, adapting it to the specific structure of Gaussian splatting's directional radiance representation [S2, unofficial]. It is part of a broader effort to compress 3DGS for edge hardware [S1].

## Sources
[S1] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S2] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29 (unofficial)
