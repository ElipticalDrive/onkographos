# Decomposition Network

Also called: Coarse and fine networks

## Overview
The Decomposition Network is a multi-layer perceptron (MLP) architecture used within neural fields to simultaneously reconstruct geometry and decompose material properties (diffuse color, roughness) from multi-view images. It distinguishes itself from standard NeRF architectures by explicitly optimizing for physically-based rendering (PBR) parameters rather than just color, employing a specialized loss schedule to handle the instability of normal estimation and BRDF fitting during early training stages.

## How it works
The method consists of two parallel networks, referred to as the coarse and fine networks, which share the same architectural layout.

**Inputs and Outputs**
*   **Input:** 3D position $\mathbf{x}$, surface normal $\mathbf{n}$, and outgoing view direction $\omega_o$.
*   **Output:** Material parameters (diffuse color, roughness) and/or color, depending on the specific network stage (coarse vs. fine).

**Data Structures**
The architecture comprises 8 MLPs. Each MLP contains 256 features with ReLU activation functions [S1, unofficial].

**Training Procedure**
1.  **Ray Sampling:** For each training image, 4096 ray directions are randomly sampled [S1, unofficial].
2.  **Disparity Sampling:** For real-world data, rays are sampled in disparity space rather than linear depth to concentrate samples closer to the camera [S1, unofficial].
3.  **Forward Pass:** The sampled rays are passed through the coarse and fine networks to predict material properties and rendered colors.
4.  **Loss Calculation:** The total loss is a weighted combination of three terms:
    *   **Main Reconstruction Loss:** Mean Squared Error (MSE) between the rendered color $\hat{x}$ and the ground truth pixel $x^*$ [S1, unofficial].
    *   **Background Segmentation Loss:** Ensures rays that do not hit the object do not contribute to the gradient [S1, unofficial].
    *   **BRDF Priming Loss:** An MSE loss that initializes the diffuse color to the actual image color and sets roughness to 0.3 [S1, unofficial].
5.  **Loss Scheduling:**
    *   The background segmentation loss fades in over 10,000 steps.
    *   The BRDF priming loss fades out over 10,000 steps.
    *   The main reconstruction loss is exponentially faded over 100,000 steps into a cosine-weighted MSE [S1, unofficial].

**Formulas**
The cosine-weighted MSE loss is defined as:
$$ L_{cosine} = \sum (x^* \omega_o \cdot n - \hat{x} \omega_o \cdot n) $$
Where:
*   $x^*$ is the ground truth color.
*   $\hat{x}$ is the predicted color.
*   $\omega_o$ is the outgoing view direction.
*   $n$ is the surface normal [S1, unofficial].

**Parameters**
*   **Optimizer:** Adam [S1, unofficial].
*   **Learning Rate:** $4 \times 10^{-4}$ [S1, unofficial].
*   **Batch Size:** 8 [S1, unofficial].
*   **Total Steps:** 400,000 [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Grazing Angles:** The approximated rendering model is least accurate at grazing angles. The Fresnel effect causes harsh highlights which can destabilize training. The cosine weighting ($\omega_o \cdot n$) reduces the factor of grazing samples to mitigate this [S1, unofficial].
*   **Normal Reliability:** Surface normals $n$ are not reliable in the early stages of training. This necessitates the fading loss scheme, where the main MSE loss is replaced by the cosine-weighted version only after initial convergence [S1, unofficial].

## Operating characteristics
*   **Hardware:** Trained on 4 NVIDIA 2080 Ti GPUs [S1, unofficial].
*   **Time:** Approximately 22 hours for 400,000 steps [S1, unofficial].
*   **Strengths:** Enables reflectance decomposition and BRDF fitting from images alone.
*   **Limitations:** Relies on accurate normal estimation; performance degrades at grazing angles without specific weighting.

## Background
The Decomposition Network is part of the Neural-PIL (Neural Pre-Integrated Lighting) framework, which aims to perform neural pre-integrated lighting for reflectance decomposition. It builds upon previous work in neural radiance fields and material decomposition, specifically addressing the challenge of optimizing latent spaces for material parameters via backpropagation through the decoder [S1, unofficial].

## Sources
[S1] Neural-PIL: Neural Pre-Integrated Lighting for Reflectance Decomposition, https://doi.org/10.48550/arxiv.2110.14373, fetched 2026-10-26 (unofficial)
