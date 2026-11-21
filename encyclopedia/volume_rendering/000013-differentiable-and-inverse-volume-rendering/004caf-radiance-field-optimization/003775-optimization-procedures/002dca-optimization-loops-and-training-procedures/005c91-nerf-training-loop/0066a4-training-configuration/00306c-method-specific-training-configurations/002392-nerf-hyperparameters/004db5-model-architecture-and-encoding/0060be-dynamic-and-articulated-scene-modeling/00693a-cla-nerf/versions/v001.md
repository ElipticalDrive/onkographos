# CLA-NeRF

Also called: Category-Level Articulated Neural Radiance Field

## Overview
CLA-NeRF is a differentiable representation for articulated objects that explicitly models part and joint attributes, disentangling camera pose, part pose, part segmentation, and joint attributes to allow independent control during rendering [S1]. It solves the problem of rendering deformable articulated objects at unseen poses and estimating their articulated pose from RGB images, without requiring CAD models or depth data [S1]. Unlike standard NeRF formulations which model camera poses but ignore object articulation, CLA-NeRF predicts a part segmentation field alongside density and color, enabling the inference of joint attributes via line-fitting on the segmentation field [S1]. It differs from sibling methods like NARF by focusing on general articulated objects without assuming known joint attributes (e.g., bone length) at test time, and by using part segmentation labels for supervision rather than explicit joint attributes [S1].

## How it works
**Inputs and Outputs:**
*   **Training Inputs:** A set of RGB images with ground truth camera poses and part segmentation labels for a specific object category [S1].
*   **Inference Inputs:** A few RGB views (few-shot) of an unseen 3D object instance within the known category, along with an articulated pose input [S1].
*   **Outputs:** Rendered RGB images, part segmentation maps, and estimated articulated pose (via inverse rendering) [S1].

**Data Structures:**
*   **MLP Backbone ($F_\Theta$):** A multilayer perceptron parameterized by weights $\Theta$ [S1].
*   **Positional Encoding ($\gamma$):** High-frequency positional encoding applied to 3D positions [S1].
*   **Segmentation Logits ($\mathbf{s}$):** A vector of dimension $P + 1$, where $P$ is the number of parts and the extra dimension represents the background [S1].

**Numbered Steps:**

1.  **Feature Retrieval and Conditioning:** Retrieve features from input RGB images to condition the NeRF model [S1].
2.  **Deformation Matrix Calculation:** Calculate a deformation matrix based on the input articulated pose [S1].
3.  **Ray Deformation:** Deform the sampled rays along the camera ray using the calculated deformation matrix [S1].
4.  **Field Prediction:** For each 3D location $\mathbf{x}$ and viewing direction $\mathbf{d}$, predict volumetric density $\sigma$, RGB color $\mathbf{c}$, and part segmentation logits $\mathbf{s}$ using the MLP backbone:
    $$ (\sigma, \mathbf{c}, \mathbf{s}) = F_\Theta(\gamma(\mathbf{x}), \mathbf{d}) $$
    where $\mathbf{s}$ has $P + 1$ dimensions [S1].
5.  **Articulation-Aware Volume Rendering:** Render the deformed visual image using the learned framework via volume rendering [S1]. The rendered color $\hat{\mathbf{C}}(\mathbf{r})$ for a ray $\mathbf{r}$ is computed as:
    $$ \hat{\mathbf{C}}(\mathbf{r}) = \sum_{k=1}^{K} T_k \alpha_k \mathbf{c}_k $$
    where $\alpha_k = 1 - \exp(-\sigma_k(t_{k+1} - t_k))$ and $T_k = \exp(-\sum_{k'<k} \sigma_{k'}(t_{k'+1} - t_{k'}))$ [S1].
