# GauFRe

Also called: Gaussian Reconstruction

## Overview
GauFRe is a method for dynamic 3D Gaussian Splatting that decouples static and dynamic scene components to improve optimization stability and reconstruction quality [S1]. It addresses the difficulty of jointly optimizing deformation fields and static geometry by initializing the point cloud into separate static and dynamic subsets, optimizing them independently, and rendering them together [S1]. Unlike methods that use positional encoding for time, GauFRe employs compact MLPs for different attributes and applies specific normalization operations to scaling and rotation parameters to ensure reasonable optimization [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of images capturing a dynamic scene with significant static background.
*   **Outputs:** A set of 3D Gaussians representing the scene, categorized as static or dynamic, with parameters optimized for novel view synthesis.

**Data Structures:**
The method maintains two distinct sets of 3D Gaussians:
1.  **Static Gaussians:** Parameters (position, scale, rotation, color, opacity) are constant over time.
2.  **Dynamic Gaussians:** Parameters are modulated by a deformation field represented by an MLP.

**Steps:**
1.  **Initialization:** Randomly initialize the point cloud into two disjoint sets: static point clouds and dynamic point clouds [S1].
2.  **Decoupled Optimization:**
    *   Optimize the static 3D Gaussians first to establish a stable background [S1].
    *   Optimize the deformation field (MLP) for the dynamic Gaussians [S1].
    *   The MLP predicts delta values for scaling and rotation.
3.  **Parameter Normalization:**
    *   For **Scaling**: Apply an exponential operation to the predicted delta values to ensure positive scale values [S1].
    *   For **Rotation**: Apply a normalization operation to the predicted delta values (represented as quaternions or rotation vectors) to ensure they remain valid unit rotations [S1].
4.  **Rendering:** Render both static and dynamic Gaussians together to produce the final image [S1].

**Formulas:**
Let $\Delta \mathbf{s}$ and $\Delta \mathbf{q}$ be the delta values predicted by the MLP for scale and rotation, respectively.
*   Final Scale $\mathbf{s}$:
    $$ \mathbf{s} = \exp(\Delta \mathbf{s}) $$
    (Ensures $\mathbf{s} > 0$) [S1].
*   Final Rotation $\mathbf{q}$:
    $$ \mathbf{q} = \text{normalize}(\mathbf{q}_{base} + \Delta \mathbf{q}) $$
    (Ensures $\|\mathbf{q}\| = 1$) [S1].

**Parameters:**
*   **MLP Architecture:** Compact MLPs are used for different attributes [S1].
*   **Performance:** On the D-NeRF dataset, GauFRe achieves a PSNR of 34.80, SSIM of 0.985, and LPIPS of 0.028 [S1].

**Edge Cases and Pitfalls:**
*   **Optimization Stability:** Directly learning scale and rotation without normalization can lead to unstable gradients. The exponential and normalization operations mitigate this [S1].
*   **Static/Dynamic Separation:** Incorrect initialization or mixing of static and dynamic components can lead to artifacts where static parts appear to deform or dynamic parts fail to move correctly [S1].

## Subtopics

### Foreground Gaussian Scale Residual Learning
A variant that initializes 3D Gaussian scales using KNN distances and learns scale residuals from volume latent features via a decoder to improve convergence and model capacity [S2].
*   **Definition:** Instead of directly learning scales $\mathbf{s}$, it initializes $\mathbf{s}_{init}$ using the average distance of $K$ nearest neighbors via the KNN algorithm and learns the scale residual $\Delta \mathbf{s}$ from the volume latent feature $\mathbf{f}_i$ via a decoder $\mathcal{D}_{cov}$ [S2].
*   **Steps:**
    1.  Initialize scale vector $\mathbf{s}_{init}$ using the average distance of $K$ nearest neighbors via the KNN algorithm [S2].
    2.  Decode scale residual $\Delta \mathbf{s}$ from volume latent feature $\mathbf{f}_i$ using decoder $\mathcal{D}_{cov}$ [S2].
    3.  Compute final scale $\mathbf{s} = \mathbf{s}_{init} + \Delta \mathbf{s}$ [S2].
    4.  Use a 3D vector $\mathbf{s}$ for scaling and a quaternion $\mathbf{q}$ for rotation separately to allow independent optimization [S2].
*   **Formula:**
    $$ \mathbf{s} = \mathbf{s}_\text{init} + \Delta \mathbf{s}, \quad \Delta \mathbf{s} = \mathcal{D}_{cov}(\mathbf{f}_i) $$
    [S2].
*   **When to choose:** Use when faster convergence and enhanced model capacity are required, particularly in scenarios where initial scale estimation is critical [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On the D-NeRF dataset, GauFRe achieves PSNR 34.80, SSIM 0.985, and LPIPS 0.028 [S1].
*   **Strengths:** Decouples static and dynamic parts for stable training; uses compact MLPs for efficiency; normalization operations ensure reasonable optimization [S1].
*   **Limitations:** Requires careful initialization of static vs. dynamic points; performance may vary on scenes with complex dynamic interactions not captured by the MLP [S1].
*   **When to use:** Suitable for dynamic scene reconstruction with significant static backgrounds where joint optimization is unstable [S1].
*   **When not:** Not suitable for purely static scenes where simpler 3DGS is sufficient [S1].

## Background
GauFRe was introduced as a method to handle dynamic 3D Gaussian Splatting by decoupling static and dynamic components [S1]. It was developed to address the challenges of optimizing deformation fields alongside static geometry, which can lead to unstable training [S1]. The method is cited in recent surveys on 3D Gaussian Splatting advancements [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
