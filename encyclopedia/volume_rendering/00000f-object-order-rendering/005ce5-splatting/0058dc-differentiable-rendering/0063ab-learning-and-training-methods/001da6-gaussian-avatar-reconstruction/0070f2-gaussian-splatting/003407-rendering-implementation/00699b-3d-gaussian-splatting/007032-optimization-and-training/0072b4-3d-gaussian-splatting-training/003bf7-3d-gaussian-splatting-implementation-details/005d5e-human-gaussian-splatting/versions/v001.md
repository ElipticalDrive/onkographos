# Human Gaussian Splatting
Also called: HUGS, Human Gaussian Splats

## Overview
Human Gaussian Splats (HUGS) is a neural rendering framework that represents an animatable human avatar and a static scene using 3D Gaussians, trained on 50–100 frames of monocular video to enable novel view and novel pose synthesis at 60 FPS [S1]. It sets itself apart from implicit neural field methods by allowing test-time rendering without evaluating triplane/MLPs for new poses, and from static 3DGS by disentangling the human and scene to allow human transfer to other scenes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A monocular video sequence of 50–100 frames containing a moving human and a static scene.
*   **Output:** A set of 3D Gaussians representing the static scene and a set of 3D Gaussians representing the human. The human Gaussians are parameterized by mean locations in a canonical space, features from a triplane, and properties predicted by MLPs. The system outputs rendered images for novel views and novel poses.

**Data Structures:**
*   **Scene Gaussians:** Standard 3DGS representation (center $\mu$, opacity, color, scale, rotation) initialized from COLMAP point cloud.
*   **Human Gaussians:**
    *   **Canonical Center ($\mu_i$):** 3D location in the canonical (T-pose) space.
    *   **Feature Vector ($f_i$):** Concatenation of feature vectors $f_{xi}, f_{yi}, f_{zi}$ retrieved from a triplane $F \in \mathbb{R}^{3 \times h \times w \times d}$ at location $\mu_i$.
    *   **Appearance Properties:** RGB color, opacity $o_i$.
    *   **Geometry Properties:** Offset $\Delta \mu_i$, Rotation matrix $R_i$ (parameterized by first two columns), Scale $S_i$ (three axes).
    *   **Deformation Properties:** LBS weights $W_i \in \mathbb{R}^{n_k}$, where $n_k$ is the number of joints.
    *   **Base Scale ($s_i$):** A scalar multiplier for the MLP-predicted scale.

**Numbered Steps:**

1.  **Initialization:**
    1.  Estimate SMPL pose parameters $\theta_t$ for each image $t$ and shared body shape parameters $\beta$ using a pretrained SMPL regressor [S1].
    2.  Initialize Human Gaussian centers $\mu$ at the vertices of the canonical-posed SMPL mesh (T-pose). The number of human Gaussians equals the number of SMPL vertices ($n_v = 110,210$) [S1].
    3.  Initialize Scene Gaussians from the COLMAP structure-from-motion point cloud [S1].
    4.  Pretrain the feature triplane and three MLPs (Appearance $D_A$, Geometry $D_G$, Deformation $D_D$) for 5,000 iterations to output:
        *   RGB: $[0.5, 0.5, 0.5]$
        *   Opacity: $0.1$
        *   Offset $\Delta \mu$: $0$
        *   Rotation $R$: Aligning the z-axis with the corresponding SMPL vertex normal.
        *   Scale $S$: Average incoming edge lengths of the SMPL mesh.
        *   LBS Weights $W$: Weights from the SMPL model [S1].
    5.  Initialize base scale $s_i = 1$ for each human Gaussian [S1].

2.  **Optimization Loop (12,000 iterations):**
    1.  **Forward Pass (Rendering):**
        1.  Given a joint configuration $G$ (pose), for each Human Gaussian $i$:
            *   Interpolate the triplane at $\mu_i$ to get $f_{xi}, f_{yi}, f_{zi}$.
            *   Concatenate to form $f_i$.
            *   Input $f_i$ to $D_A$ to output RGB and opacity.
            *   Input $f_i$ to $D_G$ to output $\Delta \mu_i, R_i, S_i$.
            *   Input $f_i$ to $D_D$ to output LBS weights $W_i$.
        2.  **Deformation:** Transform Human Gaussians using Linear Blend Skinning (LBS) with $W_i$ and joint transformations $G_k \in SE(3)$ [S1].
        3.  **Splatting:** Combine transformed Human Gaussians and Scene Gaussians. Splat onto the image plane using alpha blending [S1].
    2.  **Loss Calculation:** Compute total loss $\mathcal{L}$ (see Formulas) [S1].
    3.  **Backpropagation:** Update $\mu$, triplane, and MLP parameters using Adam optimizer ($lr=10^{-3}$, cosine schedule) [S1].
    4.  **Density Control (Every 600 iterations):**
        1.  **Clone:** Repeat existing centers in $\mu$.
        2.  **Split:** Randomly sample new centers within existing Gaussians' shapes ($R, S$).
        3.  **Prune:** Remove Gaussians based on loss gradient and opacity.
        4.  **Scale Adjustment:** When splitting, divide the base scale $s$ by 1.6. The actual scale is $s \times S_{MLP}$ [S1].

