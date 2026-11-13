# DFF (Decomposing NeRF for Editing Via Feature Field Distillation)


## Overview
DFF enables 3D semantic segmentation and editable novel-view synthesis by distilling features from a pre-trained 2D teacher network (such as LSeg or DINO) into a neural radiance field [S1]. It solves the problem that standard NeRF feature spaces are entangled with low-level factors like color or spatial adjacency rather than semantic similarity, making naive decomposition unreliable [S1]. Unlike sibling methods that rely on optimization-based editing or inference-time interception of geometry, DFF explicitly learns a semantic feature field via distillation, allowing for multi-view consistent region selection, deletion, and extraction without requiring semantic label supervision during training [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of posed training images $I$, camera poses, and a pre-trained 2D teacher network (LSeg or DINO). For DINO-DFF, an image patch query is also required.
*   **Outputs:** A radiance field $F_\Theta$ mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$ and a feature field $F_\Phi$ mapping $(\mathbf{x}, \mathbf{d}) \to \mathbf{f}$, where $\mathbf{f}$ represents semantic features.

**Data Structures:**
1.  **Radiance Field MLP ($F_\Theta$):**
    *   Architecture: 8 ReLU layers with 256 dimensions.
    *   Output heads: A linear layer for density $\sigma$, 3 linear layers for color $\mathbf{c}$, and 3 linear layers for features $\mathbf{f}$ (if using branch-based architecture) [S1].
    *   Positional Encoding (PE): Length 10 for input coordinate $\mathbf{x}$ and its skip connection; Length 4 for viewing direction $\mathbf{d}$ [S1].
2.  **Independent Feature Field MLP ($F_\Phi$) (Optional):**
    *   Architecture: 4 layers.
    *   Skip connection: At the third layer if PE is used [S1].
    *   Smoothness variant: An independent MLP without positional encoding can be used as a smoothness regularizer [S1].
3.  **Teacher Network Features:**
    *   Target feature maps $f_{img}(I, \mathbf{r})$ resized to the original image size to compensate for reduced resolution from the teacher network [S1].

**Training Procedure:**
The method consists of two phases: initial radiance field training and feature field distillation.

1.  **Phase 1: Radiance Field Training**
    *   Train the NeRF (without feature branch) for 200,000 iterations using photometric loss $L_p$ [S1].
    *   Optimizer: Adam with linearly decaying learning rate from $5 \times 10^{-4}$ to $8 \times 10^{-5}$ [S1].
    *   Sampling: Coarse sampling with 64 points (stratified); Fine sampling with 128 points (importance sampling) [S1].
    *   Noise: Gaussian noise is applied to density during this phase [S1].
    *   Batch size: 1024 rays for Replica dataset; 2048 rays for other datasets [S1].

2.  **Phase 2: Feature Field Distillation**
    *   **Setup:** Remove Gaussian noise. Set learning rate to $1 \times 10^{-4}$ [S1].
    *   **Training Duration:** 5,000 iterations [S1].
    *   **Sampling:** Use coarse sampling (64 points) for training the feature branch. For edited rendering, fine sampling can be used, but coarse training ensures smoother segmentations [S1].
    *   **Loss Function:** Minimize $L = L_p + \lambda L_f$, where $L_p$ is photometric loss and $L_f$ is feature distillation loss [S1].
    *   **Feature Loss ($L_f$):** Defined in Equation 4 of the source [S1]. It measures the difference between the rendered feature field output and the resized teacher network feature map at the corresponding ray position.

**Querying and Decomposition:**
*   **LSeg-DFF (Text Query):**
    *   Encode text labels using the LSeg text encoder.
    *   Calculate probability $p$ of each point using Equation 22 (softmax over negative queries) [S1].
*   **DINO-DFF (Image Patch Query):**
    *   Calculate patch feature by averaging features of all pixels in the query patch.
    *   Compute thresholded cosine similarity between the query patch feature and the rendered feature $\mathbf{f}$.
    *   Set $p = 1$ if similarity exceeds threshold, $p = 0$ otherwise (hard decomposition) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DFF:
    def __init__(self, teacher_network, use_independent_feature_mlp=False, use_no_pe_smoothness=False):
        """
        Initialize DFF.
        :param teacher_network: Pre-trained LSeg or DINO encoder.
        :param use_independent_feature_mlp: If True, use separate MLP for features.
        :param use_no_pe_smoothness: If True, use MLP without positional encoding for smoothness.
        """
        self.teacher = teacher_network
        self.use_independent = use_independent_feature_mlp
        self.use_no_pe = use_no_pe_smoothness
        
        # Radiance Field MLP (Shared or Base)
        # 8 ReLU layers, 256 dims. Linear heads for sigma, color, feature.
        self.radiance_mlp = self._build_radiance_mlp()
        
        # Feature Field MLP
        if self.use_independent:
            if self.use_no_pe:
                self.feature_mlp = self._build_feature_mlp_no_pe()
            else:
                self.feature_mlp = self._build_feature_mlp_with_pe()
        else:
            self.feature_mlp = None # Features come from radiance_mlp branch

    def _build_radiance_mlp(self):
        # 8 ReLU layers, 256 dims
        layers = []
        for _ in range(8):
            layers.append(nn.Linear(256, 256))
            layers.append(nn.ReLU())
        # Linear layer for density
        layers.append(nn.Linear(256, 1))
        # 3 layers for color
        color_layers = []
        for _ in range(3):
            color_layers.append(nn.Linear(256, 256))
            color_layers.append(nn.ReLU())
        color_layers.append(nn.Linear(256, 3))
        layers.extend(color_layers)
        # 3 layers for feature (if branch-based)
        feat_layers = []
        for _ in range(3):
            feat_layers.append(nn.Linear(256, 256))
            feat_layers.append(nn.ReLU())
        feat_layers.append(nn.Linear(256, 256)) # Feature dimension
        layers.extend(feat_layers)
        return nn.Sequential(*layers)

    def _build_feature_mlp_with_pe(self):
        # 4 layers, skip connection at 3rd layer
        layers = []
        layers.append(nn.Linear(256, 256))
        layers.append(nn.ReLU())
        layers.append(nn.Linear(256, 256))
        layers.append(nn.ReLU())
        # Skip connection logic handled in forward
        layers.append(nn.Linear(256, 256))
        layers.append(nn.ReLU())
        layers.append(nn.Linear(256, 256))
        return nn.Sequential(*layers)

    def _build_feature_mlp_no_pe(self):
        # 4 layers, no PE, no skip connection mentioned for this variant
        layers = []
        layers.append(nn.Linear(256, 256))
        layers.append(nn.ReLU())
        layers.append(nn.Linear(256, 256))
        layers.append(nn.ReLU())
        layers.append(nn.Linear(256, 256))
        layers.append(nn.ReLU())
        layers.append(nn.Linear(256, 256))
        return nn.Sequential(*layers)

    def positional_encoding(self, x, pe_len):
        # Standard NeRF PE
        scales = torch.tensor([2**i for i in range(pe_len//2)])
        xb = torch.reshape(x, (x.shape[0], -1, x.shape[1]))[:, :, None, :] * scales[None, None, :, :]
        xb = torch.reshape(xb, (x.shape[0], -1, 2*pe_len//2))
        return torch.sin(torch.cat([xb, xb + torch.pi/2], dim=-1))

    def forward_radiance(self, x, d):
        # x: [N, 3], d: [N, 3]
        pe_x = self.positional_encoding(x, 10)
        pe_d = self.positional_encoding(d, 4)
        
        # Concatenate x and skip connection
        input_vec = torch.cat([pe_x, x], dim=-1)
        
        h = self.radiance_mlp[:9](input_vec) # First 9 modules: 8 ReLU + 1 Linear? 
        # Correction: The source says "8 ReLU layers... followed by linear layer for density, 3 for color, 3 for feature".
        # This implies the first 8 layers are shared.
        # Let's assume standard NeRF structure where first 8 layers are shared.
        
        # Re-implementing based on source: "eight ReLU layers with 256 dimensions, followed by a linear layer for density, 
        # three layers for color, and three layers for feature"
        # This usually means:
        # Shared: Linear(256, 256) -> ReLU (x8)
        # Density Head: Linear(256, 1)
        # Color Head: Linear(256, 256) -> ReLU -> Linear(256, 256) -> ReLU -> Linear(256, 3)
        # Feature Head: Linear(256, 256) -> ReLU -> Linear(256, 256) -> ReLU -> Linear(256, 256)
        
        # For simplicity in pseudocode, we assume the module structure handles this.
        # In a real implementation, you'd split the weights.
        pass 

    def train_phase1(self, rays, images, optimizer, num_iters=200000):
        """Phase 1: Train Radiance Field only."""
        self.radiance_mlp.train()
        if self.use_independent:
            self.feature_mlp.train()
            
        for i in range(num_iters):
            # Sample rays
            # Render using coarse sampling (64 points)
            sigma, color, feature = self.render_rays(rays, sampling='coarse', use_feature=False)
            
            # Photometric Loss
            target_color = get_target_color(rays)
            loss_p = F.mse_loss(color, target_color)
            
            # Add Gaussian noise to density
            sigma_noisy = sigma + torch.randn_like(sigma) * 0.01 # [G] Placeholder noise scale
            
            loss_p.backward()
            optimizer.step()
            optimizer.zero_grad()
            
            # Decay LR linearly from 5e-4 to 8e-5
            adjust_learning_rate(optimizer, i, num_iters, 5e-4, 8e-5)

    def train_phase2(self, rays, images, teacher_features, optimizer, num_iters=5000, lambda_f=1.0):
        """Phase 2: Distill Features."""
        self.radiance_mlp.eval() # Freeze radiance field? Source says "finetune them with a feature branch".
        # Usually, the radiance field is kept fixed or lightly updated. Source implies finetuning the feature branch.
        # "Gaussian noise is removed, and the learning rate is set to 1e-4"
        
        if self.use_independent:
            self.feature_mlp.train()
        else:
            # If branch-based, the feature head of radiance_mlp is trained
            pass
            
        optimizer = torch.optim.Adam(self.get_feature_params(), lr=1e-4)
        
        for i in range(num_iters):
            # Render using coarse sampling (64 points) for feature training
            sigma, color, feature = self.render_rays(rays, sampling='coarse', use_feature=True)
            
            # Photometric Loss
            target_color = get_target_color(rays)
            loss_p = F.mse_loss(color, target_color)
            
            # Feature Loss
            # Resize teacher features to match ray positions
            target_feature = resize_teacher_features(teacher_features, rays)
            loss_f = F.mse_loss(feature, target_feature)
            
            loss = loss_p + lambda_f * loss_f
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()

    def render_rays(self, rays, sampling='coarse', use_feature=False):
        """
        Volume rendering.
        :param rays: Origin and direction.
        :param sampling: 'coarse' (64 points) or 'fine' (128 points).
        :param use_feature: If True, return feature field output.
        """
        if sampling == 'coarse':
            num_samples = 64
            sample_method = 'stratified'
        else:
            num_samples = 128
            sample_method = 'importance'
            
        t_vals = sample_t(rays, num_samples, sample_method)
        z_vals = t_vals
        
        # Query MLP
        x = rays.origins + rays.directions * z_vals.unsqueeze(-1)
        d = rays.directions.unsqueeze(1).expand_as(x)
        
        if use_feature and self.use_independent:
            # Independent feature MLP
            pe_x = self.positional_encoding(x, 10)
            pe_d = self.positional_encoding(d, 4)
            input_vec = torch.cat([pe_x, x], dim=-1)
            
            # Forward through feature MLP
            h = self.feature_mlp(input_vec)
            # Handle skip connection if present
            if hasattr(self.feature_mlp, 'skip_connection'):
                h = h + self.feature_mlp.skip(input_vec)
            feature = h
            sigma, color = self.radiance_mlp.get_sigma_and_color(x, d)
        else:
            # Branch-based or no feature
            sigma, color, feature = self.radiance_mlp(x, d)
            
        # Volume Rendering Equation
        # C(r) = sum T(t_i) (1 - exp(-sigma(t_i) delta_i)) c(t_i)
        deltas = z_vals[:, 1:] - z_vals[:, :-1]
        delta_inf = 1e10 * torch.ones_like(deltas[:, :1])
        deltas = torch.cat([deltas, delta_inf], dim=-1)
        
        alpha = 1.0 - torch.exp(-sigma * deltas)
        T = torch.cumprod(1.0 - alpha + 1e-10, dim=-1)
        weights = alpha * T[:, :-1]
        
        color_out = torch.sum(weights.unsqueeze(-1) * color, dim=-2)
        sigma_out = torch.sum(weights * sigma, dim=-2) # Transmittance/opacity related
        
        if use_feature:
            feature_out = torch.sum(weights.unsqueeze(-1) * feature, dim=-2)
            return sigma_out, color_out, feature_out
        else:
            return sigma_out, color_out, None

    def query_decomposition(self, rendered_feature, query_type='text', query_data=None):
        """
        Compute probability map for editing.
        """
        if query_type == 'text':
            # LSeg: Encode text, compute probability via softmax (Eq 22)
            text_emb = self.teacher.encode_text(query_data)
            prob = self.teacher.compute_prob(rendered_feature, text_emb)
        elif query_type == 'patch':
            # DINO: Thresholded cosine similarity
            patch_feat = self.teacher.encode_patch(query_data)
            sim = F.cosine_similarity(rendered_feature, patch_feat, dim=-1)
            prob = (sim > self.threshold).float()
        return prob
```

## Operating characteristics
*   **Time Complexity:** Training requires 200,000 iterations for the radiance field and 5,000 for distillation. The distillation phase is short because feature loss converges significantly faster than photometric loss [S1].
*   **Memory Complexity:** Branch-based DFF adds minimal overhead as it shares layers with the radiance field. Independent feature MLP adds lightweight parameters [S1].
*   **Performance:** DFF achieves 3D semantic segmentation performance comparable to or better than supervised sparse 3D convolution models like MinkowskiNet42 [S1].
*   **Strengths:**
    *   Enables multi-view consistent editing (deletion/extraction) unlike image inpainting baselines like LaMa [S1].
    *   Independent feature MLP allows introducing DFF to arbitrary 3D representations (dynamic NeRFs, meshes) without retraining the radiance field [S1].
    *   Coarse sampling produces smooth segmentations, avoiding high-frequency artifacts common in fine sampling [S1].
*   **Limitations:**
    *   Deleting foreground objects can leave floating artifacts and blurred volumes in the far distance behind the deleted object [S1].
    *   Excessively increasing the feature loss weight ($\lambda \times 10$) hurts view synthesis without improving segmentation [S1].
    *   Naive use of NeRF's internal hidden layers (e.g., 8th layer) is not robust for decomposition due to entanglement with low-level factors [S1].

## Background
DFF was introduced in the paper "Decomposing NeRF for Editing Via Feature Field Distillation" [S1]. It addresses the challenges of editable NeRFs, specifically surface decomposition, volume decomposition, lighting decomposition, and estimation of unobserved parts [S1]. The method leverages pre-trained 2D teacher networks (LSeg and DINO) to distill semantic knowledge into the 3D radiance field, enabling zero-shot semantic segmentation and editing [S1].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
