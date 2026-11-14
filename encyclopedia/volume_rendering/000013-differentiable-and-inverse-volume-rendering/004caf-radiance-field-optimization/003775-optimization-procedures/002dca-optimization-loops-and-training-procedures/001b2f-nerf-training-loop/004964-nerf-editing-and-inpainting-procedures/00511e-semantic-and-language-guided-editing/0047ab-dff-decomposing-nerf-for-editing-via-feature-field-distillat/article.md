# DFF (Decomposing NeRF for Editing Via Feature Field Distillation)

Also called: Feature Field Distillation, LSeg-DFF, DINO-DFF

## Overview
DFF enables 3D semantic segmentation and editable novel-view synthesis by distilling features from a pre-trained 2D teacher network (such as LSeg or DINO) into a neural radiance field [S1]. It solves the problem that standard NeRF feature spaces are entangled with low-level factors like color or spatial adjacency rather than semantic similarity, making naive decomposition unreliable [S1]. Unlike methods that rely on semantic labels during training, DFF achieves this through unsupervised distillation, allowing for zero-shot text-based or image-patch-based editing without retraining the underlying radiance field geometry [S1].

## How it works
**Inputs:**
*   A set of posed training images $I_r$ with camera poses $r$.
*   A pre-trained NeRF radiance field $f_\theta$ (or a newly trained one).
*   A 2D teacher network (LSeg or DINO) and its associated query mechanism (text prompt for LSeg, image patch for DINO).

**Outputs:**
*   A decomposed NeRF representation consisting of a radiance field (density $\sigma$ and color $\mathbf{c}$) and a feature field $\mathbf{f}$.
*   Semantic segmentation probabilities for any 3D point or 2D pixel.

**Data Structures:**
The implementation uses two primary neural network architectures:

1.  **Radiance Field MLP ($f_\theta$):**
    *   Input: Positional encoded coordinate $\mathbf{x}$ and viewing direction $\mathbf{d}$.
    *   Structure: 8 hidden ReLU layers with 256 dimensions.
    *   Outputs:
        *   Density $\sigma$: Linear layer (1 output).
        *   Color $\mathbf{c}$: 3 hidden layers (256 dims) $\to$ Linear layer (3 outputs, RGB).
        *   Feature $\mathbf{f}$: 3 hidden layers (256 dims) $\to$ Linear layer (output dim depends on teacher, e.g., 256 for DINO, 128 for LSeg).
    *   *Note:* In the "branch-based" variant, the feature branch shares the first 8 layers with the radiance field. In the "independent" variant, the feature field is a separate MLP.

2.  **Independent Feature Field MLP (Optional):**
    *   Used to avoid retraining the radiance field.
    *   Structure: 4 hidden layers.
    *   Skip connection: Added at the 3rd layer if positional encoding is used.
    *   Input: Positional encoded coordinate $\mathbf{x}$ (and optionally $\mathbf{d}$).
    *   Output: Feature vector $\mathbf{f}$.

**Positional Encoding:**
*   Input coordinate $\mathbf{x}$: Length 10.
*   Viewing direction $\mathbf{d}$: Length 4.
*   Skip connections: Applied to the input coordinate.

**Sampling Strategy:**
*   **Coarse MLP:** Stratified sampling with $N_c = 64$ points.
*   **Fine MLP:** Importance sampling with $N_f = 128$ points (total 192 points including coarse).
*   *Distillation Note:* Feature fields are trained using **coarse sampling** to ensure smoothness. Rendering for editing uses **fine sampling**.

**Training Phases:**

1.  **Phase 1: Radiance Field Training (Standard NeRF)**
    *   Train the radiance field $f_\theta$ for 200,000 iterations.
    *   Loss: Photometric loss $L_p$ (e.g., $L_1$ or SSIM between rendered and input images).
    *   Optimizer: Adam, linearly decaying learning rate from $5 \times 10^{-4}$ to $8 \times 10^{-5}$.
    *   Regularization: Gaussian noise added to density.
    *   Batch size: 1024 rays (Replica) or 2048 rays (others).
    *   Image resolution: 320×240 (Replica) or 1008×756 (others).

2.  **Phase 2: Feature Field Distillation (Finetuning)**
    *   Initialize feature branch (either attached to radiance field or independent).
    *   Remove Gaussian noise from density.
    *   Set learning rate to $1 \times 10^{-4}$.
    *   Train for 5,000 iterations.
    *   Loss: $L = L_p + \lambda L_f$.
    *   *Note:* Feature loss converges significantly faster than photometric loss; thus, short training is sufficient. Excessively increasing $\lambda$ (e.g., $\times 10$) degrades view synthesis without improving segmentation.

**Feature Distillation Loss ($L_f$):**
The feature loss minimizes the distance between the rendered feature $\mathbf{f}_{render}$ and the target teacher feature $\mathbf{f}_{img}$.

$$
L_f = \sum_{r} \sum_{i \in \text{pixels}} \| \mathbf{f}_{render}(r, i) - \mathbf{f}_{img}(I_r, r, i) \|^2
$$

