# Domain-Specific and Advanced Configurations


## Overview
This family encompasses training configurations tailored to specific domains (e.g., humans, objects), advanced physical properties (PBR, HDR), or specific optimization stages (fine representation). It solves the problem of applying generic NeRF training loops to complex inverse rendering tasks where standard assumptions (e.g., Lambertian surfaces, static scenes, LDR inputs) fail. It differs from generic [Training Configuration](2332-training-configuration.md) by introducing domain-specific loss terms, physical constraints, and specialized data handling.

## Shared principles
All configurations in this family extend the standard volume rendering loss $\mathcal{L}_{rec}$ by adding domain-specific regularization or physical constraints. They typically involve:
1.  **Specialized Loss Terms:** Adding losses $\mathcal{L}_{domain}$ (e.g., PBR energy conservation, Eikonal, geometric priors) to the total loss $\mathcal{L} = \mathcal{L}_{rec} + \sum \lambda_i \mathcal{L}_i$.
2.  **Multi-Stage or Joint Optimization:** Decoupling geometry, material, and illumination estimation into sequential phases or optimizing them jointly with specific freezing/unfreezing strategies.
3.  **Domain-Specific Data Handling:** Modifying ray sampling or query points to handle dynamic bodies, HDR inputs, or specific object categories.

## Subtopics

### [Training Objective for Fine Representation](4686-training-objective-for-fine-representation.md)
Defines the loss for the high-resolution radiance field after coarse convergence. It reduces regularization weights to prevent over-smoothing of high-frequency details.

### [PBR-NeRF Joint Optimization](21723-pbr-nerf-joint-optimization.md)
Decouples geometry, material, and illumination into three phases (Geometry, Material, Joint) before final refinement. It optimizes for material parameters (roughness, metallicness) and incident lighting using physics-based losses (energy conservation, inter-reflection).

### [H-NeRF Training Configuration](21746-h-nerf-training-configuration.md)
Specifies hyperparameters for training humans in motion using the imGHUM parametric model. It addresses memory limitations by limiting query points and distinguishes between dynamic human regions and static backgrounds.

### NeRF-RPN Training Augmentation and Optimization
*   **Step:** Randomly flip input scenes along x and y axes with probability 0.5. Randomly rotate input scenes along z-axis by $\pi/2$ with probability 0.5. Randomly rotate scenes along z-axis by an angle $\alpha \in [-\pi/18, \pi/18]$ with probability 0.5 [S1].
*   **Parameter:** Optimizer: AdamW with initial learning rate 0.0003 and weight decay 0.001. Loss weight $\lambda = 5.0$ for Eq. (3) and $\lambda = 1.0$ for Eq. (6) [S1].

### HDR NeRF Training
*   **Definition:** A variant of NeRF trained with 32-bit HDR images to produce 3-channel radiance in linear color space, reconciling the color space difference between NeRF (sRGB) and path tracing (linear) [S2].
*   **Step:** Train the NeRF using 32-bit HDR images as ground truth supervision instead of standard 8-bit images [S2].
*   **Parameter:** Color depth: 32-bit per channel for HDR NeRF, 8-bit per channel for standard LDR NeRF [S2].
*   **Comparison:** HDR NeRF achieves higher lighting realism compared to 8-bit LDR NeRF, as shown in Figure 5 [S2].

### NeRF Personalization
*   **Definition:** A procedure to enhance visual quality for a specific subject or adapt to an unseen identity by fine-tuning the network [S3, unofficial].
*   **Step:** 1. Fine-tune network with learning rate $10^{-5}$ for few iterations. 2. Keep weights of multiplicative module frozen. 3. Learn identity and latent codes for the target subject [S3, unofficial].
*   **Parameter:** Learning rate for personalization: $10^{-5}$ [S3, unofficial].

## Background
The need for domain-specific configurations arises from the ill-posed nature of inverse rendering. Standard NeRF assumes Lambertian surfaces and static scenes, which fails for dynamic humans [S1], physically based materials [S1], or high-dynamic-range lighting [S2]. The concept of multi-stage optimization (coarse-to-fine) is standard in NeRF [S1, S2], while joint optimization of geometry and material is introduced by PBR-NeRF [S1]. HDR NeRF addresses the color space mismatch between neural rendering and physical simulation [S2]. Personalization techniques leverage few-shot fine-tuning to adapt general models to specific identities [S3, unofficial].

## Sources
[S1] NeRF-RPN: A general framework for object detection in NeRFs, https://doi.org/10.1109/cvpr52729.2023.02253, fetched 2026-10-26
[S2] Dynamic Mesh-Aware Radiance Fields, https://doi.org/10.1109/iccv51070.2023.00042, fetched 2026-10-26
[S3] MI-NeRF: Learning a Single Face NeRF from Multiple Identities, https://arxiv.org/abs/2403.19920v2, fetched 2026-10-26 (unofficial)
