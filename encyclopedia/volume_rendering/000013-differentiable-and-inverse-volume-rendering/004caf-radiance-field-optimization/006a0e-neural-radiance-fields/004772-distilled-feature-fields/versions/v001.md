# Distilled Feature Fields

Also called: Neural Perceptual Fields

## Overview
Distilled Feature Fields (DFF) extend Neural Radiance Fields by adding a 3D neural feature field that maps every 3D coordinate to a semantic feature descriptor, enabling query-based local editing of NeRF scenes by semantically decomposing 3D space without re-training [S1]. Unlike methods that optimize NeRF parameters to match text prompts (which may affect unintended regions) or rely on costly instance segmentation annotations, DFF enables efficient, open-set decomposition and accurate selective editing of specific scene regions via decomposition [S1]. It builds on [Neural Radiance Fields](4972-neural-radiance-fields.md) by distilling knowledge from off-the-shelf 2D image feature extractors (teachers) into the 3D field via volume rendering [S1].

## How it works
**Inputs:**
*   A set of input images with known camera poses.
*   A pre-trained 2D teacher network (e.g., LSeg for text/image queries or DINO for self-supervised features) [S1].
*   User query: text string, image patch, or point-and-click selection [S1].

**Outputs:**
*   A trained NeRF model (density $\sigma$ and color $c$) [S1].
*   A trained 3D Feature Field $f(\mathbf{x})$ [S1].
*   Rendered images and decomposed/edited views based on user queries [S1].

**Data Structures:**
*   **NeRF MLP:** Takes 3D coordinate $\mathbf{x}$ and view direction $\mathbf{d}$, outputs density $\sigma(\mathbf{x})$ and color $c(\mathbf{x}, \mathbf{d})$.
*   **Feature Field MLP:** Takes 3D coordinate $\mathbf{x}$, outputs a semantic feature vector $f(\mathbf{x}) \in \mathbb{R}^D$ [S1].
*   **Teacher Network:** A frozen 2D encoder (LSeg or DINO) that takes an image $I$ and returns feature maps $f_{img}(I, \mathbf{r})$ [S1].
*   **Query Encoder:** A frozen encoder (part of LSeg or DINO) that maps a text label $l$ or image patch to a feature vector $f_q(l)$ [S1].

**Training Procedure:**
The method trains the NeRF and the Feature Field in parallel using a teacher-student distillation approach [S1].

1.  **Ray Sampling:** For each pixel, sample $K$ points $\mathbf{x}_k$ along the camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ within bounds $[t_{near}, t_{far}]$ [S1].
2.  **NeRF Evaluation:** Query the NeRF MLP for density $\sigma(\mathbf{x}_k)$ and color $c(\mathbf{x}_k, \mathbf{d})$ [S1].
3.  **Feature Field Evaluation:** Query the Feature Field MLP for the 3D feature vector $f(\mathbf{x}_k)$ [S1].
4.  **Volume Rendering (NeRF):** Compute the rendered color $\hat{C}(\mathbf{r})$ using the standard emission-absorption model [S1]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{k=1}^{K} \hat{T}(t_k) \alpha(\sigma(\mathbf{x}_k)\delta_k) c(\mathbf{x}_k, \mathbf{d})
    $$
    where $\hat{T}(t_k) = \exp \left( -\sum_{k'=1}^{k-1} \sigma(\mathbf{x}_{k'})\delta_{k'} \right)$, $\alpha(x) = 1 - \exp(-x)$, and $\delta_k = t_{k+1} - t_k$ [S1].
5.  **Volume Rendering (Features):** Compute the rendered feature vector $\hat{F}(\mathbf{r})$ by volume rendering the 3D features $f(\mathbf{x}_k)$ using the same transmittance weights as the color [S1]:
    $$
    \hat{F}(\mathbf{r}) = \sum_{k=1}^{K} \hat{T}(t_k) \alpha(\sigma(\mathbf{x}_k)\delta_k) f(\mathbf{x}_k)
    $$
    [Note: The source describes rendering features "in the same way as NeRF does" [S1]. The exact formula for $\hat{F}$ is implied by the volume rendering equation applied to $f(\mathbf{x})$].
6.  **Teacher Projection:** Render the input image $I$ from the current camera pose and pass it through the frozen teacher network to obtain the teacher's feature output $f_{img}(I, \mathbf{r})$ [S1].
7.  **Loss Computation:**
    *   **Photometric Loss ($L_p$):** Mean squared error between rendered color $\hat{C}(\mathbf{r})$ and ground-truth pixel color [S1].
    *   **Feature Loss ($L_f$):** Mean squared error between rendered features $\hat{F}(\mathbf{r})$ and teacher features $f_{img}(I, \mathbf{r})$ [S1].
    *   **Total Loss:** $L = L_p + \lambda L_f$, where $\lambda$ is a weighting factor [S1].
8.  **Optimization:** Minimize $L$ with respect to the parameters of the NeRF MLP and the Feature Field MLP using gradient descent [S1].

**Query-based Decomposition and Editing:**
At test time, the trained DFF enables zero-shot segmentation and editing [S1].

1.  **Query Encoding:** Encode the user query (text $l$ or image patch) into a feature vector $f_q(l)$ using the frozen query encoder [S1].
2.  **Segmentation Probability:** For any 3D point $\mathbf{x}$, calculate the probability $p(l|\mathbf{x})$ of belonging to label $l$ using the dot product of the 3D feature $f(\mathbf{x})$ and the query feature $f_q(l)$, followed by a softmax over a set of labels $\mathcal{L}$ [S1]:
    $$
    p(l|\mathbf{x}) = \frac{\exp(f(\mathbf{x})f_q(l)^T)}{\sum_{l' \in \mathcal{L}} \exp(f(\mathbf{x})f_q(l')^T)}
    $$
    [S1]
