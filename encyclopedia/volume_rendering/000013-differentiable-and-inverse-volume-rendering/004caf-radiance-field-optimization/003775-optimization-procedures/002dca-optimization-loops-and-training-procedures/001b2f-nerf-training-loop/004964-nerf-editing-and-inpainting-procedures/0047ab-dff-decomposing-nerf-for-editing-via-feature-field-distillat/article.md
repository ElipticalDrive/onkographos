# DFF (Decomposing NeRF for Editing Via Feature Field Distillation)

Also called: Feature Field Distillation, LSeg-DFF, DINO-DFF

## Overview
DFF enables 3D semantic segmentation and editable novel-view synthesis by distilling features from a pre-trained 2D teacher network (such as LSeg or DINO) into a neural radiance field [S1]. It solves the problem of decomposing scenes into semantic regions for editing, addressing challenges in surface decomposition, volume decomposition, and handling unobserved parts [S1]. Unlike standard NeRFs, whose hidden features are entangled with low-level factors like color or spatial adjacency and fail at semantic decomposition, DFF produces multi-view consistent, semantically meaningful feature fields that allow for selective editing (e.g., deletion, extraction, color transformation) without breaking unintended areas [S1].

## How it works
DFF extends a standard NeRF representation $F_\Theta$ by adding a feature branch or an independent feature field, trained to match features from a 2D teacher network. The method operates in two phases: initial radiance field training, followed by feature field distillation.

### Inputs and Outputs
*   **Inputs**: A set of posed training images $\{I_r\}$, camera poses, and a query modality (text for LSeg, image patch for DINO).
*   **Outputs**: A radiance field $F_\Theta(\mathbf{x}, \mathbf{d})$ outputting density $\sigma$ and color $c$, and a feature field $F_{feat}(\mathbf{x})$ outputting a semantic feature vector $\mathbf{f}$.

### Data Structures and Layouts
The neural radiance field uses an MLP architecture [S1]:
*   **Shared Layers**: Eight ReLU layers with 256 dimensions.
*   **Branching**:
    *   **Density**: A linear layer outputs $\sigma$.
    *   **Color**: Three layers output color $c$.
    *   **Feature**: Three layers output the feature vector $\mathbf{f}$ (in the branch-based variant).
*   **Independent Feature MLP** (alternative variant): Four layers. If positional encoding is used, a skip connection is added at the third layer [S1].
*   **Positional Encoding**: Length 10 for input coordinate $\mathbf{x}$ and its skip connection; length 4 for viewing direction $\mathbf{d}$ [S1].

### Numbered Steps

1.  **Teacher Feature Extraction**:
    For each training image $I_r$ with ray direction $r$, encode the image using the teacher network (LSeg or DINO) to generate target feature maps $f_{img}(I, r)$ [S1].
    *   *Note*: Teacher networks produce reduced-size feature maps. Resize these maps to the original image resolution to align with the NeRF pixel grid [S1].

2.  **Initial Radiance Field Training**:
    Train the NeRF radiance field $F_\Theta$ (without the feature branch) for 200,000 iterations using only the photometric loss $L_p$ [S1].
    *   **Optimizer**: Adam with a linearly decaying learning rate from $5 \times 10^{-4}$ to $8 \times 10^{-5}$ [S1].
    *   **Sampling**: Hierarchical sampling with 64 coarse points (stratified) and 128 fine points (importance sampling) [S1].
    *   **Noise**: Apply Gaussian noise to density during this phase [S1].
    *   **Batch Size**: 1024 rays for Replica dataset; 2048 rays for other datasets [S1].

3.  **Feature Field Distillation (Finetuning)**:
    Add the feature branch (or independent MLP) to the trained NeRF. Finetune the combined network for 5,000 iterations using the composite loss $L = L_p + \lambda L_f$ [S1].
    *   **Parameters**: Remove Gaussian noise. Set learning rate to $1 \times 10^{-4}$ [S1].
    *   **Sampling**: Use coarse sampling (64 points) for training the feature field to ensure smoothness [S1].
    *   **Rendering**: Use coarse sampling for training; fine sampling can be used for final edited rendering to maintain visual quality while keeping semantic coherence [S1].

