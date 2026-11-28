# GaussianFlow

Also called: DDDM

## Overview
GaussianFlow is a point-based differentiable rendering method for dynamic 3D scene reconstruction that models a 4D scene as a set of deformable 3D Gaussian particles advected by a 3D velocity field [S1, unofficial][S3]. It solves the problem of efficient, high-fidelity reconstruction from monocular or multi-view video by replacing implicit neural deformation fields with an explicit Dual-Domain Deformation Model (DDDM) [S3]. Unlike its sibling [Dynamic 3D Gaussians](5703-dynamic-3d-gaussian-splatting.md), which tracks time-varying attributes via local-rigid and ARAP constraints, GaussianFlow explicitly models attribute deformations in both time and frequency domains to capture complex motions without neural networks [S3][S5, unofficial]. It differs from [Gaussian Splatting Lucas-Kanade](7333-gaussian-splatting-lucas-kanade.md), which applies optical flow directly to rendered primitives for alignment, by using an explicit flow field for temporal regularization of the underlying Gaussian parameters [S1, S4, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A sequence of images (monocular or multi-view) with known camera poses (or estimated poses) and timestamps $t$.
- **Outputs:** A set of 3D Gaussians $\{G_j\}$ where each Gaussian $j$ has time-varying attributes: position $\boldsymbol{\mu}_j(t)$, rotation $q_j(t)$, scale $\mathbf{s}_j(t)$, and color $\mathbf{c}_j(t)$.

**Data Structures:**
Each Gaussian $j$ stores canonical (static) attributes and deformation coefficients for the DDDM.
- **Canonical Position:** $\boldsymbol{\mu}_j^0 \in \mathbb{R}^3$
- **Canonical Rotation:** $q_j^0 \in \mathbb{R}^4$ (unit quaternion)
- **Canonical Scale:** $\mathbf{s}_j^0 \in \mathbb{R}^3$
- **Canonical Color:** $\mathbf{c}_j^0 \in \mathbb{R}^3$ (or SH coefficients)
- **Deformation Coefficients:** Stored per attribute, representing polynomial and Fourier coefficients.

**Dual-Domain Deformation Model (DDDM):**
The DDDM models the time-dependent residual for each deformable attribute (position, rotation, scale, color) by combining polynomial fitting in the time domain and Fourier approximation in the frequency domain [S3][S5, unofficial].

For a generic attribute $A$ (e.g., position component $x$), the value at time $t$ is:
$$ A_j(t) = A_j^0 + \Delta A_j^{\text{poly}}(t) + \Delta A_j^{\text{fourier}}(t) $$

1.  **Time Domain (Polynomial):** Captures smooth, low-frequency motion.
    $$ \Delta A_j^{\text{poly}}(t) = \sum_{k=1}^{K} \alpha_{j,k}^A t^k $$
    where $\alpha_{j,k}^A$ are learned polynomial coefficients and $K$ is the polynomial order [S3][S5, unofficial].

2.  **Frequency Domain (Fourier):** Captures high-frequency, "violent" motions or oscillations.
    $$ \Delta A_j^{\text{fourier}}(t) = \sum_{m=1}^{M} \left( \beta_{j,m}^A \cos(2\pi m t) + \gamma_{j,m}^A \sin(2\pi m t) \right) $$
    where $\beta_{j,m}^A, \gamma_{j,m}^A$ are learned Fourier coefficients and $M$ is the number of harmonics [S3][S5, unofficial].

**Adaptive Timestep Scaling:**
To prevent optimization problems caused by uniform time division (e.g., overfitting to frames with violent motion), adaptive timestep scaling is applied [S2, S3]. The time $t$ is normalized or scaled based on the temporal distribution of the input frames, ensuring that the polynomial and Fourier bases are well-conditioned across the video duration [S2, S3].

**Optimization Loop:**
The training iterates between static optimization and dynamic optimization [S2][S5, unofficial].
1.  **Static Optimization:** Optimizes canonical attributes $\boldsymbol{\mu}_j^0, q_j^0, \mathbf{s}_j^0, \mathbf{c}_j^0$ while keeping deformation coefficients fixed or zero-initialized.
2.  **Dynamic Optimization:** Optimizes the deformation coefficients ($\alpha, \beta, \gamma$) and potentially refines canonical attributes.

**Loss Functions:**
- **Image Loss:** Standard photometric loss between rendered and input images.
- **Bidirectional Optical Flow Loss:** 2D flow estimation results are introduced into training to support deformation modeling between neighboring frames [S2][S5, unofficial].
- **Temporal Smoothness Loss:** Encourages smooth variation of attributes over time [S2][S5, unofficial].
- **KNN Rigid Loss:** Regularizes the motion trajectory by applying KNN-based rigid constraints, ensuring that neighboring Gaussians move coherently [S3][S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_gaussians(images, timestamps):
    """
    Initialize a set of 3D Gaussians from static 3DGS initialization.
    Returns list of Gaussian objects with canonical attributes and zero deformation coeffs.
    """
    # [G] Standard 3DGS initialization from SfM points or random seeding
    gaussians = []
    for i in range(num_gaussians):
        g = Gaussian()
        g.mu_0 = canonical_position[i]
        g.q_0 = canonical_rotation[i]
        g.s_0 = canonical_scale[i]
        g.c_0 = canonical_color[i]
        # Initialize deformation coefficients to zero
        g.alpha_poly = np.zeros(K)
        g.beta_fourier = np.zeros(M)
        g.gamma_fourier = np.zeros(M)
        gaussians.append(g)
    return gaussians

def compute_deformation(g, t):
    """
    Compute time-varying attributes for Gaussian g at time t using DDDM.
    """
    # Polynomial part
    poly_pos = sum(g.alpha_poly[k] * (t ** (k+1)) for k in range(K))
    
    # Fourier part
    fourier_pos = sum(
        g.beta_fourier[m] * np.cos(2 * np.pi * (m+1) * t) +
        g.gamma_fourier[m] * np.sin(2 * np.pi * (m+1) * t)
        for m in range(M)
    )
    
    # Total deformation residual
    delta_pos = poly_pos + fourier_pos
    
    # Apply to canonical position
    mu_t = g.mu_0 + delta_pos
    
    # Repeat for rotation, scale, color (using respective coefficients)
    # [G] Rotation requires quaternion integration; scale/color are additive/multiplicative
    return mu_t, q_t, s_t, c_t

def adaptive_time_scaling(timestamps):
    """
    Scale timestamps to prevent optimization issues from uniform division.
    """
    # [G] Normalize timestamps to [0, 1] or use a non-linear mapping based on frame density
    t_min = min(timestamps)
    t_max = max(timestamps)
    scaled_ts = [(t - t_min) / (t_max - t_min) for t in timestamps]
    return scaled_ts

def train_gaussian_flow(images, timestamps, num_iterations):
    gaussians = initialize_gaussians(images, timestamps)
    scaled_timestamps = adaptive_time_scaling(timestamps)
    
    for iteration in range(num_iterations):
        # Alternate between static and dynamic optimization
        if iteration % 2 == 0:
            # Static Optimization
            optimize_canonical_attributes(gaussians, images, scaled_timestamps)
        else:
            # Dynamic Optimization
            optimize_deformation_coeffs(gaussians, images, scaled_timestamps)
            
        # Compute losses
        img_loss = compute_image_loss(gaussians, images, scaled_timestamps)
        flow_loss = compute_bidirectional_flow_loss(gaussians, images, scaled_timestamps)
        smooth_loss = compute_temporal_smoothness_loss(gaussians, scaled_timestamps)
        rigid_loss = compute_knn_rigid_loss(gaussians, scaled_timestamps)
        
        total_loss = img_loss + flow_loss + smooth_loss + rigid_loss
        total_loss.backward()
        optimizer.step()
        
    return gaussians
```

## Subtopics

### [Gaussian Splatting Lucas-Kanade](7333-gaussian-splatting-lucas-kanade.md)
Gaussian Splatting Lucas-Kanade is a method that directly applies optical flow estimation to the rendered Gaussians, using the flow to align dynamic objects across frames and improve temporal consistency. It distinguishes itself from parent method [GaussianFlow](6149-gaussianflow.md), which uses an explicit flow field for temporal regularization, by operating directly on the rendered Gaussian primitives rather than an explicit geometric point cloud or a separate flow MLP.

### MotionGS
MotionGS is a dynamic representation method that adds explicit motion supervision to the deformation field by decomposing optical flow into camera and object motion components [S6]. It uses the formula $F^M_{t\to t+1} = F_{t\to t+1} - F^C_{t\to t+1}$, where $F^M_{t\to t+1}$ is the object motion flow, $F_{t\to t+1}$ is the total optical flow, and $F^C_{t\to t+1}$ is the camera-induced flow [S6]. The flow loss is defined as $L_{\text{flow}} = \| s_g(F^M_{t\to t+1}) - F^G_{t\to t+1} \|_1$, where $s_g$ is the stop-gradient operator and $F^G_{t\to t+1}$ is the projected Gaussian flow [S6]. A known pitfall is that MotionGS exhausted 80 GB GPU memory at iteration 7,600 on the DyNeRF flame_steak sequence and did not complete [S6].

## Operating characteristics
- **Training Speed:** Achieves 5× faster training speed compared with per-frame 3DGS modeling [S3].
- **Rendering Speed:** Preserves ultra-fast rendering speed comparable to original 3DGS, enabling real-time rendering [S3].
- **Memory:** More memory-efficient than implicit neural field methods and per-frame 3DGS, as it stores a single set of Gaussians with deformation coefficients rather than duplicating geometry per frame [S3].
- **Strengths:** Handles complex scene deformations across long video footage; supports both multi-view and monocular videos; explicit deformation modeling avoids the slow training/rendering of NeRF-based approaches [S3].
- **Limitations:** Requires careful tuning of polynomial order $K$ and Fourier harmonics $M$; adaptive timestep scaling is necessary to avoid overfitting to violent motions [S3].
- **When to Use:** Dynamic scene reconstruction and real-time rendering from monocular or multi-view videos where training speed and rendering fidelity are critical [S3].
- **When Not to Use:** If the scene has extremely sparse temporal sampling or if the motion is highly non-smooth and cannot be captured by low-order polynomials and Fourier series [S3].

## Background
GaussianFlow was introduced by Yao et al. in 2024 [S3]. It builds on the success of 3D Gaussian Splatting (3DGS) for static scenes and addresses the limitations of NeRF-based dynamic reconstruction methods, which suffer from slow training and rendering speeds [S3]. The method was developed to enable fast dynamic scene reconstruction and real-time rendering from both multi-view and monocular videos [S3]. It is used in applications requiring high-fidelity 4D reconstruction and real-time rendering, such as virtual reality and digital twins [S3].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] Gaussian-Flow: 4D Reconstruction with Dynamic 3D Gaussian Particle, https://doi.org/10.1109/cvpr52733.2024.01997, fetched 2026-10-26
[S4] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S5] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S6] Gaussian Splatting for real-time scene reconstruction: Methods, systems, and trade-offs, https://doi.org/10.1016/j.patcog.2026.115120, fetched 2026-10-26