Where:
*   $\mathbf{f}_{img}(I_r, r, i)$ is the feature map from the teacher network for image $I_r$ at pixel $i$.
*   Teacher feature maps are resized to the original image size to match the NeRF pixel grid.
*   $\mathbf{f}_{render}$ is obtained by volume rendering the feature field $\mathbf{f}$ along the ray, similar to color rendering:
    $$
    \mathbf{f}_{render} = \sum_{j=1}^{N} T_j (1 - \alpha_j) \mathbf{f}_j
    $$
    where $T_j$ is the accumulated transmittance and $\alpha_j$ is the opacity at sample $j$.

**Semantic Probability Calculation:**
To perform segmentation or editing, the probability $p$ that a 3D point (or rendered pixel) belongs to a query is computed differently for LSeg and DINO.

1.  **LSeg (Text Query):**
    *   Encode text label $t$ using LSeg text encoder to get $\mathbf{t}_{vec}$.
    *   Compute cosine similarity between rendered feature $\mathbf{f}_{render}$ and $\mathbf{t}_{vec}$.
    *   Probability $p$ is derived via softmax over all possible labels (Equation 22 in [S1]).

2.  **DINO (Image Patch Query):**
    *   Select a query patch from an image.
    *   Compute patch feature $\mathbf{q}$ by averaging DINO features of all pixels in the patch.
    *   Compute cosine similarity between $\mathbf{f}_{render}$ and $\mathbf{q}$.
    *   **Hard Decomposition:** If similarity $> \tau$ (threshold), $p=1$; else $p=0$.
    *   *Note:* DINO does not use softmax with negative queries; it uses direct thresholded cosine similarity.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DFF:
    def __init__(self, teacher_type='LSeg', use_independent_feature=False):
        """
        teacher_type: 'LSeg' or 'DINO'
        use_independent_feature: If True, trains a separate MLP for features.
        """
        self.teacher_type = teacher_type
        self.use_independent_feature = use_independent_feature
        
        # Radiance Field MLP (Shared or Base)
        # 8 hidden layers, 256 dims, ReLU
        self.radiance_mlp = RadianceMLP(hidden_dim=256, num_hidden=8)
        
        if use_independent_feature:
            # Independent Feature MLP: 4 layers, skip connection at 3rd
            self.feature_mlp = IndependentFeatureMLP(hidden_dim=256, num_hidden=4)
        else:
            # Branch-based: Feature output comes from radiance_mlp
            pass
            
        self.teacher = load_teacher_network(teacher_type)
        
    def encode_input(self, x, d):
        """Apply positional encoding."""
        # x: [B, 3], d: [B, 3]
        # PE length 10 for x, 4 for d
        x_pe = positional_encoding(x, num_frequencies=10)
        d_pe = positional_encoding(d, num_frequencies=4)
        return x_pe, d_pe

    def forward_radiance(self, x_pe, d_pe):
        """Forward pass through radiance MLP."""
        # x_pe includes skip connection to input
        h = self.radiance_mlp(x_pe, d_pe)
        sigma = h['sigma']
        color = h['color']
        if not self.use_independent_feature:
            feature = h['feature']
        else:
            feature = None
        return sigma, color, feature

    def forward_feature(self, x_pe):
        """Forward pass through feature MLP."""
        if self.use_independent_feature:
            return self.feature_mlp(x_pe)
        else:
            # Feature is already returned by radiance forward
            return None

    def render_ray(self, ray_origins, ray_directions, sigma, color, feature):
        """
        Standard volume rendering.
        Inputs: Ray parameters, and per-sample outputs from MLPs.
        Returns: Rendered color, feature, and weights.
        """
        # Stratified sampling for coarse, Importance for fine
        # This function assumes sigma, color, feature are tensors of shape [N_samples, ...]
        
        # Compute alpha from sigma and delta (distance between samples)
        # alpha = 1 - exp(-sigma * delta)
        # Transmittance T = product(1 - alpha)
        
        # Rendered Color C = sum(T * alpha * color)
        # Rendered Feature F = sum(T * alpha * feature)
        
        # [G] Standard volume rendering implementation details omitted for brevity,
        # assuming standard differentiable renderer is available.
        pass

    def get_teacher_features(self, image):
        """
        Get target features from teacher network.
        Resize to original image size.
        """
        raw_features = self.teacher.encode(image)
        # Resize raw_features to match image resolution
        target_features = F.interpolate(raw_features, size=image.shape[2:], mode='bilinear', align_corners=False)
        return target_features

    def compute_loss(self, rendered_color, rendered_feature, target_color, target_feature, query_vector=None):
        """
        Compute photometric and feature losses.
        """
        # Photometric Loss (L_p)
        l_p = F.l1_loss(rendered_color, target_color)
        
        # Feature Loss (L_f)
        if self.teacher_type == 'LSeg':
            # Cosine similarity with text vector, then softmax/probability
            # [G] Assuming target_feature is already the probability map or feature vector
            # L_f is MSE between rendered feature and teacher feature
            l_f = F.mse_loss(rendered_feature, target_feature)
        elif self.teacher_type == 'DINO':
            # Cosine similarity with patch query
            # [G] If using hard decomposition, loss might be binary cross entropy on thresholded prob
            # But typically distillation minimizes feature distance
            l_f = F.mse_loss(rendered_feature, target_feature)
            
        return l_p, l_f

    def train_step(self, images, poses, optimizer):
        """
        Single training step for distillation phase.
        """
        optimizer.zero_grad()
        
        # 1. Sample rays
        rays = sample_rays(images, poses, batch_size=1024)
        
        # 2. Get teacher features for these rays
        # Note: Teacher features must be aligned with ray origins/directions
        teacher_feats = self.get_teacher_features(images)
        target_feats = interpolate_teacher_features(teacher_feats, rays)
        
        # 3. Forward pass (Coarse sampling for feature training)
        # [G] Hierarchical sampling logic:
        # Coarse samples -> Coarse MLP -> Coarse Rendering -> Fine Sampling -> Fine MLP
        # For DFF, we train feature field using Coarse sampling strategy
        
        sigma_coarse, color_coarse, feat_coarse = self.forward_radiance(rays.origins, rays.directions)
        # [G] Perform coarse volume rendering to get weights for fine sampling
        # Then sample fine points based on weights
        sigma_fine, color_fine, feat_fine = self.forward_radiance(fine_origins, fine_directions)
        
        # Render final color and feature
        rendered_color, rendered_feature = self.render_ray(
            rays.origins, rays.directions, 
            sigma_coarse, color_coarse, feat_coarse,
            sigma_fine, color_fine, feat_fine
        )
        
        # 4. Compute Loss
        target_color = rays.colors
        l_p, l_f = self.compute_loss(rendered_color, rendered_feature, target_color, target_feats)
        
        # Lambda for feature loss (typical value not explicitly given, but note warns against *10)
        lambda_f = 1.0 
        loss = l_p + lambda_f * l_f
        
        loss.backward()
        optimizer.step()
        
        return loss.item()

    def edit_scene(self, query_type, query_value, render_fn):
        """
        Edit scene based on query.
        query_type: 'text' (LSeg) or 'patch' (DINO)
        """
        if query_type == 'text':
            # Encode text
            query_vec = self.teacher.encode_text(query_value)
            # Get probabilities for all points
            probs = compute_lseg_probability(render_fn, query_vec)
        elif query_type == 'patch':
            # Encode patch
            query_vec = self.teacher.encode_patch(query_value)
            # Get similarity
            sims = compute_dino_similarity(render_fn, query_vec)
            # Threshold for hard decomposition
            probs = (sims > threshold).float()
            
        return probs