4.  **Query-Based Decomposition**:
    To edit a specific region, compute the probability $p$ that a 3D point belongs to the query class using the distilled feature field [S1].
    *   **LSeg (Text Query)**: Encode the text label using the LSeg text encoder. Compute the probability of each point using cosine similarity between the point's feature $\mathbf{f}$ and the text embedding [S1]. The probability is calculated via Equation 22 in the source (softmax-like normalization over classes) [S1].
    *   **DINO (Image Patch Query)**: Calculate the patch feature by averaging the features of all pixels in the query patch. Compute the probability $p$ using thresholded cosine similarity between the point's feature and the patch feature [S1]. If similarity exceeds a threshold, set $p=1$ (hard decomposition); otherwise $p=0$ [S1].

5.  **Editing**:
    Apply the edit (e.g., deletion, color change) to rays based on the probability $p$. Rays with high $p$ are modified; rays with low $p$ are preserved from the original radiance field [S1].

### Formulas
*   **Total Loss**:
    $$ L = L_p + \lambda L_f $$
    Where $L_p$ is the photometric loss (e.g., MSE between rendered and input color) and $L_f$ is the feature distillation loss [S1].
*   **Feature Distillation Loss ($L_f$)**:
    $$ L_f = \| F_{feat}(\mathbf{x}) - f_{img}(I, r) \|^2 $$
    (Implicitly defined by the alignment of rendered features to resized teacher features) [S1].
*   **LSeg Probability**:
    $$ p_c(\mathbf{x}) = \frac{\exp(\text{sim}(\mathbf{f}(\mathbf{x}), \mathbf{t}_c))}{\sum_{c'} \exp(\text{sim}(\mathbf{f}(\mathbf{x}), \mathbf{t}_{c'}))} $$
    Where $\mathbf{t}_c$ is the text embedding for class $c$ [S1].
*   **DINO Probability**:
    $$ p(\mathbf{x}) = \begin{cases} 1 & \text{if } \text{sim}(\mathbf{f}(\mathbf{x}), \mathbf{f}_{patch}) > \tau \\ 0 & \text{otherwise} \end{cases} $$
    Where $\mathbf{f}_{patch}$ is the averaged feature of the query patch and $\tau$ is a threshold [S1].

### Parameters
*   **Teacher Networks**: LSeg [44] or DINO [12] (pre-trained, publicly available) [S1].
*   **Training Iterations**: 200K (radiance only), 5K (distillation) [S1].
*   **Learning Rate**: $5 \times 10^{-4} \to 8 \times 10^{-5}$ (linear decay) for initial; $1 \times 10^{-4}$ for distillation [S1].
*   **Sampling**: 64 coarse, 128 fine [S1].
*   **Image Size**: 320×240 (Replica); 1008×756 (others) [S1].
*   **Feature Loss Weight ($\lambda$)**: Standard value not explicitly given, but increasing $\lambda$ by 10x hurts view synthesis [S1].

