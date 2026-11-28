# NeRF-W Hyperparameter Configuration
Also called: NeRF-W Training Parameters

## Overview
NeRF-W (NeRF in the Wild) is a method for reconstructing 3D scenes from unconstrained, in-the-wild photo collections, specifically addressing the presence of transient objects (e.g., pedestrians, vehicles) and varying lighting conditions [S1]. It solves the problem of geometric reconstruction in dynamic environments by decomposing the scene representation into static geometry and transient appearance components [S1]. What sets it apart from standard NeRF is the explicit modeling of transient effects via a separate neural head and appearance embeddings, allowing the static geometry MLP to converge on the underlying scene structure rather than being dominated by view-dependent transient artifacts [S1, S2].

## How it works
The method employs a multi-branch MLP architecture to separate static scene geometry from transient, view-dependent effects. The training process minimizes photometric error while regularizing the transient components to prevent them from absorbing static geometry [S1, S2].

**Inputs and Outputs**
*   **Inputs:** A set of uncalibrated or weakly calibrated images from an unconstrained collection; optional segmentation masks for transient objects [S1, S2].
*   **Outputs:** A static signed distance field (SDF) or density field representing the static scene geometry, and a transient field representing dynamic objects and lighting variations.

**Data Structures**
The model consists of three distinct MLPs:
1.  **Base MLP:** 8 layers, 512 hidden units. Encodes the static scene density and base radiance [S1].
2.  **Static MLP:** 4 layers, 128 hidden units. Refines the static radiance [S1].
3.  **Transient/Uncertainty MLP:** 4 layers, 128 hidden units. Encodes transient appearance and uncertainty [S1].

**Positional Encoding**
*   Position: 15 frequencies [S1].
*   Viewing direction: 4 frequencies [S1].

**Latent Embeddings**
*   Appearance embedding dimension $n(a) = 48$ [S1].
*   Transient object embedding dimension $n(\tau) = 16$ [S1].

**Sampling**
*   Training: 512 points per ray per model (coarse and fine), totaling 1024 points per ray [S1].
*   Evaluation: 2048 points per ray [S1].

**Optimization Parameters**
*   Optimizer: Adam [S1].
*   Batch size: 2048 [S1].
*   Initial learning rate: 0.001 [S1].
*   Learning rate schedule: Decays ten-fold every 150,000 iterations [S1].
*   Total iterations: 300,000 [S1].
*   L1 regularizer multiplier for transient uncertainty: $\lambda_u = 0.01$ [S1].
*   Minimum importance threshold: $\beta_{min} = 0.03$ [S1].

**Algorithm Steps**
1.  **Preprocessing:** Filter input images to remove low-quality images (e.g., NIMA score < 3) and images where transient objects occupy >80% of the area [S1].
2.  **Initialization:** Initialize the Base, Static, and Transient MLPs. Initialize appearance embeddings for each image and transient embeddings for detected dynamic objects [S1].
3.  **Ray Sampling:** For each training batch, sample 512 points along each ray for the coarse network and 512 for the fine network [S1].
4.  **Forward Pass:**
    *   Encode position and viewing direction using positional encodings.
    *   Pass through Base MLP to get static density and base color.
    *   Pass through Static MLP to refine static color.
    *   Pass through Transient MLP, conditioned on the transient embedding, to get transient color and uncertainty [S1].
5.  **Rendering:** Composite the static and transient contributions along the ray to produce the final pixel color [S1].
6.  **Loss Calculation:**
    *   Compute photometric loss (MSE or L1) between rendered and ground-truth colors [S1].
    *   Apply L1 regularization on the transient uncertainty term weighted by $\lambda_u = 0.01$ [S1].
    *   If segmentation masks are available, exclude rays belonging to dynamic objects from the static geometry supervision to prevent the transient head from dominating the geometry [S2].
7.  **Update:** Update weights using Adam optimizer with the specified learning rate schedule [S1].
8.  **Evaluation:** During evaluation, double the number of samples per ray to 2048 [S1].

**Pitfalls and Edge Cases**
*   **Transient Dominance:** If the transient NeRF head is not properly regularized or if segmentation masks are not used, the transient head may converge faster than the geometry MLP, causing the network to model static scene structures as view-dependent transient effects [S2].
*   **Sky Reconstruction:** Textureless sky regions lack motion parallax. Using a background NeRF for foreground-background separation can lead to reconstructions in spherical shells. Background rays should be labeled with semantic masks and penalized as free space [S2].

## Subtopics

### Transient Object Handling via Segmentation Masks
A method to handle dynamic objects in Internet photo collections by using segmentation masks to exclude rays belonging to dynamic objects during training [S2]. This prevents the transient NeRF head from dominating the rendered color, which occurs because the transient head converges faster than the geometry MLP, leading the network to model scene structures as view-dependent transient effects rather than static geometry [S2].

## Operating characteristics
*   **Time Complexity:** Training requires 300,000 iterations with a batch size of 2048 [S1]. The use of multiple MLPs (Base, Static, Transient) increases computational cost compared to standard NeRF [S1].
*   **Memory Complexity:** The model stores three MLPs and latent embeddings for appearance ($n(a)=48$) and transient objects ($n(\tau)=16$) per image [S1].
*   **Performance:** The hyperparameters were optimized for the Brandenburg Gate scene of the Phototourism dataset and reused for remaining scenes, suggesting potential sensitivity to scene characteristics [S1].
*   **Strengths:** Effectively handles unconstrained photo collections with transient objects and varying lighting [S1].
*   **Limitations:** Requires careful handling of transient objects to avoid geometric artifacts; performance may degrade if segmentation masks are inaccurate or unavailable [S2].

## Background
NeRF-W was introduced by Mildenhall et al. in 2021 [S1]. It extends the original NeRF framework to handle "in-the-wild" photo collections, which are characterized by uncontrolled capture conditions, transient objects, and varying illumination [S1]. The method was developed to address the limitations of standard NeRF, which assumes a static scene and controlled lighting [S1]. The hyperparameters were determined via grid search on the Brandenburg Gate scene from the Phototourism dataset [S1].

## Sources
[S1] NeRF in the Wild: Neural Radiance Fields for Unconstrained Photo Collections, https://doi.org/10.1109/cvpr46437.2021.00713, fetched 2026-10-26
[S2] Neural 3D Reconstruction in the Wild, https://doi.org/10.1145/3528233.3530718, fetched 2026-10-29