```

**Edge Cases and Pitfalls:**
*   **Feature Loss Weight:** Excessively increasing the feature loss weight ($\lambda \times 10$) hurts view synthesis quality without improving segmentation [S1].
*   **Sampling Mismatch:** Using fine sampling for training feature fields introduces high-frequency artifacts. Coarse sampling is preferred for smooth decomposition [S1].
*   **Naive NeRF Features:** Using hidden layers of the original NeRF (e.g., the 8th layer) for decomposition is not robust; selections leak to regions with similar color/geometry but different semantics [S1].
*   **Deletion Artifacts:** Deleting foreground objects may leave floating artifacts and blurred volumes in the far distance behind the object [S1].
*   **Training Duration:** Feature loss converges much faster than photometric loss; 5K iterations is sufficient for distillation [S1].

## Operating characteristics
*   **Time Complexity:** Distillation phase is short (5,000 iterations) compared to initial training (200,000 iterations) [S1].
*   **Memory Complexity:** Branch-based DFF adds minimal overhead (3 extra layers). Independent DFF adds a separate small MLP (4 layers) [S1].
*   **Performance:** DFF achieves 3D semantic segmentation performance comparable to or better than supervised sparse 3D convolution models (e.g., MinkowskiNet) [S1].
*   **Strengths:**
    *   Enables zero-shot text-based and image-patch-based editing [S1].
    *   Multi-view consistent results for deletion/extraction, unlike 2D inpainting baselines (LaMa) [S1].
    *   Independent feature field allows adding semantics to existing NeRFs without retraining geometry [S1].
*   **Limitations:**
    *   Struggles with "volume decomposition" smoothness if fine sampling is used for training features [S1].
    *   Deletion can leave artifacts in occluded background regions [S1].
    *   Original NeRF features are not semantically meaningful [S1].

## Background
DFF was introduced in the paper "Decomposing NeRF for Editing Via Feature Field Distillation" [S1]. It addresses the limitations of editable NeRFs, which previously struggled with surface decomposition, volume decomposition, and estimating unobserved parts [S1]. The method leverages pre-trained 2D vision-language models (LSeg) and self-supervised models (DINO) as teachers to distill semantic information into the 3D field, bypassing the need for 3D semantic labels during training [S1].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
