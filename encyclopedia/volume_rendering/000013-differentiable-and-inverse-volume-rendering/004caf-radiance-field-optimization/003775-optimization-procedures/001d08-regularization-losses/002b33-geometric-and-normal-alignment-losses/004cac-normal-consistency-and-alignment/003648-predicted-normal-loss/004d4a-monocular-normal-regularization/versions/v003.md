# Monocular Normal Regularization

Also called: Monocular Normal Loss, Omnidata Normal Supervision, Predicted Normal Loss (Monocular), Stable Normal supervision

## Overview
Monocular Normal Regularization is a geometric loss function that supervises the predicted surface normals of a volume rendering optimization using smooth normal estimates from a pretrained monocular network, rather than noisy gradients of rendered depths. It addresses the artifact-prone nature of gradient-based normal estimation in complex scenes by leveraging external geometric priors. It differs from [Density Gradient Normal Loss](9575-density-gradient-normal-loss.md) by relying on external monocular cues instead of internal density gradients.

## How it works
**Inputs and Outputs:**
- **Inputs:** Rendered normals $\mathbf{N}$ (or predicted normals $\hat{\mathbf{N}}$ depending on the specific implementation variant), monocular normal predictions $\hat{\mathbf{N}}_{mono}$ from a pretrained model (e.g., Omnidata [S1, S3] or Stable Normal [S2]), and optional margin $m$ [S2].
- **Outputs:** Scalar loss value $\mathcal{L}_n$ or $\mathcal{L}_{\hat{N}}$.

**Data Structures:**
- Normals are represented as 3D unit vectors $\mathbf{n} \in \mathbb{R}^3$.
- Monocular predictions are typically per-pixel maps aligned with the camera view.

**Steps:**
1. **Obtain Monocular Cues:** Render or predict the scene from the current camera viewpoint. Pass the input image or rendered depth/color through a pretrained monocular normal estimation network (e.g., Omnidata [S1, S3] or Stable Normal [S2]) to get $\hat{\mathbf{N}}_{mono}$.
2. **Compute Rendered/Predicted Normals:**
   - In Gaussian Splatting contexts, normals are derived from the Gaussian covariance. For a Gaussian $i$ with rotation matrix $R_i$ and scaling coefficients $s_i \in \mathbb{R}^3$, the normal direction is approximated by the axis with the smallest scale:
     $$ \hat{\mathbf{n}}_i = R_i \cdot \text{OneHot}(\arg\min(s_{i,1}, s_{i,2}, s_{i,3})) $$
     [S1]
   - In SDF/SRDF contexts, normals are derived from the gradient of the signed distance field or density function, or composited from sampled points:
     $$ \mathbf{N} = \sum_{i=1}^N T_i \alpha_i \mathbf{n}_i $$
     [S3]
3. **Calculate Loss:**
   - **L1 Alignment (Omnidata):** Minimize the L1 distance between predicted normals $\hat{\mathbf{N}}$ and monocular cues $\mathbf{N}$:
     $$ \mathcal{L}_{\hat{N}} = \frac{1}{|\hat{N}|} \sum \| \hat{\mathbf{N}} - \mathbf{N} \|_1 $$
     [S1]
   - **Margin-based Loss (Stable Normal):** Use a hinge-like loss with margin $m$ to handle inconsistencies across views:
     $$ L_n = \sum \max(\| \mathbf{N} - \hat{\mathbf{N}} \| - m, 0) $$
     [S2]
   - **Depth and Normal Joint Loss (SRDF):** If depth is also involved, align rendered depth $D$ with monocular depth $\bar{D}$ using affine parameters $w, q$, and enforce normal consistency:
     $$ \mathcal{L}_d = \frac{1}{N_s} \sum_{\mathbf{r} \in \mathcal{S}^2} \| (w D + q) - \bar{D} \|_2 $$
     $$ \mathcal{L}_n = \frac{1}{N_s} \sum_{\mathbf{r} \in \mathcal{S}^2} \left( \| \mathbf{N} - \bar{\mathbf{N}} \|_1 + \| 1 - \mathbf{N}^T \bar{\mathbf{N}} \|_1 \right) $$
     [S3]
     Here, $\bar{D}, \bar{\mathbf{N}}$ are ground truth estimates from Omnidata, and $T_i, \alpha_i$ are transmittance/opacity weights.

**Parameters:**
- $m$: Small margin for Stable Normal loss. Value not given in sources. [S2]
- $\lambda_n$: Weight of the monocular normal loss in the total objective. Value not given in sources. [S2]
- $w, q$: Scale and shift parameters for depth alignment, optimized per minibatch. [S3]

**Edge Cases and Pitfalls:**
- **Depth Ambiguity:** Monocular depth prediction suffers from scale and shift ambiguities. This is resolved by optimizing affine parameters $w$ and $q$ per minibatch, following MonoSDF [S3].
- **View Inconsistency:** Monocular models may produce inconsistent normals across different views. The margin $m$ in the Stable Normal loss helps mitigate this [S2].
- **Noise in Gradients:** Using gradients of rendered depths for supervision results in artifacts, especially in complex scenes. Monocular cues provide smoother estimates [S1].

## Subtopics

### [Geometric Prior Guided Geometry Learning](16865-geometric-prior-guided-geometry-learning.md)
A two-stage method for high-fidelity 3D reconstruction that leverages geometric priors during both pre-training and volume rendering optimization. It addresses the lack of ground-truth surface normals in monocular settings by using a pretrained monocular normal estimator to supervise the optimization stage, while using ground-truth mesh normals during pre-training. It builds on [Monocular Normal Regularization](13425-monocular-normal-regularization.md) by extending the use of external normal predictions to the entire optimization pipeline, rather than just as a regularizer on gradients.

## Operating characteristics
- **Time Complexity:** Depends on the monocular network inference time. Adding a pretrained network inference step increases per-step cost compared to gradient-based methods.
- **Memory Complexity:** Requires storing the pretrained monocular model weights and intermediate feature maps if used in a differentiable pipeline, or just the output normal maps if treated as static supervision.
- **Strengths:** Provides smoother normal estimates than gradient-based methods, reducing artifacts in complex scenes [S1]. Handles geometric priors effectively in monocular settings.
- **Limitations:** Relies on the quality and consistency of the pretrained monocular model. May introduce view-inconsistent biases if not handled (e.g., via margin $m$) [S2].
- **When to use:** When gradient-based normals are too noisy or when ground-truth normals are unavailable.
- **When not to use:** When computational resources are extremely limited and monocular inference is prohibitive, or when ground-truth normals are available.

## Background
Originated from the need to improve 3D reconstruction quality in monocular settings where ground-truth normals are unavailable. Previous works used gradients of rendered depths, which are noisy [S1, S2]. The method leverages foundational monocular models like Omnidata [S1, S3] and Stable Normal [S2] to provide smooth geometric cues. It is used in Gaussian Splatting [S1, S2] and SDF/SRDF-based volume rendering [S3] to regularize geometry.

## Sources
[S1] DN-Splatter: Depth and Normal Priors for Gaussian Splatting and Meshing, https://doi.org/10.1109/wacv61041.2025.00241, fetched 2026-10-26
[S2] OMeGa: Joint Optimization of Explicit Meshes and Gaussian Splats for Robust Scene-Level Surface Reconstruction, https://doi.org/10.1109/WACV61042.2026.00427, fetched 2026-10-30
[S3] Ray-Distance Volume Rendering for Neural Scene Reconstruction, https://doi.org/10.1007/978-3-031-72630-9_22, fetched 2026-10-26
