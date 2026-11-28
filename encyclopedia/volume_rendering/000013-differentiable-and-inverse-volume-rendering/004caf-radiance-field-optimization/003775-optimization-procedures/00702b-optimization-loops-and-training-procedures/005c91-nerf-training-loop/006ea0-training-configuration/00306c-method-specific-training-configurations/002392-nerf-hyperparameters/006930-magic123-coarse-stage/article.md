# Magic123 Coarse Stage

Also called: NeRF optimization stage

## Overview
The Magic123 Coarse Stage is a single-image-to-3D optimization procedure that trains a Neural Radiance Field (NeRF) to reconstruct a foreground object from a single reference image [S1, unofficial]. It distinguishes itself by combining Instant-NGP acceleration with multi-modal regularization priors—specifically depth and normal smoothness—derived from off-the-shelf 2D models to prevent geometric collapse and high-frequency artifacts [S1, unofficial]. Unlike standard NeRF training which relies primarily on photometric loss, this method integrates novel view guidance from diffusion priors to constrain the unobserved geometry [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Reference image $I_r$ (foreground object with background).
*   Binary segmentation mask $M$ extracted from $I_r$ [S1, unofficial].
*   Pseudo-depth map $d_r$ from $I_r$ [S1, unofficial].
*   Fixed camera pose $v_r$ (assumed front view) [S1, unofficial].

**Outputs:**
*   Optimized NeRF parameters $\theta$ [S1, unofficial].
*   Rendered RGB image $G_\theta(v_r)$ [S1, unofficial].
*   Rendered depth map $d$ [S1, unofficial].
*   Rendered normal map $n$ [S1, unofficial].

### Data Structures
The implementation uses the Instant-NGP [57] NeRF representation [S1, unofficial].
*   **NeRF Parameters ($\theta$):** Contains the hash grid encodings and MLP weights for density ($\sigma$) and color ($c$) [S1, unofficial].
*   **Mask Integration:** The foreground mask $M(G_\theta(v_r))$ is acquired by integrating volume density along the ray [S1, unofficial].
*   **Normal Estimation:** Normal vectors are estimated via finite differences of the rendered depth map [S1, unofficial].

### Numbered Steps

1.  **Preprocessing:**
    *   Extract binary segmentation mask $M$ from $I_r$ using a Dense Prediction Transformer [67] [S1, unofficial].
    *   Extract pseudo-depth map $d_r$ from $I_r$ using a pretrained MiDaS [68] model [S1, unofficial].
    *   Assign fixed camera pose $v_r$ to the reference image [S1, unofficial].

2.  **Rendering:**
    *   Render the reference view $G_\theta(v_r)$ using the current NeRF parameters $\theta$ [S1, unofficial].
    *   Render the background as pure white, as no background is modeled [S1, unofficial].
    *   Compute the rendered foreground mask $M(G_\theta(v_r))$ by integrating volume density along each ray [S1, unofficial].
    *   Compute the rendered depth map $d$ [S1, unofficial].
    *   Compute the rendered normal map $n$ using finite differences on $d$ [S1, unofficial].

3.  **Loss Computation:**
    *   Compute Reference View Reconstruction Loss $L_{rec}$ [S1, unofficial].
    *   Compute Novel View Guidance Loss $L_g$ (details in §2.2 of source) [S1, unofficial].
    *   Compute Depth Prior Loss $L_d$ [S1, unofficial].
    *   Compute Normal Smoothness Loss $L_n$ [S1, unofficial].

4.  **Optimization:**
    *   Compute total loss $L_c$ [S1, unofficial].
    *   Update $\theta$ using gradient descent [S1, unofficial].

### Formulas

**Reference View Reconstruction Loss ($L_{rec}$):**
$$L_{rec} = \lambda_{rgb} \| M \odot (I_r - G_\theta(v_r)) \|_2^2 + \lambda_{mask} \| M - M(G_\theta(v_r)) \|_2^2$$
Where:
*   $\lambda_{rgb}, \lambda_{mask}$ are weights for RGB and mask losses [S1, unofficial].
*   $\odot$ is the Hadamard product [S1, unofficial].
*   $M(G_\theta(v_r))$ is the foreground mask acquired by integrating volume density along the ray [S1, unofficial].

**Depth Prior Loss ($L_d$):**
$$L_d = \frac{1}{2} \left( 1 - \frac{\text{cov}(M \odot d_r, M \odot d)}{\sigma(M \odot d_r)\sigma(M \odot d)} \right)$$
Where:
*   $d_r$ is the pseudo depth from MiDaS [S1, unofficial].
*   $d$ is the NeRF depth output [S1, unofficial].
*   $\text{cov}(\cdot)$ denotes covariance [S1, unofficial].
*   $\sigma(\cdot)$ measures standard deviation [S1, unofficial].
*   This normalized negative Pearson correlation avoids value mismatch issues inherent in MSE for depth [S1, unofficial].

**Normal Smoothness Loss ($L_n$):**
$$L_n = \| n - \tau(g(n, k)) \|$$
Where:
*   $n$ is the rendered normal map [S1, unofficial].
*   $\tau(\cdot)$ is the stop-gradient operation [S1, unofficial].
*   $g(\cdot)$ is a Gaussian blur with kernel size $k=9 \times 9$ [S1, unofficial].

**Total Coarse Loss ($L_c$):**
$$L_c = L_{rec} + L_g + \lambda_d L_d + \lambda_n L_n$$
Where:
*   $L_g$ is the novel view guidance loss [S1, unofficial].
*   $\lambda_d, \lambda_n$ are weights for depth and normal regularizations [S1, unofficial].

### Parameters
*   **Gaussian Blur Kernel Size ($k$):** $9 \times 9$ [S1, unofficial].
*   **Background Color:** Pure white [S1, unofficial].
*   **Weights ($\lambda_{rgb}, \lambda_{mask}, \lambda_d, \lambda_n$):** Not given in the sources.
*   **Instant-NGP Settings:** Not given in the sources.

### Edge Cases and Pitfalls
*   **Flat Geometry Collapse:** Without depth and normal regularization, the model may generate textures that appear on the surface without capturing actual geometric details [S1, unofficial].
*   **High-Frequency Artifacts:** NeRFs tend to produce high-frequency artifacts on object surfaces; the normal smoothness loss mitigates this [S1, unofficial].
*   **Depth Value Mismatch:** Using MSE for depth loss is suboptimal due to value mismatches between pseudo-depth and NeRF depth; normalized correlation is used instead [S1, unofficial].
*   **Background Modeling:** The background is not modeled and rendered as pure white; this assumes the input image has a separable foreground [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Fast inference and optimization due to the use of Instant-NGP [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Handles complex topological changes smoothly; recovers complex geometry; prevents flat geometry and high-frequency artifacts [S1, unofficial].
*   **Limitations:** Relies on off-the-shelf 2D priors (MiDaS, Dense Prediction Transformer) which may introduce noise or inaccuracies; requires fixed camera poses [S1, unofficial].
*   **When to use:** Single-image-to-3D generation where fast convergence and geometric plausibility are prioritized over photorealistic texture fidelity in the coarse stage [S1, unofficial].
*   **When not to use:** Multi-view reconstruction where camera poses are unknown or dynamic; scenes with complex backgrounds that cannot be segmented [S1, unofficial].

## Background
Magic123 is a pipeline for object-level image-to-3D generation [S1, unofficial]. The coarse stage was introduced in "Magic123: One Image to High-Quality 3D Object Generation Using Both 2D and 3D Diffusion Priors" [S1, unofficial]. It builds on the Instant-NGP implementation [57] and integrates diffusion priors for novel view guidance [S1, unofficial]. The stage is followed by a fine stage using DMTet for high-resolution refinement [S1, unofficial].

## Sources
[S1] Magic123: One Image to High-Quality 3D Object Generation Using Both 2D and 3D Diffusion Priors, https://doi.org/10.48550/arxiv.2306.17843, fetched 2026-10-30 (unofficial)
