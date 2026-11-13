# GauFRe

Also called: Gaussian Field Reconstruction

## Overview
GauFRe is a method for dynamic scene reconstruction that extends 3D Gaussian Splatting by decoupling static and dynamic components and applying specific parameterization constraints to ensure stable optimization [S1]. It solves the problem of training instability and convergence issues in dynamic scenes by using compact MLPs for attribute updates and separating the point cloud into static and dynamic subsets [S1]. Unlike standard 3DGS which optimizes all attributes directly, GauFRe applies exponential and normalization operations to scale and rotation residuals predicted by an MLP, ensuring the parameters remain within reasonable bounds [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of camera poses and corresponding images of a dynamic scene.
*   **Output:** A set of 3D Gaussians representing the scene, categorized as static or dynamic, with optimized positions, rotations, scales, opacities, and appearance coefficients.

**Data Structures:**
The method maintains two distinct sets of 3D Gaussians:
1.  **Static Gaussians:** Representing the non-moving parts of the scene.
2.  **Dynamic Gaussians:** Representing moving objects or parts of the scene.
Each Gaussian $i$ retains the standard 3DGS attributes: position $\mathbf{p}_i$, rotation $q_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and spherical harmonic coefficients $h_i$ [S1].

**Steps:**
1.  **Initialization:** Randomly initialize the point cloud into two disjoint sets: dynamic point clouds and static point clouds [S1].
2.  **Attribute Prediction:** Use compact Multi-Layer Perceptrons (MLPs) to predict delta values (residuals) for scaling and rotation attributes [S1].
3.  **Parameterization:**
    *   For **Scale**: Apply an exponential operation to the predicted delta value to ensure positive scale values.
    *   For **Rotation**: Apply a normalization operation to the predicted delta value (added to the current rotation) to ensure the result remains a valid unit quaternion [S1].
4.  **Optimization:**
    *   Optimize the static Gaussians and the dynamic Gaussians (including their deformation fields) separately but concurrently [S1].
    *   The dynamic part is represented by an MLP that models the deformation over time [S1].
5.  **Rendering:** Render the static and dynamic Gaussians together to produce the final image [S1].

**Formulas:**
Let $\Delta \mathbf{s}_i$ and $\Delta q_i$ be the delta values predicted by the MLP for Gaussian $i$.
The final scale $\mathbf{s}_i$ is computed as:
$$ \mathbf{s}_i = \exp(\Delta \mathbf{s}_i) $$
*(Note: The source states "exponential ... operation to the scaling", implying the base scale is either identity or the exponential is applied to the residual to ensure positivity. Standard 3DGS uses log-scale, but GauFRe explicitly mentions exponential operation on the delta.)*

The final rotation $q_i$ is computed by adding the delta to the current rotation and normalizing:
$$ q_i = \text{normalize}(q_{i, \text{current}} + \Delta q_i) $$
*(Note: Quaternion addition is not standard; this likely implies a small angle approximation or rotation vector representation where $\Delta q$ is a rotation vector, or the source simplifies the quaternion update. Given the text "normalization operation to the ... rotation", it implies the result of the update is normalized.)*

**Parameters:**
*   **MLP Architecture:** Compact MLPs for different attributes [S1].
*   **Dynamic/Static Split:** Random initialization [S1].

**Edge Cases and Pitfalls:**
*   **Optimization Stability:** Directly learning scale and rotation can lead to instability. The exponential and normalization operations mitigate this [S1].
*   **Dynamic/Static Coupling:** Failing to decouple static and dynamic parts can lead to artifacts where static objects move or dynamic objects blur. The random initialization into two sets helps decouple them [S1].

## Subtopics

### Foreground Gaussian Scale Residual Learning
This variant, associated with EVolSplat, improves convergence and model capacity by learning scale residuals from volume latent features rather than optimizing scales directly [S2]. It initializes 3D Gaussian scales using K-Nearest Neighbors (KNN) distances and decodes scale residuals from volume latent features [S2]. This allows for independent optimization of scaling (using a 3D vector $\mathbf{s}$) and rotation (using a quaternion $\mathbf{q}$) [S2].

**Steps:**
1.  Initialize scale vector $\mathbf{s}_{init}$ using the average distance of $K$ nearest neighbors via the KNN algorithm [S2].
2.  Decode scale residual $\Delta \mathbf{s}$ from volume latent feature $\mathbf{f}_i$ using a decoder $\mathcal{D}_{cov}$ [S2].
3.  Compute final scale $\mathbf{s} = \mathbf{s}_{init} + \Delta \mathbf{s}$ [S2].
4.  Use a 3D vector $\mathbf{s}$ for scaling and a quaternion $\mathbf{q}$ for rotation separately to allow independent optimization [S2].

**Formula:**
$$ \mathbf{s} = \mathbf{s}_\text{init} + \Delta \mathbf{s}, \quad \Delta \mathbf{s} = \mathcal{D}_{cov}(\mathbf{f}_i) $$

**When to choose:** Use this variant when faster convergence and enhanced model capacity are required, particularly in urban view synthesis scenarios where volume latent features are available [S2].

## Operating characteristics
*   **Performance:** On the D-NeRF dataset, GauFRe achieves a PSNR of 34.80, SSIM of 0.985, and LPIPS of 0.028 [S1].
*   **Complexity:** The method uses compact MLPs, suggesting lower memory overhead compared to larger neural fields, but requires maintaining two sets of Gaussians (static and dynamic) [S1].
*   **Strengths:** Decoupling static and dynamic parts improves reconstruction quality for scenes with large static backgrounds [S1]. The parameterization ensures stable training [S1].
*   **Limitations:** Requires careful initialization of static/dynamic splits [S1].

## Background
GauFRe was introduced in the context of advancing 3D Gaussian Splatting for dynamic scenes. It addresses the limitations of standard 3DGS in dynamic environments by introducing a decoupled optimization strategy and specific parameterization techniques [S1]. The method is cited as [69] in the source literature, which corresponds to the paper "GauFRe: Gaussian Field Reconstruction for Dynamic Scenes" (inferred from context and citation style).

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
