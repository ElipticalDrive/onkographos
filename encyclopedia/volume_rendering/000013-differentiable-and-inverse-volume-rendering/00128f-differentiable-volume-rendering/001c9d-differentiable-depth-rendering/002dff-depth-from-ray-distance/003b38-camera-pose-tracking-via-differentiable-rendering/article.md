# Camera Pose Tracking via Differentiable Rendering

Also called: Camera Pose Tracking (Image-to-Model Alignment), Pose Optimization via Volumetric Gradients, Radiance Field Tracking

## Overview
This method estimates the 6-DoF camera pose by treating the pose parameters as the optimization variables while keeping the radiance field parameters (density $\vec{\sigma}$ and color $\vec{c}$) static, performing image-to-model alignment using RGB-D input [S1]. It differs from traditional methods by computing the derivative of the image rendering function with respect to the pose using online learning of an approximate local gradient, thereby avoiding the need for analytical image formation models and explicit 2D-to-3D correspondence solving [S2].

## How it works
**Inputs:**
*   A pre-built volumetric radiance field (e.g., Plenoxels) with fixed density $\sigma$ and color $c$ parameters [S1].
*   A target RGB-D image (reference) [S1, S2].
*   An initial guess for the camera pose (rotation $R$ and translation $t$) [S2].
*   Camera intrinsics $K$ [S2].

**Outputs:**
*   Optimized 6-DoF camera pose $(R, t)$ [S2].
*   Rendered depth $\hat{D}$ and color $\hat{C}$ aligned with the target [S1].

**Data Structures:**
*   **Radiance Field:** Voxel-based representation where density $\sigma_i$ and color $c_i$ are stored at sample points $\vec{p}_i$ along rays [S1].
*   **Pose Parameters:** Rotation matrix $R$ and translation vector $t$. In [S2], rotation is represented using Classical Rodrigues Parameters (CRPs) $q = [q_1, q_2, q_3]^T$ to facilitate polynomial system solving without trigonometric functions [S2].
*   **Ray Definition:** A ray is defined as $\vec{r} = \vec{o} + t_i \vec{d}$, where $\vec{o}$ is the origin and $\vec{d}$ is the direction [S1].

**Steps:**

1.  **Initial Rendering:** Synthesize an RGB image of the object using the current pose guess and the radiance field [S2].
2.  **Feature Extraction:** Extract a set of reference features from the rendered image that match with features from the target reference image [S2].
3.  **Perturbation:** Generate arbitrary views by applying small perturbations to the current pose guess in its neighborhood [S2].
4.  **Gradient Approximation:** Numerically compute an approximate gradient (Jacobian) of the image rendering with respect to the pose parameters using the registered poses and their respective image renderings [S2].
5.  **Optimization:** Use the computed gradient to optimize the view via gradient-based optimization to minimize misalignment error, reproducing the 2D view geometry and estimating the 6-DoF relative pose [S2].

**Formulas:**

The derivative of the rendered depth $\hat{D}$ with respect to the ray $\vec{r}$ is computed using the chain rule [S1]:

$$
\frac{\partial \hat{D}}{\partial \vec{r}} = \sum_{i=1}^{N} \frac{\partial \hat{D}}{\partial \sigma_i} \frac{\partial \sigma_i}{\partial \vec{p}_i} \frac{\partial \vec{p}_i}{\partial \vec{r}}
$$

The derivative of a sample point $\vec{p}_i$ with respect to the ray $\vec{r}$ is decomposed into derivatives with respect to the origin $\vec{o}$ and direction $\vec{d}$ [S1]:

$$
\frac{\partial \vec{p}_i}{\partial \vec{r}} = \left[ \frac{\partial \vec{p}_i}{\partial \vec{o}}, \frac{\partial \vec{p}_i}{\partial \vec{d}} \right] = [1, t_i]
$$

The perspective projection transformation from 3D reference points $a_i$ to view space 3D coordinates $b_i$ is given by [S2]:

$$
b_i = R a_i + t
$$

where $R^T R = I$ and $\text{det}(R) = 1$ [S2].

The perspective projection extends to 2D homogeneous image projections $l_i = [u_i, v_i, 1]^T$ via [S2]:

$$
s l_i = K [R | t] [a_i, 1]^T
$$

where $s$ is the depth factor and $K$ is the intrinsic camera matrix [S2].

**Parameters:**
*   **Optimizer:** Adam [S1]. (Note: RMSProp is used for mapping, but Adam is specified for tracking [S1]).
*   **Perturbation Size:** Not given in the sources.
*   **Feature Matching:** Sparse 2D feature correspondences are used [S2].

**Pitfalls:**
*   The partial derivatives $\frac{\partial \sigma_i}{\partial \vec{p}_i}$, $\frac{\partial c_i}{\partial \vec{p}_i}$, and $\frac{\partial \vec{p}_i}{\partial \vec{r}}$ are model-specific. For trilinear interpolation, these produce messy analytical functions [S1].
*   The method relies on an initial pose guess; if the guess is too far from the true pose, the local gradient approximation may fail [S2].
*   No importance sampling, keyframing, or similar optimization schemes are used, which may affect convergence speed [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Tested on synthetic (Replica) and real-world (ScanNet) indoor datasets [S1].
*   **Strengths:** Avoids solving the 2D-to-3D correspondence problem explicitly; uses online learning for gradient approximation [S2].
*   **Limitations:** Requires a pre-built radiance field; analytical derivatives are complex for trilinear interpolation [S1].
*   **When to use:** When a volumetric model is already available and precise pose estimation is needed using RGB-D data [S1].
*   **When not to use:** When the radiance field is not yet built (use mapping instead) or when analytical gradients are preferred over numerical approximation [S1, S2].

## Background
This method originates from work on RGB-D mapping and tracking in plenoxel radiance fields [S1] and differentiable rendering for pose estimation in proximity operations [S2]. It builds on the concept of differentiable volume rendering [Parent] by treating the camera pose as the differentiable variable. The use of Classical Rodrigues Parameters (CRPs) for rotation representation was introduced to facilitate optimization via polynomial system solving [S2].

## Sources
[S1] RGB-D Mapping and Tracking in a Plenoxel Radiance Field, https://doi.org/10.1109/wacv57701.2024.00331, fetched 2026-10-26
[S2] Differentiable Rendering for Pose Estimation in Proximity Operations, https://core.ac.uk/download/552738213.pdf, fetched 2026-10-30
