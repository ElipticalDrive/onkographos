# Portrait-NeRF

Also called: Portrait Neural Radiance Fields (Portrait-NeRF), Single-image NeRF for portraits

## Overview
Portrait-NeRF is a method for estimating Neural Radiance Fields from a single headshot portrait image [S1, unofficial]. It addresses the ill-posed nature of training an MLP from a single image by pretraining the MLP with a meta-learning framework on a light stage dataset and finetuning it on the input image [S1, unofficial]. Unlike standard NeRF which requires multiple views, or 3D model-based methods that exclude upper head and torso, Portrait-NeRF reconstructs the full portrait including hairs and accessories using a canonical coordinate warping strategy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A single frontal headshot portrait image (Support Set $D_s$) and a pre-trained meta-learning model.
*   **Output:** A subject-specific MLP parameter set $\theta_{s^*}$ capable of rendering novel views of the subject.

**Data Structures:**
*   **MLP $f_\theta$:** A coordinate-based Multilayer Perceptron mapping inputs to color and density.
    *   **Input:** Warped 3D position $\tilde{x}' = sR\tilde{x} + t$ and viewing direction $\tilde{d}$.
    *   **Output:** Color $(RGB)$ and volumetric density/occupancy $(\rho)$.
*   **3D Morphable Model (3DMM):** Provides a rigid transform $(s, R, t)$ to map world coordinates $\tilde{x}$ to canonical face coordinates. $s$ is scale, $R$ is rotation, $t$ is translation.
*   **Light Stage Dataset:** Contains multiple subjects. Each subject has a Support Set $D_s$ (frontal view) and a Query Set $D_q$ (24 other views in a 5x5 grid).

**Numbered Steps:**

1.  **Meta-Learning Pretraining:**
    *   Iterate through $K$ subjects in the light stage dataset, indexed by $m = \{0, ..., K-1\}$.
    *   For each subject $m$, define a task $T_m$.
    *   Initialize a temporary model $\theta_{p,m}$ (starting from the global pretrained weights).
    *   Compute the gradient of the loss on the Support Set $D_s$ (frontal view) with respect to $\theta_{p,m}$.
    *   Update $\theta_{p,m}$ using one or more gradient descent steps to obtain $\theta_{m}$.
    *   Compute the loss on the Query Set $D_q$ (remaining 24 views) using the updated parameters $\theta_{m}$.
    *   Update the global pretrained weights $\theta_p$ using the gradient of the Query Set loss with respect to $\theta_p$ (meta-gradient).
    *   Repeat for all subjects to obtain the final pretrained parameters $\theta_{p^*}$ [S1, unofficial].

2.  **Test Time Finetuning (Subject Adaptation):**
    *   Given a new subject $s$ and their single frontal image $D_s$.
    *   Initialize the NeRF MLP parameters $\theta$ with the pretrained weights $\theta_{p^*}$.
    *   Finetune $\theta$ by minimizing the reconstruction loss between the rendered image from the frontal camera pose and the input image $D_s$.
    *   Obtain the subject-specific parameters $\theta_{s^*}$ [S1, unofficial].

3.  **View Synthesis:**
    *   For a target camera pose, cast rays through the scene.
    *   Sample points $\tilde{x}$ along the rays.
    *   Warp the world coordinates $\tilde{x}$ to canonical coordinates using the subject-specific rigid transform $(s, R, t)$ derived from the 3DMM: $\tilde{x}' = sR\tilde{x} + t$.
    *   Query the finetuned MLP $f_{\theta_{s^*}}$ with $(\tilde{x}', \tilde{d})$ to get density $\sigma$ and color $c$.
    *   Perform standard volume rendering to compute the pixel color [S1, unofficial].

**Formulas:**
*   **Volume Rendering:**
    $$ \hat{C}(r) = \sum_{i=1}^{N} T_i \alpha_i c_i = \sum_{i=1}^{N} T_i (1 - e^{-\sigma_i \delta_i}) c_i $$
    where $T_i = \prod_{j=1}^{i-1} e^{-\sigma_j \delta_j}$, $\sigma_i$ is density, $c_i$ is color, and $\delta_i$ is interval length [S1, unofficial].
*   **Canonical Warping:**
    $$ \tilde{x}' = sR\tilde{x} + t $$
    where $\tilde{x}$ is the world coordinate, and $(s, R, t)$ is the rigid transform from the 3DMM [S1, unofficial].
*   **Meta-Learning Update:**
    Let $\mathcal{L}_s(\theta)$ be the loss on the support set and $\mathcal{L}_q(\theta')$ be the loss on the query set after one update step $\theta' = \theta - \alpha \nabla_\theta \mathcal{L}_s(\theta)$. The meta-update is:
    $$ \theta_p \leftarrow \theta_p - \beta \nabla_{\theta_p} \mathcal{L}_q(\theta - \alpha \nabla_\theta \mathcal{L}_s(\theta)) $$
    where $\alpha$ is the inner learning rate and $\beta$ is the outer learning rate [S1, unofficial].

**Parameters:**
*   **Light Stage Camera Sampling:** 5-by-5 training views per subject.
*   **Field of View:** Vertical 25 degrees, Horizontal 15 degrees.
*   **Camera Distance:** Fixed distance between camera and subject.
*   **Support Set $D_s$:** Center view (frontal).
*   **Query Set $D_q$:** Remaining 24 views [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Ill-posedness:** Training an MLP from a single image from scratch is fundamentally ill-posed because there are infinite solutions where renderings match the input image. This is mitigated by the meta-learning pretraining which provides a strong prior [S1, unofficial].
*   **3DMM Accuracy:** The quality of the canonical warping depends on the accuracy of the 3DMM fit. Poor fit may lead to distorted geometry in unobserved regions.

## Operating characteristics
*   **Time Complexity:** Pretraining is performed offline on a dataset. Test-time finetuning requires only a few gradient steps on a single image, making it significantly faster than training a NeRF from scratch for a new subject.
*   **Memory Complexity:** Depends on the MLP size and the number of samples per ray. Similar to standard NeRF.
*   **Performance:** Quantitative evaluation on controlled captures shows a +5dB PSNR improvement against the second-best approach in ground truth validation [S1, unofficial].
*   **Strengths:** Handles shape variations via canonical warping; reconstructs full head including hair and torso; works with a single image.
*   **Limitations:** Requires a 3DMM for warping; performance depends on the quality of the frontal input image and 3DMM fit; may struggle with extreme poses not covered by the canonical space.

## Background
Portrait-NeRF was proposed to address the impracticality of standard NeRF for portrait view synthesis, which requires multiple images of static subjects [S1, unofficial]. It builds on the concept of Neural Radiance Fields (NeRF) [S1, unofficial] and applies gradient-based meta-learning techniques [S1, unofficial] to enable rapid adaptation to unseen subjects. The method leverages domain-specific knowledge about face shape via 3D Morphable Models (3DMM) to define a canonical coordinate system [S1, unofficial].

## Sources
[S1] Portrait Neural Radiance Fields from a Single Image, https://doi.org/10.48550/arxiv.2012.05903, fetched 2026-10-26 (unofficial)
