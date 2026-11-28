# HUGS

Also called: HuGS (Human Gaussian Splatting)

## Overview
HuGS is a method for creating and animating virtual human avatars using 3D Gaussian Splatting (3DGS) by defining a coarse-to-fine deformation algorithm that combines forward Linear Blend Skinning (LBS) with local learning-based refinement via a Multi-Layer Perceptron (MLP) [S2]. It distinguishes itself from NeRF-based formulations by bypassing difficult inverse skinning approaches, allowing it to fit body shapes with loose clothing and achieve real-time rendering speeds [S2]. It builds on the tri-plane representation to factorize the canonical space, enabling reconstruction from monocular video [S1][S3, unofficial].

## How it works
**Inputs:** Monocular video sequence (50–100 frames) of a human subject [S1][S3, unofficial]. Optional assistance from Segment Anything Model (SAM) and LiDAR depth data for scene segmentation and initialization [S3, unofficial].
**Outputs:** A set of animated 3D Gaussians representing the human avatar and scene, renderable at high frame rates [S2].

**Data Structures:**
1.  **3D Gaussians:** Each Gaussian $i$ is defined by:
    *   Position $\mathbf{p}_i$ in canonical space.
    *   Covariance matrix $\mathbf{\Sigma}_i$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$).
    *   Opacity $\alpha_i$.
    *   Spherical Harmonics coefficients $\mathbf{c}_i$ for view-dependent color.
    *   **Per-Gaussian Latent Code:** A learnable vector $\mathbf{z}_i \in \mathbb{R}^{16}$ [S2].
    *   **Per-Gaussian Skinning Weights:** Learnable weights $\mathbf{w}_i \in \mathbb{R}^K$ (where $K$ is the number of joints), learned during training rather than imported from a template mesh [S2].
2.  **Skeleton:** A predefined skeleton structure (e.g., SMPL-like) with joint positions $\mathbf{J}_j$ and rotation matrices $\mathbf{Q}_j$ for a given pose [S2].
3.  **MLP:** A small neural network taking $\mathbf{z}_i$ and pose information as input, outputting a transformation offset [S2].

**Parameters:**
*   Latent code dimension: 16 floats [S2].
*   Rendering resolution: 512x512 pixels (standard evaluation) [S2].
*   Training time: 5–20 hours on Tesla V100 [S2].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize static 3D Gaussians for the background and static parts of the scene [S3, unofficial].
    *   Initialize dynamic 3D Gaussians for the human subject in a canonical pose (e.g., T-pose) [S2].
    *   Initialize per-gaussian latent codes $\mathbf{z}_i$ and skinning weights $\mathbf{w}_i$ [S2].

2.  **Coarse Deformation (LBS):**
    *   For each Gaussian $i$ and a target pose defined by joint rotations $\mathbf{Q}_j$:
    *   Compute the Linear Blend Skinning transformation. Let $T_{LBS}(\mathbf{p}_i, \mathbf{w}_i, \{\mathbf{Q}_j\})$ be the transformed position.
    *   The weight $\mathbf{w}_i$ determines the influence of each joint $j$ on Gaussian $i$. Unlike template-based methods, these weights are learned, allowing adaptation to any body shape and correction of pose estimation errors [S2].
    *   $\mathbf{p}'_i = T_{LBS}(\mathbf{p}_i, \mathbf{w}_i, \{\mathbf{Q}_j\})$ [S2].

3.  **Fine Deformation (MLP Refinement):**
    *   The coarse LBS deformation may lack non-rigid details (e.g., garment wrinkles) and may not perfectly align with the observed data.
    *   Input the per-gaussian latent code $\mathbf{z}_i$ and the current pose information into the MLP [S2].
    *   The MLP outputs a translation $\Delta \mathbf{t}_i$ and rotation $\Delta \mathbf{R}_i$ (or a full SE(3) transformation) to refine the deformation [S2].
    *   $\mathbf{p}''_i = \mathbf{p}'_i + \Delta \mathbf{t}_i$
    *   $\mathbf{R}''_i = \Delta \mathbf{R}_i \mathbf{R}'_i$ (where $\mathbf{R}'_i$ is the rotation from LBS) [S2].
    *   *Note:* The MLP input uses the compact latent code $\mathbf{z}_i$ instead of canonical position with Fourier features, which is more efficient and helps decode information in small MLPs [S2].

4.  **Rendering:**
    *   Project the transformed Gaussians ($\mathbf{p}''_i, \mathbf{R}''_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_i$) onto the 2D image plane using standard 3DGS rasterization [S2].
    *   Composite the colors based on opacity and view-dependent SH coefficients [S2].

5.  **Optimization:**
    *   Minimize the reconstruction loss (e.g., L1 loss + SSIM) between the rendered image and the ground truth image from the input video [S2].
    *   Optimize Gaussian parameters (position, scale, rotation, opacity, SH coefficients), latent codes $\mathbf{z}_i$, and skinning weights $\mathbf{w}_i$ [S2].
    *   The process is assisted by SAM and LiDAR depth data to separate static and dynamic components and improve initialization [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Sparse Camera Setups:** Degrade 3D reconstruction quality due to overfitting on training observations, resulting in poor novel pose synthesis [S2].
*   **Extrapolation:** The MLP does not extrapolate novel deformations; it only fits observed garment deformations and replicates them for novel poses [S2].
*   **Independence:** Each Gaussian is optimized and deformed independently, ignoring relations between Gaussians in local neighborhoods [S2].
*   **Shading Component:** Removing the MLP's shading/refinement component forces the model to learn duplicate Gaussians with different colors for shaded areas, leading to overfitting [S2].

## Operating characteristics
*   **Rendering Speed:** ~12ms (80 FPS) on a Tesla V100 GPU for 512x512 resolution. Transformation computation takes 10ms; Gaussian rasterization takes 2ms [S2]. Another source cites 20 FPS [S1].
*   **Training Time:** 5 to 20 hours on a Tesla V100, depending on dataset size [S2].
*   **Memory:** Not given in the sources.
*   **Strengths:** Real-time rendering; bypasses inverse skinning; fits loose clothing; generalizes well to novel poses compared to NeRF-based methods [S2].
*   **Limitations:** Poor performance with sparse camera setups; cannot extrapolate unseen deformations; ignores local Gaussian correlations [S2].
*   **When to use:** Real-time animation of human avatars from monocular video; applications requiring fast rendering and fitting of complex clothing [S2].
*   **When not to use:** Scenes with sparse camera views; applications requiring extrapolation of novel, unseen deformations [S2].

## Background
HuGS was introduced in the paper "Human Gaussian Splatting: Real-Time Rendering of Animatable Avatars" [S2]. It was developed to address the limitations of NeRF-based human avatars, which suffer from slow rendering and complex inverse skinning requirements. By leveraging 3D Gaussian Splatting, HuGS achieves real-time performance while maintaining high fidelity. It builds on the tri-plane representation to factorize the canonical space [S1][S3, unofficial]. The method has been cited in surveys of 3D Gaussian Splatting advancements [S1][S3, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Human Gaussian Splatting: Real-Time Rendering of Animatable Avatars, https://doi.org/10.1109/cvpr52733.2024.00081, fetched 2026-10-26
[S3] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