6.  **Segmentation Rendering:** Predict the part segmentation map $\hat{\mathbf{S}}(\mathbf{r})$ following the volume rendering procedure:
    $$ \hat{\mathbf{S}}(\mathbf{r}) = \sum_{k=1}^{K} T_k \alpha_k \mathbf{s}_k $$
    where $\mathbf{s}_k$ is the predicted part segmentation of the sampled point $\mathbf{r}(t_k)$ [S1].
7.  **Joint Attribute Inference:** Infer joint attributes by performing line-fitting on the predicted part segmentation field [S1].
8.  **Inverse Rendering (Pose Estimation):** Estimate the articulated pose via inverse rendering by minimizing the residual between rendered and observed pixels [S1].

**Formulas:**
*   **Segmentation Rendering:**
    $$ \hat{\mathbf{S}}(\mathbf{r}) = \sum_{k=1}^{K} T_k \alpha_k \mathbf{s}_k $$
    $$ T_k = \exp\left(-\sum_{k'<k} \sigma_{k'}(t_{k'+1} - t_{k'})\right) $$
    where $\mathbf{s}_k$ is the predicted part segmentation of sampled point $\mathbf{r}(t_k)$ [S1].
*   **Color Loss ($\mathcal{L}_{color}$):**
    $$ \mathcal{L}_{color} = \sum_{\mathbf{r} \in \mathbf{R}} \left( \|\hat{\mathbf{C}}_c(\mathbf{r}) - \mathbf{C}(\mathbf{r})\|_2^2 + \|\hat{\mathbf{C}}_f(\mathbf{r}) - \mathbf{C}(\mathbf{r})\|_2^2 \right) $$
    where $\mathbf{R}$ is the set of rays, and the subscripts $c$ and $f$ likely refer to different components or views as per the source notation [S1].
*   **Segmentation Loss ($\mathcal{L}_{seg}$):**
    $$ \mathcal{L}_{seg} = -\sum_{\mathbf{r} \in \mathbf{R}} \sum_{i=1}^{P+1} \left( p_i(\mathbf{r}) \log \hat{p}_i(\mathbf{r}) + p_i(\mathbf{r}) \log \hat{p}_i(\mathbf{r}) \right) $$
    where $\hat{p}_i(\mathbf{r}) = \frac{\exp(\hat{\mathbf{S}}_i(\mathbf{r}))}{\sum_{j} \exp(\hat{\mathbf{S}}_j(\mathbf{r}))}$ is the softmax probability for part $i$ [S1]. Note: The source formula for $\mathcal{L}_{seg}$ appears to have a typo or specific notation where $p_i(\mathbf{r})$ is used for both ground truth and predicted terms in the log expression; standard cross-entropy would use $p_i^{gt} \log \hat{p}_i$. The source text is reproduced as cited [S1].

**Parameters:**
*   $P$: The number of parts in the articulated object, determining the dimensionality of the segmentation logits ($P + 1$ including background) [S1].

**Pitfalls:**
*   Standard NeRF formulations explicitly model camera poses but do not consider object articulation, making them unable to render deformable articulated objects at different poses [S1].
*   The method assumes revolute joints causing 1D rotational motion (e.g., eyeglasses) in the experiments [S1].

## Operating characteristics
Not given in the sources.

## Background
CLA-NeRF was proposed by Wei-Cheng Tseng, Hung-Ju Liao, Lin Yen-Chen, and Min Sun [S1]. It was published in the IEEE International Conference on Robotics and Automation (ICRA) in 2022 [S1]. The work addresses the challenge of modeling general articulated objects (composed of rigid parts connected by joints) without relying on instance-level CAD models or depth data, enabling few-shot inference on unseen 3D object instances within a known category [S1]. It is the first method to model general articulated objects with neural radiance fields, allowing for view synthesis, part segmentation, and articulated pose estimation [S1].

## Sources
[S1] CLA-NeRF: Category-Level Articulated Neural Radiance Field, https://doi.org/10.1109/icra46639.2022.9812272, fetched 2026-10-29