3.  **Editing via Blending:** To apply a geometric transformation $g$ to a region defined by query $l$, blend the original scene $(\sigma, c)$ with a transformed scene $(\sigma_2, c_2)$ [S1].
    *   Define the original scene as Scene 1: $\sigma_1 = \sigma$, $c_1(\mathbf{x}, \mathbf{d}) = c(\mathbf{x}, \mathbf{d})$ [S1].
    *   Define the transformed scene as Scene 2: $\sigma_2(\mathbf{x}) = \sigma(g^{-1}(\mathbf{x}))$, $c_2(\mathbf{x}, \mathbf{d}) = c(g^{-1}(\mathbf{x}), g^{-1}(\mathbf{d}))$ [S1].
    *   Compute the blend ratio $\rho_k$ for sample $k$ using the segmentation probability $p(l|\mathbf{x}_k)$ [S1]:
        $$
        \rho_k = \frac{\alpha(\sigma_1(\mathbf{x}_k)\delta_k)}{\alpha(\sigma_1(\mathbf{x}_k)\delta_k) + \alpha(\sigma_2(\mathbf{x}_k)\delta_k)}
        $$
        [S1]
    *   Set the effective opacity and color for the edited render [S1]:
        $$
        \alpha(\sigma_1(\mathbf{x}_k)\delta_k)_{new} = (1 - p(l|\mathbf{x}_k)) \alpha(\sigma(\mathbf{x}_k)\delta_k)
        $$
        $$
        \alpha(\sigma_2(\mathbf{x}_k)\delta_k)_{new} = p(l|g^{-1}(\mathbf{x}_k)) \alpha(\sigma(g^{-1}(\mathbf{x}_k))\delta_k)
        $$
        $$
        c_1(\mathbf{x}_k, \mathbf{d})_{new} = (1 - p(l|\mathbf{x}_k)) c(\mathbf{x}_k, \mathbf{d})
        $$
        $$
        c_2(\mathbf{x}_k, \mathbf{d})_{new} = p(l|g^{-1}(\mathbf{x}_k)) c(g^{-1}(\mathbf{x}_k), g^{-1}(\mathbf{d}))
        $$
        [S1]
    *   Render the final image using the standard volume rendering equation with these modified opacities and colors [S1].

**Parameters:**
*   Teacher Networks: LSeg (CLIP-inspired language-driven semantic segmentation) and DINO (self-supervised network aware of object boundaries) [S1].
*   Loss Weight $\lambda$: Not given in the sources.
*   Number of Samples $K$: Not given in the sources.

**Edge Cases and Pitfalls:**
*   The goal is not state-of-the-art 3D semantic segmentation on discrete point clouds/voxelgrids, but smooth segmentation results on continuous 3D space for editing [S1].
*   Introducing view-dependency to the segmentation (e.g., for referring expressions like "the chair left to the table") is noted as an interesting extension but is not implemented in the base method [S1].

## Subtopics

### Query-based Decomposition and Editing
A technique to perform 3D zero-shot segmentation and editing by calculating the probability of a label $l$ for a point $\mathbf{x}$ using the dot product of the 3D feature $f(\mathbf{x})$ and a query feature $f_q(l)$ [S1]. It uses teacher networks LSeg and DINO for obtaining image and text features [S1]. It enables geometric transformations of specific regions by blending the original and transformed scenes weighted by segmentation probabilities [S1].

## Operating characteristics
*   **Time Complexity:** Training requires simultaneous optimization of two MLPs (NeRF and Feature Field) and forward passes through a large teacher network for every batch, increasing training time compared to standard NeRF [S1]. Rendering time is similar to NeRF, with the addition of querying the Feature Field MLP [S1].
*   **Memory Complexity:** Requires storing parameters for both the NeRF and Feature Field MLPs, plus the frozen teacher network weights [S1].
*   **Strengths:** Enables efficient, open-set decomposition without re-training; allows accurate selective editing of specific scene regions; leverages pre-trained foundation models for zero-shot segmentation [S1].
*   **Limitations:** Relies on the quality of the 2D teacher network; segmentation is limited to the semantic space of the teacher (e.g., LSeg classes); does not support view-dependent segmentation in the base implementation [S1].
*   **When to use:** When interactive, query-based editing of NeRF scenes is required without re-training the scene representation [S1].
*   **When not to use:** When state-of-the-art discrete 3D semantic segmentation is the primary goal, or when view-dependent segmentation is required [S1].

## Background
Distilled Feature Fields were introduced by Sosuke Kobayashi, Eiichi Matsumoto, and Vincent Sitzmann in 'Decomposing NeRF for Editing Via Feature Field Distillation' (arXiv:2205.15585v2, Oct 2022, NeurIPS 2022) [S1]. The work addresses the challenge of editing NeRF scenes, which are not object-centric, by distilling 2D vision and language foundation models (LSeg, DINO) into a 3D feature field [S1].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
