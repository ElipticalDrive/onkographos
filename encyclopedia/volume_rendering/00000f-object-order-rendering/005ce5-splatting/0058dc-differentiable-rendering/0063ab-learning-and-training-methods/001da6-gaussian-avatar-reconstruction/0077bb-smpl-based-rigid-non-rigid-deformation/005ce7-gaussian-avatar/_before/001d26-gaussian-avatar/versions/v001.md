# Gaussian Avatar

Also called: 3DGS-Avatar, SMPL-based Gaussian Splatting Avatar

## Overview
Gaussian Avatar is a method for reconstructing animatable full-body human avatars by leveraging the SMPL parametric model as a structural prior [S1][S2, unofficial]. It solves the problem of generating dynamic, high-fidelity 3D humans from video by predicting 3D Gaussian parameters (position, scale, rotation, opacity, and color) conditioned on the avatar's pose [S1][S2, unofficial]. It is distinguished from its sibling [Gaussian Splatting](946-gaussian-splatting.md) by its explicit dependency on a parametric body model to drive deformation, and from [Gaussian Avatars](1477-gaussian-avatars.md) which typically focus on head-only reconstruction using the FLAME model.

## How it works
**Inputs:**
*   A sequence of input images or video frames of a human subject.
*   The SMPL model [S1][S2, unofficial], which provides a parametric representation of human body shape and pose.
*   SMPL parameters: global pose ($\theta$), body shape ($\beta$), and potentially joint angles.

**Outputs:**
*   A set of 3D Gaussians $\{G_i\}$ whose parameters are functions of the SMPL pose.
*   A rendered image sequence consistent with the input views.

**Data Structures:**
*   **Gaussian Parameters:** Each Gaussian $i$ is defined by a mean position $\mu_i$, a covariance matrix $\Sigma_i$ (parameterized by scale $s_i$ and rotation $R_i$), opacity $\alpha_i$, and spherical harmonics coefficients for color $c_i$.
*   **Pose Feature Vector:** A vector derived from the SMPL pose parameters, typically including global rotation and joint angles.
*   **Feedforward Network (MLP):** A neural network that maps pose features to Gaussian parameters.

**Method Steps:**
1.  **SMPL Fitting:** For each input frame, estimate the SMPL parameters (pose $\theta$, shape $\beta$) that best fit the observed image data [S1][S2, unofficial].
2.  **Pose Feature Extraction:** Encode the SMPL pose parameters into a feature vector. This may involve concatenating joint angles or using a pose embedding [S1][S2, unofficial].
3.  **Gaussian Parameter Prediction:** Pass the pose feature vector through a feedforward neural network (MLP) [S1][S2, unofficial]. The network outputs the parameters for each Gaussian:
    $$ \text{Params}_i = \text{MLP}(\text{PoseFeatures}) $$
    where $\text{Params}_i$ includes $\mu_i, s_i, R_i, \alpha_i, c_i$ [S1][S2, unofficial].
4.  **Rendering:** Render the predicted Gaussians using standard differentiable splatting [Gaussian Splatting](946-gaussian-splatting.md) to produce an image.
5.  **Optimization:** Minimize the difference between the rendered image and the input image using gradient descent. The SMPL parameters and/or the MLP weights are optimized to minimize this loss [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Poses:** Extreme poses may lead to inaccurate SMPL fitting or poor generalization by the MLP [G].
*   **Occlusion:** Self-occlusion can make SMPL fitting ambiguous, affecting the accuracy of the predicted Gaussian positions [G].

## Subtopics

### GoMAvatar
A monocular video avatar method that uses the SMPL-X rig and is coupled with a mesh [S3, unofficial]. It differs from standard Gaussian Avatar by integrating mesh constraints, potentially offering better geometric consistency for monocular inputs [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Dependent on the number of Gaussians and the complexity of the feedforward network. Rendering is $O(N)$ where $N$ is the number of Gaussians. Training involves iterative optimization of SMPL parameters and network weights.
*   **Memory Complexity:** $O(N)$ to store Gaussian parameters.
*   **Strengths:** High-fidelity reconstruction of full-body avatars; leverages strong priors from SMPL for plausible anatomy [S1][S2, unofficial].
*   **Limitations:** Requires accurate SMPL fitting; may struggle with non-humanoid subjects or extreme deformations not captured by SMPL [G].
*   **When to use:** When animatable full-body avatars are needed from video data.
*   **When not to use:** For head-only avatars (use [Gaussian Avatars](1477-gaussian-avatars.md)) or when no parametric model prior is desired.

## Background
Gaussian Avatar was introduced as a method to leverage the SMPL model as a prior for reconstructing full-body avatars [S1][S2, unofficial]. It builds on the principles of 3D Gaussian Splatting [Gaussian Splatting](946-gaussian-splatting.md) and applies them to dynamic human reconstruction by conditioning Gaussian parameters on pose [S1][S2, unofficial]. It is closely related to [3DGS-Avatar](1480-3dgs-avatar.md), which also uses SMPL but combines Gaussians with both nonrigid and rigid transformations [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S3] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