### Edge Cases and Pitfalls
*   **Feature Loss Convergence**: Feature loss converges significantly faster than photometric loss. Therefore, short finetuning (5K iterations) is sufficient [S1].
*   **Excessive Feature Weight**: Increasing the feature loss weight $\lambda$ by 10x degrades view synthesis quality without improving segmentation performance [S1].
*   **Naive NeRF Features**: Using hidden layers of the original NeRF (e.g., the 8th layer) for decomposition fails; regions leak to other parts with similar colors, geometry, or positions because NeRF features are entangled with low-level factors [S1].
*   **Deletion Artifacts**: Deleting foreground objects may result in floating artifacts and blurred volumes in the far distance behind the deleted object [S1].
*   **Sampling Impact**: Fine sampling introduces high-frequency artifacts in segmentation. Coarse sampling produces smoother segmentations, which is preferred for natural editable novel-view synthesis [S1].
*   **Smoothness Regularization**: To achieve smooth volume decomposition, training an independent feature field MLP *without* positional encoding (PE) acts as a smoothness regularizer, minimizing high-frequency floating artifacts [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DFF:
    def __init__(self, teacher_network, query_type='text', query_value=None):
        """
        Initialize DFF.
        :param teacher_network: Pre-trained LSeg or DINO encoder.
        :param query_type: 'text' for LSeg, 'patch' for DINO.
        :param query_value: Text string or image patch tensor.
        """
        self.teacher = teacher_network
        self.query_type = query_type
        self.query_value = query_value
        
        # Initialize Radiance Field (NeRF)
        # 8 ReLU layers (256 dim), then branches for density, color, feature
        self.radiance_mlp = self._build_radiance_mlp()
        
        # Optional: Independent Feature MLP (if not using branch)
        self.feature_mlp = None 
        self.use_independent_feature = False
        self.no_pe_smoothness = False

    def _build_radiance_mlp(self):
        # Simplified structure: 8 shared layers, then separate heads
        # Shared layers
        shared_layers = []
        for _ in range(8):
            shared_layers.append(nn.Linear(256, 256))
            shared_layers.append(nn.ReLU())
        # Heads
        density_head = nn.Linear(256, 1)
        color_head = nn.Sequential(nn.Linear(256, 256), nn.ReLU(), 
                                   nn.Linear(256, 256), nn.ReLU(), 
                                   nn.Linear(256, 3))
        feature_head = nn.Sequential(nn.Linear(256, 256), nn.ReLU(), 
                                     nn.Linear(256, 256), nn.ReLU(), 
                                     nn.Linear(256, 256)) # Feature dim depends on teacher
        return nn.ModuleList([nn.Sequential(*shared_layers), density_head, color_head, feature_head])

    def train_radiance_field(self, images, poses, iterations=200000):
        """
        Phase 1: Train radiance field without feature branch.
        """
        optimizer = torch.optim.Adam(self.radiance_mlp.parameters(), lr=5e-4)
        # Linear decay to 8e-5
        scheduler = torch.optim.lr_scheduler.LinearLR(optimizer, start_factor=1.0, end_factor=0.16, total_iters=iterations)
        
        for i in range(iterations):
            rays = self.sample_rays(images, poses, batch_size=1024) # 2048 for non-Replica
            x, d = rays['xyz'], rays['direction']
            
            # Positional Encoding
            x_pe = self.positional_encode(x, length=10)
            d_pe = self.positional_encode(d, length=4)
            
            # Forward through shared layers
            shared_out = self.radiance_mlp[0](x_pe)
            
            # Density
            sigma = self.radiance_mlp[1](shared_out)
            # Add Gaussian noise to sigma
            sigma = sigma + torch.randn_like(sigma) * 0.1
            
            # Color
            color = self.radiance_mlp[2](shared_out)
            
            # Volume Rendering (Coarse: 64 samples, Stratified)
            # (Implementation of standard NeRF volume rendering omitted for brevity, 
            # assuming standard differentiable renderer)
            rendered_color = self.volume_render(x, d, sigma, color, num_samples=64, sampling='stratified')
            
            # Loss
            target_color = rays['color']
            loss = F.mse_loss(rendered_color, target_color)
            
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            scheduler.step()

    def distill_features(self, images, poses, iterations=5000, lambda_f=1.0):
        """
        Phase 2: Finetune with feature branch/distillation.
        """
        # Remove Gaussian noise
        # Set LR to 1e-4
        optimizer = torch.optim.Adam(self.radiance_mlp.parameters(), lr=1e-4)
        
        # Pre-compute teacher features
        teacher_features = []
        for img in images:
            # Encode image
            feat_map = self.teacher.encode(img)
            # Resize to original image size
            feat_map_resized = F.interpolate(feat_map, size=img.shape[1:], mode='bilinear', align_corners=False)
            teacher_features.append(feat_map_resized)

        for i in range(iterations):
            rays = self.sample_rays(images, poses, batch_size=1024)
            x, d = rays['xyz'], rays['direction']
            
            x_pe = self.positional_encode(x, length=10)
            d_pe = self.positional_encode(d, length=4)
            
            shared_out = self.radiance_mlp[0](x_pe)
            
            # Density and Color (for photometric loss)
            sigma = self.radiance_mlp[1](shared_out)
            color = self.radiance_mlp[2](shared_out)
            
            # Feature (for distillation loss)
            feat = self.radiance_mlp[3](shared_out)
            
            # Volume Rendering for Color (Coarse sampling for consistency with training)
            rendered_color = self.volume_render(x, d, sigma, color, num_samples=64, sampling='stratified')
            
            # Volume Rendering for Feature (Coarse sampling)
            rendered_feat = self.volume_render_feat(x, d, sigma, feat, num_samples=64, sampling='stratified')
            
            # Photometric Loss
            loss_p = F.mse_loss(rendered_color, rays['color'])
            
            # Feature Loss
            # Map rendered feature to 2D plane to compare with teacher feature map
            # (Assuming ray casting maps 3D point to 2D pixel)
            target_feat = self.get_target_feature_from_ray(rays, teacher_features)
            loss_f = F.mse_loss(rendered_feat, target_feat)
            
            loss = loss_p + lambda_f * loss_f
            
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

    def get_query_probability(self, x):
        """
        Compute probability of point x belonging to query.
        """
        x_pe = self.positional_encode(x, length=10)
        shared_out = self.radiance_mlp[0](x_pe)
        feat = self.radiance_mlp[3](shared_out)
        
        if self.query_type == 'text':
            # LSeg: Text embedding
            text_emb = self.teacher.encode_text(self.query_value)
            # Cosine similarity
            sim = F.cosine_similarity(feat, text_emb, dim=-1)
            # Softmax over classes (simplified, assuming single class prob or multi-class)
            # Equation 22 implies softmax normalization
            prob = torch.sigmoid(sim) # Simplified for binary/text match
            return prob
        elif self.query_type == 'patch':
            # DINO: Patch feature
            patch_feat = self.query_value # Pre-averaged patch feature
            sim = F.cosine_similarity(feat, patch_feat, dim=-1)
            threshold = 0.5 # Example threshold
            prob = (sim > threshold).float()
            return prob

    def positional_encode(self, x, length):
        # Standard NeRF positional encoding
        # ...
        pass

    def volume_render(self, x, d, sigma, color, num_samples, sampling):
        # Standard NeRF volume rendering
        # ...
        pass

    def volume_render_feat(self, x, d, sigma, feat, num_samples, sampling):
        # Volume rendering for feature field
        # ...
        pass

    def sample_rays(self, images, poses, batch_size):
        # Sample rays from images
        # ...
        pass

    def get_target_feature_from_ray(self, rays, teacher_features):
        # Retrieve teacher feature at ray's 2D pixel location
        # ...
        pass
```

## Operating characteristics
*   **Time Complexity**: Training requires two phases: 200K iterations for radiance field and 5K for distillation. The distillation phase is short because feature loss converges faster than photometric loss [S1].
*   **Memory Complexity**: Adds a feature branch (3 layers) or an independent MLP (4 layers) to the NeRF. The branch-based approach has small computational and parameter overhead compared to original NeRF [S1].
*   **Performance**: DFF achieves 3D semantic segmentation performance comparable to or better than supervised sparse 3D convolution models (e.g., MinkowskiNet42) [S1].
*   **Strengths**:
    *   Enables multi-view consistent editing (deletion, extraction) unlike image inpainting baselines (e.g., LaMa) [S1].
    *   Coarse sampling produces smooth segmentations, avoiding high-frequency artifacts [S1].
    *   Independent feature MLP allows introducing DFF into arbitrary 3D representations (dynamic NeRFs, meshes) without retraining the radiance field [S1].
*   **Limitations**:
    *   Deleting foreground objects can leave floating artifacts and blurred volumes in the background [S1].
    *   Excessive feature loss weight degrades view synthesis [S1].
    *   Naive use of NeRF hidden features fails at semantic decomposition [S1].

## Background
DFF was introduced in the paper "Decomposing NeRF for Editing Via Feature Field Distillation" [S1]. It addresses the limitations of editable NeRFs, which struggle with surface decomposition, volume decomposition, and handling unobserved regions [S1]. The method leverages pre-trained 2D teacher networks (LSeg for text-based segmentation, DINO for patch-based similarity) to distill semantic features into the 3D radiance field, enabling zero-shot semantic segmentation and editing [S1].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