3.  **Test-Time Rendering:**
    1.  Construct the 3D Gaussians explicitly from the optimized parameters.
    2.  For a new pose $G$, apply LBS to the Human Gaussians using the learned $W_i$ and $G$.
    3.  Splat and render. No MLP evaluation is required for pose changes [S1].

**Formulas:**

*   **SMPL Posed Vertex (for initialization/regularization only):**
    $$v_i = \left( \sum_{k=1}^{n_k} W_{k,i} G_k \right) p_i$$
    Where $v_i$ is the posed vertex, $W_{k,i}$ is the LBS weight for joint $k$ and vertex $i$, $G_k$ is the joint transformation in $SE(3)$, and $p_i$ is the resting vertex location [S1].

*   **Total Loss:**
    $$\mathcal{L} = \underbrace{\lambda_1 \mathcal{L}_1 + \lambda_2 \mathcal{L}_{\text{ssim}} + \lambda_3 \mathcal{L}_{\text{vgg}}}_{\text{scene + human}} + \underbrace{\lambda_1 \mathcal{L}^h_1 + \lambda_2 \mathcal{L}^h_{\text{ssim}} + \lambda_3 \mathcal{L}^h_{\text{vgg}}}_{\text{human}} + \lambda_4 \mathcal{L}_{\text{LBS}}$$
    Where $\mathcal{L}_1, \mathcal{L}_{\text{ssim}}, \mathcal{L}_{\text{vgg}}$ are L1, SSIM, and Perceptual losses. Human losses are computed on human-only renderings against ground truth human regions (from segmentation) [S1].

*   **LBS Regularization Loss:**
    $$\mathcal{L}_{\text{LBS}} = \| W - \hat{W} \|_2^F$$
    Where $\hat{W}$ is the distance-weighted average of LBS weights of the $k=6$ nearest SMPL vertices [S1].

**Parameters:**
*   $\lambda_1 = 0.8$
*   $\lambda_2 = 0.2$
*   $\lambda_3 = 1.0$
*   $\lambda_4 = 1000$
*   Optimizer: Adam, learning rate $10^{-3}$, cosine schedule [S1].
*   SMPL Mesh: Upsampled version with $n_v = 110,210$ vertices and $n_t = 220,416$ faces. Used only for initialization and regularization [S1].
*   Split Scale Divisor: 1.6 [S1].

**Pitfalls:**
*   **Artifacts:** Holes or pop-outs can occur during articulation if Gaussians are not coordinated to retain surface integrity. This is mitigated by learning LBS weights [S1].
*   **Pose Errors:** Off-the-shelf SMPL regressors may produce inaccurate pose parameters $\theta$. This is mitigated by adjusting $\theta$ per image during optimization [S1].

## Operating characteristics
*   **Training Time:** Approximately 30 minutes on a GeForce 3090Ti GPU [S1].
*   **Rendering Speed:** 60 FPS at HD resolution [S1].
*   **Memory:** Final human representation consists of ~200K Gaussians on average [S1].
*   **Strengths:** ~100x faster to train than previous works (e.g., NeuMan, Vid2Avatar); state-of-the-art rendering quality; disentangled representation allows human transfer to other scenes; test-time animation without MLP evaluation [S1].
*   **Limitations:** Requires monocular video input; relies on SMPL prior for initialization (though Gaussians can deviate); potential artifacts if LBS coordination fails [S1].
*   **When to use:** When animatable human avatars from monocular video are needed with high rendering speed and novel pose synthesis.
*   **When not to use:** When multi-view capture is available and higher geometric fidelity without parametric priors is required, or when real-time training is not feasible.

## Background
HUGS was introduced by Jen-Hao Rick Chang et al. in 2023 (published at CVPR 2024) [S1]. It builds on 3D Gaussian Splatting [15] and the SMPL body model [4] to address the challenge of animating humans in neural rendering. Previous methods relied on implicit neural fields or multi-view capture setups. HUGS enables novel-pose synthesis and novel-view synthesis from short, in-the-wild monocular videos [S1].

## Sources
[S1] HUGS: Human Gaussian Splats, https://doi.org/10.1109/cvpr52733.2024.00055, fetched 2026-10-26
