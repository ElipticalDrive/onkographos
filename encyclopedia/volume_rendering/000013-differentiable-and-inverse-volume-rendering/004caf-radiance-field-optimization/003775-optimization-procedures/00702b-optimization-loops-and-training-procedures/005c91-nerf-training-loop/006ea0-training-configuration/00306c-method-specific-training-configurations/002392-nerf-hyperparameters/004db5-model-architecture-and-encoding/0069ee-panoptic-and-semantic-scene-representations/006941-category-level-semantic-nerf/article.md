# Category-Level Semantic NeRF

Also called: CLA-NeRF

## Overview
Category-Level Semantic NeRF (CLA-NeRF) extends the standard NeRF framework to predict part segmentation logits in addition to density and color, enabling category-level generalization for articulated objects [S1]. Unlike sibling methods that may focus on instance-specific reconstruction, CLA-NeRF conditions the radiance field on image features extracted from input views, allowing the model to share knowledge across objects of the same category and render semantic labels for unseen poses [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of input images $I$ with known relative camera poses; part segmentation labels for the objects in the training set.
*   **Outputs:** For a ray $\mathbf{r}$, the model predicts volume density $\sigma$, RGB color $\mathbf{c}$, and part segmentation logits $\mathbf{s}$ with $P+1$ dimensions ($P$ parts plus background) [S1].

**Data Structures**
*   **Image Encoder Output:** A feature map $W = E(I)$ extracted from the input image $I$ using an image encoder $E$ [S1].
*   **Projected Coordinates:** For a sampled 3D point $\mathbf{x}$, the projected coordinate on the input image plane is denoted $\pi(\mathbf{x})$ [S1].
*   **Segmentation Logits:** $\mathbf{s}_k \in \mathbb{R}^{P+1}$ is the raw output for the $k$-th sample along a ray [S1].

**Network Architecture**
The method builds on the shared principles of Panoptic and Semantic Scene Representations [20495-panoptic-and-semantic-scene-representations.md]. The core MLP backbone $F_\Theta$ is augmented to accept image features. For each 3D location $\mathbf{x}$ and viewing direction $\mathbf{d}$, the network predicts density, color, and segmentation:
$$ (\sigma, \mathbf{c}, \mathbf{s}) = F_\Theta(\gamma(\mathbf{x}), \mathbf{d}, W(\pi(\mathbf{x}))) $$
where $\gamma(\cdot)$ is the positional encoding, and $W(\pi(\mathbf{x}))$ is the feature vector sampled from the image feature map $W$ at the projected coordinate $\pi(\mathbf{x})$ [S1].

**Rendering Procedure**
1.  **Ray Sampling:** Generate $K$ samples along ray $\mathbf{r}$ at depths $t_k$ [S1].
2.  **Network Query:** For each sample $k$, query the network to obtain $\sigma_k$, $\mathbf{c}_k$, and $\mathbf{s}_k$ [S1].
3.  **Alpha Computation:** Compute the transmittance $\hat{T}_k$ for sample $k$:
    $$ \hat{T}_k = \exp\left(-\sum_{k' < k} \sigma_{k'} (t_{k'+1} - t_{k'})\right) $$
    [S1].
4.  **Segmentation Rendering:** Compute the rendered segmentation $\hat{\mathbf{S}}(\mathbf{r})$ by volume rendering the logits $\mathbf{s}_k$:
    $$ \hat{\mathbf{S}}(\mathbf{r}) = \sum_{k=1}^{K} \hat{T}_k (1 - \exp(-\sigma_k (t_{k+1} - t_k))) \mathbf{s}_k $$
    [S1].
5.  **Probability Conversion:** Convert logits to probabilities using softmax:
    $$ \hat{p}_i(\mathbf{r}) = \frac{\exp(\hat{S}_i(\mathbf{r}))}{\sum_{j=1}^{P} \exp(\hat{S}_j(\mathbf{r}))} $$
    Note: The denominator sums over $P$ parts, excluding the background class index in the normalization sum as per the source formula structure, though the numerator index $i$ ranges $1 \dots P+1$ [S1].

**Training Losses**
The model is jointly trained using a coarse and fine network (standard NeRF hierarchy).
1.  **Color Loss ($\mathcal{L}_{color}$):** Mean squared error between ground truth color $\mathbf{C}(\mathbf{r})$ and predictions from coarse ($\hat{\mathbf{C}}_c$) and fine ($\hat{\mathbf{C}}_f$) networks [S1]:
    $$ \mathcal{L}_{color} = \sum_{\mathbf{r} \in \mathcal{R}} \frac{||\hat{\mathbf{C}}_c(\mathbf{r}) - \mathbf{C}(\mathbf{r})||^2 + ||\hat{\mathbf{C}}_f(\mathbf{r}) - \mathbf{C}(\mathbf{r})||^2}{2} $$
    [S1].
2.  **Segmentation Loss ($\mathcal{L}_{seg}$):** Multi-class cross-entropy between ground truth part probabilities $p_i(\mathbf{r})$ and predicted probabilities $\hat{p}_i^c(\mathbf{r})$ (coarse) and $\hat{p}_i^f(\mathbf{r})$ (fine) [S1]:
    $$ \mathcal{L}_{seg} = - \sum_{\mathbf{r} \in \mathcal{R}} \sum_{i=1}^{P+1} \left( p_i(\mathbf{r}) \log \hat{p}_i^c(\mathbf{r}) + p_i(\mathbf{r}) \log \hat{p}_i^f(\mathbf{r}) \right) $$
    [S1].
3.  **Total Loss:**
    $$ \mathcal{L}_{total} = \mathcal{L}_{color} + \lambda \cdot \mathcal{L}_{seg} $$
    where $\lambda$ modulates the segmentation loss [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_ray(ray_origin, ray_direction, image, image_encoder, nerf_model, num_samples):
    """
    Renders a single ray using Category-Level Semantic NeRF.
    """
    # 1. Sample points along the ray
    t_vals = sample_along_ray(ray_origin, ray_direction, num_samples)
    xyz_samples = ray_origin + ray_direction * t_vals
    
    # 2. Project 3D points to the input image plane to get 2D coordinates
    # Assumes camera intrinsics/extrinsics are known for the input image 'image'
    projected_coords = project_to_image_plane(xyz_samples, image.camera_params)
    
    # 3. Extract image features
    # E is the image encoder, W is the feature map
    W = image_encoder(image)
    
    # 4. Sample features from the feature map at projected coordinates
    # Bilinear interpolation or nearest neighbor sampling
    image_features = sample_feature_map(W, projected_coords)
    
    # 5. Encode positions and directions (using shared positional encoding from parent)
    encoded_xyz = positional_encoding(xyz_samples)
    encoded_dirs = positional_encoding(ray_direction)
    
    # 6. Query the NeRF MLP
    # Input: encoded_xyz, encoded_dirs, image_features
    # Output: sigma, color, segmentation_logits
    sigma, color, seg_logits = nerf_model(encoded_xyz, encoded_dirs, image_features)
    
    # 7. Volume Render Color and Segmentation
    # Compute transmittance
    alpha = 1.0 - torch.exp(-sigma * (t_vals[1:] - t_vals[:-1]))
    transmittance = torch.cumprod(1.0 - alpha, dim=0)
    transmittance = torch.cat([torch.ones_like(transmittance[:1]), transmittance[:-1]], dim=0)
    
    weights = alpha * transmittance
    
    # Render Color
    rendered_color = torch.sum(weights.unsqueeze(-1) * color, dim=0)
    
    # Render Segmentation Logits
    rendered_seg_logits = torch.sum(weights.unsqueeze(-1) * seg_logits, dim=0)
    
    # Convert logits to probabilities via softmax
    # Note: Source formula sums denominator over P parts, excluding background index in sum
    # Assuming index P is background.
    part_logits = rendered_seg_logits[:-1] # Exclude background
    rendered_seg_probs = torch.softmax(part_logits, dim=-1)
    
    return rendered_color, rendered_seg_probs, sigma

def compute_loss(rendered_color_c, rendered_color_f, rendered_seg_c, rendered_seg_f, 
                 gt_color, gt_seg_probs, lambda_seg):
    """
    Computes the total training loss.
    """
    # Color Loss: MSE between coarse and fine predictions and GT
    color_loss_c = torch.mean((rendered_color_c - gt_color) ** 2)
    color_loss_f = torch.mean((rendered_color_f - gt_color) ** 2)
    l_color = (color_loss_c + color_loss_f) / 2.0
    
    # Segmentation Loss: Cross-Entropy between coarse and fine predictions and GT
    # gt_seg_probs is the ground truth probability distribution over P+1 classes
    l_seg_c = -torch.mean(gt_seg_probs * torch.log(rendered_seg_c + 1e-8))
    l_seg_f = -torch.mean(gt_seg_probs * torch.log(rendered_seg_f + 1e-8))
    l_seg = l_seg_c + l_seg_f
    
    l_total = l_color + lambda_seg * l_seg
    return l_total
```

**Parameters**
*   $\lambda$: Coefficient modulating segmentation loss. Value not given in the sources. [S1].
*   $P$: Number of object parts. Value not given in the sources. [S1].

**Edge Cases and Pitfalls**
*   **Projection Accuracy:** The method relies on projecting 3D points to the input image plane. If the camera parameters are inaccurate or the object is significantly occluded, the sampled features $W(\pi(\mathbf{x}))$ may be invalid or noisy [S1].
*   **Category Generalization:** The model requires training on a collection of articulated objects belonging to the same category. It does not generalize to unseen categories without retraining [S1].

## Operating characteristics
*   **Time Complexity:** Rendering requires an additional image feature sampling step and a linear layer for segmentation logits per sample, increasing computational cost over standard NeRF. Training requires joint optimization of color and segmentation losses.
*   **Memory Complexity:** Requires storing the image feature map $W$ for the input image(s).
*   **Strengths:** Enables category-level generalization for articulated objects; provides semantic part segmentation alongside geometry and appearance.
*   **Limitations:** Requires part segmentation labels for training; relies on relative camera poses; optimization is lengthy for each object instance if not generalized.
*   **When to use:** When the goal is to reconstruct and segment articulated objects from a category (e.g., chairs, tables) given limited input views, and semantic understanding is required.
*   **When not to use:** For instance-specific reconstruction without semantic needs, or for non-articulated objects where joint attributes are not relevant.

## Background
Category-Level Semantic NeRF (CLA-NeRF) was introduced in 2022 by the authors of the paper "CLA-NeRF: Category-Level Articulated Neural Radiance Field" [S1]. It extends the original NeRF [S1] by adding a segmentation head and conditioning on image features to enable generalization across objects of the same category. The method is specifically designed for articulated objects, allowing for the prediction of joint attributes (rotation axis and pivot point) from the predicted segmentation field [S1].

## Sources
[S1] CLA-NeRF: Category-Level Articulated Neural Radiance Field, https://doi.org/10.1109/icra46639.2022.9812272, fetched 2026-10-29
