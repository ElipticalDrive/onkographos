# Occlusion-Eliminating NeRF Variant

Also called: Occlusion-Eliminating NeRF, Zhu et al. NeRF

## Overview
This method addresses the degradation of novel view synthesis caused by foreground occlusions (e.g., people, vehicles) that obscure the target scene during training data capture [S1]. It solves this by decoupling the scene representation into two distinct neural networks: a scene MLP for the target geometry and a background MLP for the occluding elements [S1]. Unlike standard NeRF which treats all density as part of a single volume, this variant explicitly models the occluder to restore an occlusion-free background, distinguishing it from general regularization methods like RegNeRF [8582-regnerf.md] which do not explicitly separate occluders [S1].

## How it works
The method employs a dual-MLP architecture to jointly optimize scene geometry and camera parameters while suppressing occlusion artifacts.

**Inputs and Outputs:**
*   **Inputs:** A set of input images $I$ containing occlusions; two high-quality reference views $I_1^r$ and $I_2^r$ used for supervision [S1].
*   **Outputs:** A reconstructed scene representation free of foreground occlusions, and enhanced views $\hat{I}$ [S1].

**Data Structures:**
1.  **Scene MLP:** A multi-layer perceptron parameterized by $\theta_s$ that predicts density $\sigma$ and color $\mathbf{c}$ for the target scene geometry [S1].
2.  **Background MLP:** A multi-layer perceptron parameterized by $\theta_b$ that predicts density and color for the occluding background/foreground elements [S1].
3.  **Camera Parameters:** Learnable extrinsic and intrinsic parameters optimized jointly with the MLPs [S1].

**Loss Functions:**
The total loss combines a photometric term with specific regularization terms derived from the sources [S1]:

1.  **Photometric Loss ($\mathcal{L}_{photo}$):**
    $$ \mathcal{L}_{photo} = || \hat{I} - I_{gt} ||_2^2 $$
    Where $\hat{I}$ is the rendered view and $I_{gt}$ is the ground truth input image [S1]. Note: The source notation $I$ in the formula $L = I - I_{gt}$ refers to the rendered view $\hat{I}$ in the context of the reconstruction goal [S1].

2.  **Cost Volume Loss ($\mathcal{L}_{cost}$):**
    A strategy designed to guide the joint optimization of the Scene MLP and camera parameters [S1]. The exact mathematical formulation is not provided in the sources [S1].

3.  **Depth Bidirectional Inconsistency Prior ($\mathcal{L}_{depth}$):**
    Applied within the Background MLP to enforce depth distance supervision, ensuring the background model learns consistent depth structures to restore occlusion-free backgrounds [S1]. The exact mathematical formulation is not provided in the sources [S1].

**Algorithm Steps:**
1.  Initialize the Scene MLP and Background MLP with random weights [S1].
2.  For each training iteration:
    a. Render views $\hat{I}$ using the current Scene and Background MLPs and camera parameters [S1].
    b. Compute the photometric loss $\mathcal{L}_{photo}$ between $\hat{I}$ and the input image $I$ [S1].
    c. Compute the Cost Volume Loss $\mathcal{L}_{cost}$ to update the Scene MLP and camera parameters jointly [S1].
    d. Compute the Depth Bidirectional Inconsistency Loss $\mathcal{L}_{depth}$ using depth distance supervision to update the Background MLP [S1].
    e. Update all parameters via gradient descent [S1].

**Pitfalls:**
*   **Transparent Glass Occlusions:** When the occluding object is a light-loss transparent glass body, the method fails to generalize due to noise pollution from reflection artifacts [S1].

## Subtopics

### NeRFReN
A model proposed by Guo et al. [S1] that utilizes two types of geometric priors to enforce depth smoothness and bidirectional depth consistency losses [S1]. It is designed to separate reflection artifacts from physical entities, addressing the transparency issue found in the base Occlusion-Eliminating NeRF [S1]. It is chosen when occlusions involve reflective or transparent surfaces like glass [S1]. However, it cannot distinguish separation when physical entities and artifacts are complexly interlaced [S1].

## Operating characteristics
*   **Strengths:** Explicitly restores occlusion-free backgrounds by separating occluders from the target scene [S1].
*   **Limitations:** Generalization fails when occlusions are transparent glass bodies with reflection artifacts [S1]. It cannot handle complex interlacing of entities and artifacts [S1].
*   **Performance:** Specific timing or memory metrics are not provided in the sources [S1].

## Background
The method was proposed by Zhu et al. in 2023 [S1]. It was developed to address the common problem in multi-view reconstruction where foreground objects obstruct the view of the target scene, degrading the NeRF synthesis quality [S1].

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
