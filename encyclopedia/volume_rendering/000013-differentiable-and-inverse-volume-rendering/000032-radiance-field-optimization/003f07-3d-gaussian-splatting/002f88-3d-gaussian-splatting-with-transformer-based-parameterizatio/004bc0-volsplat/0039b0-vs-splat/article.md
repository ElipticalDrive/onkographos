# VS-Splat

Also called: Voxel-Selective feed-forward Gaussian Splatting

## Overview
VS-Splat is a feed-forward framework for 3D object reconstruction from sparse-view 2D images that employs a coarse-to-fine (C2F) strategy to predict 3D Gaussian primitives [S1, unofficial]. It distinguishes itself from sibling methods like LaRa and GenDen by performing voxel-level selection *before* generating fine primitives, rather than selecting existing primitives for refinement [S1, unofficial]. This ensures that the coarse representation determines the spatial location for new fine primitives, enabling adaptive density control without auxiliary supervision [S1, unofficial].

## How it works
**Inputs:** $N$ 2D images $\{i_n : n = 1, \dots, N\}$ and corresponding camera pose parameters.
**Outputs:** A set of 3D Gaussian primitives $\{G_k\}$ aligned to selected voxels.

**Data Structures:**
*   **Voxel Grid:** Predefined in normalized 3D space $[-0.5, 0.5]^3$, centered at the origin, using the center-coordinate system. Dimensions are $H \times W \times D$. All views share this grid [S1, unofficial].
*   **3D Gaussian Primitive $G_k$:** Parameterized by:
    *   Mean vector $\mu_k \in \mathbb{R}^3$ (position).
    *   Covariance matrix $\Sigma_k \in \mathbb{R}^{3 \times 3}$, decomposed into scale vector $s_k \in \mathbb{R}^3$ and rotation quaternion $r_k \in \mathbb{R}^4$.
    *   Opacity $o_k \in [0, 1]$.
    *   Spherical harmonics coefficients $h_k \in \mathbb{R}^{3S}$ (view-dependent color, $S$ basis functions per channel) [S1, unofficial].

**Algorithm Steps:**

1.  **View-Specific Volumetric Feature Extraction:**
    *   Extract 2D feature maps from each input image $i_n$ using a shared feature extractor.
    *   Modulate each feature map using Adaptive Layer Normalization (AdaLN) with a Plücker embedding map $p_n$ derived from camera pose parameters [S1, unofficial].
    *   Backproject the modulated feature maps into the predefined voxel grid to produce view-specific feature volumes $v_n \in \mathbb{R}^{H \times W \times D \times C}$ for $n=1, \dots, N$ [S1, unofficial].

2.  **Multi-View Feature Aggregation and Compression:**
    *   Form an initial unified feature volume $v_0$ by concatenating the voxel-wise mean and variance of the view-specific volumes:
        $$ v_0 = \text{Mean}(v_1, \dots, v_N) \oplus \text{Var}(v_1, \dots, v_N) $$
        where $\oplus$ denotes channel-wise concatenation, and $\text{Mean}(\cdot)$ and $\text{Var}(\cdot)$ are voxel-wise operators across views [S1, unofficial].
    *   Compress $v_0$ using a 3D convolutional compressor $\mathcal{C}$:
        $$ v_{\text{comp}} = \mathcal{C}(v_0), \quad \mathcal{C}: \mathbb{R}^{H \times W \times D \times 2C} \rightarrow \mathbb{R}^{H \times W \times D \times \tilde{C}} $$
        where $\tilde{C} \ll 2C$ [S1, unofficial].
    *   Transform the compressed volume using a lightweight 3D U-Net variant $\mathcal{U}_{\text{light}}$ (with one encoding and one decoding block) to obtain the unified volumetric representation $\tilde{v}$:
        $$ \tilde{v} = \mathcal{U}_{\text{light}}(v_{\text{comp}}) $$
        $\tilde{v} \in \mathbb{R}^{H \times W \times D \times \tilde{C}}$ [S1, unofficial].

3.  **Coarse Stage (Primitive Prediction):**
    *   Decode features from $\tilde{v}$ into coarse Gaussian primitives. For each voxel $(h, w, d)$, use a fully-connected decoder $\mathcal{D}_{\text{crs}}$ to predict parameters $\{\mu_{h,w,d}^{\text{crs}}, s_{h,w,d}^{\text{crs}}, r_{h,w,d}^{\text{crs}}, o_{h,w,d}^{\text{crs}}, h_{h,w,d}^{\text{crs}}\}$ [S1, unofficial].
    *   **Position Prediction:** Predict a positional offset $\Delta_k \in [-r, r]^3$ for the $k$-th voxel, where $r > 0$ is the maximum displacement. Compute the final mean:
        $$ \mu_k = \text{voxel\_center}_k + \Delta_k $$
        This ensures the primitive remains within its voxel [S1, unofficial].

4.  **Voxel Selection Stage:**
    *   Predict voxel-wise confidence scores $\sigma \in [0, 1]^{H \times W \times D}$ using a fully-connected predictor $\mathcal{P}: \mathbb{R}^{\tilde{C}} \rightarrow \mathbb{R}$ and a Gumbel-Sigmoid operator [S1, unofficial].
    *   For each feature vector $\tilde{v}_{h,w,d}$, compute the confidence logit $\mathcal{P}(\tilde{v}_{h,w,d})$.
    *   Apply Gumbel-Sigmoid to obtain the score $\sigma_{h,w,d}$:
        $$ \sigma_{h,w,d} = \frac{1}{1 + \exp\left(-\frac{\mathcal{P}(\tilde{v}_{h,w,d}) + g_0 - g_1}{\tau}\right)} $$
        where $g_0, g_1 \sim \text{Gumbel}(0, 1)$ are i.i.d. samples, and $\tau$ is the temperature [S1, unofficial].
    *   **Opacity Refinement:** Multiply the coarse opacity by the confidence score to refine it:
        $$ \tilde{o}_{h,w,d}^{\text{crs}} = \sigma_{h,w,d} \cdot o_{h,w,d}^{\text{crs}} $$
        This maintains gradient propagation during training [S1, unofficial].
    *   **Selection:** Define the set of selected indices $I$ where confidence exceeds 0.5:
        $$ I = \{(h, w, d) : \sigma_{h,w,d} > 0.5\} $$
        The coarse primitives at these indices are termed "skeletal" Gaussian primitives [S1, unofficial].

5.  **Fine Stage:**
    *   Predict multiple fine-grained Gaussian primitives near each "skeletal" primitive at selected voxels $(h, w, d) \in I$.
    *   Use a selective feature refinement module to refine coarse global features by integrating fine local geometric features [S1, unofficial].
    *   *Note: The specific architecture of the fine stage decoder is not detailed in the provided sources.*

6.  **Training:**
    *   Use 2D supervision (rendering loss) on both coarse and fine renderings [S1, unofficial].
    *   The selection module is trained end-to-end solely through the rendering objective, without pseudo-labels or auxiliary supervision [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class VSSplat:
    def __init__(self, H, W, D, C, C_tilde, r_max):
        """
        Initialize VS-Splat components.
        H, W, D: Voxel grid dimensions.
        C: Input feature channels per view.
        C_tilde: Compressed feature channels.
        r_max: Maximum positional offset displacement.
        """
        self.H, self.W, self.D = H, W, D
        self.C = C
        self.C_tilde = C_tilde
        self.r_max = r_max
        
        # Feature Extractor (Shared)
        self.feature_extractor = nn.Conv2d(3, C, kernel_size=3, padding=1)
        
        # AdaLN Modulation
        self.adaln = AdaLNModule(C)
        
        # Compressor: 2C -> C_tilde
        self.compressor = nn.Conv3d(2 * C, C_tilde, kernel_size=3, padding=1)
        
        # Lightweight 3D U-Net Variant
        self.unet_light = Lightweight3DUNet(C_tilde, C_tilde)
        
        # Coarse Decoder
        self.coarse_decoder = nn.Linear(C_tilde, 3 + 3 + 4 + 1 + 3 * 3) # mu, s, r, o, h (S=3)
        
        # Confidence Predictor
        self.confidence_predictor = nn.Linear(C_tilde, 1)
        
        # Fine Stage Decoder (Placeholder for unspecified architecture)
        self.fine_decoder = FineStageDecoder(C_tilde)

    def forward(self, images, camera_poses):
        """
        images: Tensor of shape (N, 3, H_img, W_img)
        camera_poses: List of Plücker embeddings for N views
        """
        N = images.shape[0]
        view_specific_features = []
        
        # 1. View-Specific Volumetric Feature Extraction
        for n in range(N):
            # Extract 2D features
            f_2d = self.feature_extractor(images[n]) # (C, H_img, W_img)
            
            # Modulate with AdaLN using Plücker embedding
            f_modulated = self.adaln(f_2d, camera_poses[n])
            
            # Backproject to voxel grid (Simplified: assuming projection logic handles H,W,D)
            # In practice, this involves ray casting or differentiable rasterization
            # to fill the 3D volume. Here we assume a function `backproject` exists
            # or is implemented via standard differentiable rendering backprop.
            v_n = self.backproject(f_modulated, camera_poses[n], self.H, self.W, self.D)
            view_specific_features.append(v_n)
            
        # 2. Multi-View Feature Aggregation
        # v_n shape: (N, C, H, W, D)
        v_stack = torch.stack(view_specific_features, dim=0)
        
        # Voxel-wise mean and variance across views
        mean_feat = torch.mean(v_stack, dim=0) # (C, H, W, D)
        var_feat = torch.var(v_stack, dim=0, unbiased=False) # (C, H, W, D)
        
        # Concatenate along channel dimension
        v0 = torch.cat([mean_feat, var_feat], dim=0) # (2C, H, W, D)
        
        # Compress
        v_comp = self.compressor(v0) # (C_tilde, H, W, D)
        
        # Lightweight U-Net
        v_tilde = self.unet_light(v_comp) # (C_tilde, H, W, D)
        
        # 3. Coarse Stage
        # Reshape to (H*W*D, C_tilde)
        B, C_t, H, W, D = v_tilde.shape
        v_flat = v_tilde.permute(1, 2, 3, 4, 0).reshape(-1, C_tilde)
        
        # Decode coarse primitives
        coarse_params = self.coarse_decoder(v_flat)
        
        # Split parameters
        # mu: 3, s: 3, r: 4, o: 1, h: 9 (S=3)
        mu_offset = coarse_params[:, :3]
        s = coarse_params[:, 3:6]
        r = coarse_params[:, 6:10]
        o_raw = coarse_params[:, 10:11]
        h = coarse_params[:, 11:]
        
        # Compute voxel centers
        voxel_centers = self.get_voxel_centers() # (H*W*D, 3) in [-0.5, 0.5]^3
        
        # Apply offset constraint
        mu_offset = torch.clamp(mu_offset, -self.r_max, self.r_max)
        mu = voxel_centers + mu_offset
        
        # 4. Voxel Selection
        # Predict confidence logits
        conf_logits = self.confidence_predictor(v_flat).squeeze(-1) # (H*W*D,)
        
        # Gumbel-Sigmoid
        g0 = torch.rand_like(conf_logits)
        g1 = torch.rand_like(conf_logits)
        # Gumbel(0,1) sample: -log(-log(U))
        g0 = -torch.log(-torch.log(g0 + 1e-10) + 1e-10)
        g1 = -torch.log(-torch.log(g1 + 1e-10) + 1e-10)
        
        tau = 0.1 # Temperature parameter
        sigma = 1.0 / (1.0 + torch.exp(-(conf_logits + g0 - g1) / tau))
        
        # Refine opacity
        o_refined = sigma * torch.sigmoid(o_raw) # Assuming raw opacity is logit or needs sigmoid
        
        # Select voxels
        threshold = 0.5
        selected_mask = sigma > threshold # (H*W*D,) boolean
        
        # 5. Fine Stage
        # Only process selected voxels
        selected_indices = torch.nonzero(selected_mask, as_tuple=False).squeeze()
        
        if selected_indices.numel() > 0:
            v_selected = v_flat[selected_indices]
            fine_params = self.fine_decoder(v_selected)
            # Decode fine primitives from fine_params...
            # Return both coarse and fine primitives
            return mu, s, r, o_refined, h, fine_params
        else:
            return mu, s, r, o_refined, h, None

    def get_voxel_centers(self):
        # Generate grid centers in [-0.5, 0.5]^3
        x = torch.linspace(-0.5 + 1/(2*self.H), 0.5 - 1/(2*self.H), self.H)
        y = torch.linspace(-0.5 + 1/(2*self.W), 0.5 - 1/(2*self.W), self.W)
        z = torch.linspace(-0.5 + 1/(2*self.D), 0.5 - 1/(2*self.D), self.D)
        xx, yy, zz = torch.meshgrid(x, y, z, indexing='ij')
        centers = torch.stack([xx, yy, zz], dim=-1).reshape(-1, 3)
        return centers

    def backproject(self, feature_map, pose, H, W, D):
        # Placeholder for differentiable backprojection logic
        # Returns tensor of shape (C, H, W, D)
        pass

class AdaLNModule(nn.Module):
    def __init__(self, C):
        super().__init__()
        self.norm = nn.LayerNorm(C)
        # Projection from Plücker embedding to AdaLN shift/scale
        self.proj = nn.Linear(6, 2 * C) # Assuming Plücker dim 6

    def forward(self, x, plucker):
        x = self.norm(x)
        shift_scale = self.proj(plucker)
        shift = shift_scale[:, :C]
        scale = shift_scale[:, C:]
        return x * (1 + scale) + shift

class Lightweight3DUNet(nn.Module):
    def __init__(self, in_ch, out_ch):
        super().__init__()
        self.encoder = nn.Conv3d(in_ch, out_ch, 3, padding=1)
        self.decoder = nn.Conv3d(out_ch, out_ch, 3, padding=1)
        
    def forward(self, x):
        x = self.encoder(x)
        x = self.decoder(x)
        return x

class FineStageDecoder(nn.Module):
    def __init__(self, C_tilde):
        super().__init__()
        # Placeholder for unspecified fine stage architecture
        self.linear = nn.Linear(C_tilde, 100) # Arbitrary placeholder
        
    def forward(self, x):
        return self.linear(x)
```

**Parameters:**
*   **Voxel Grid:** $[-0.5, 0.5]^3$, centered at origin [S1, unofficial].
*   **Temperature $\tau$:** Controls Gumbel-Sigmoid stochasticity. Low $\tau$ pushes scores to 0 or 1 [S1, unofficial].
*   **Selection Threshold:** 0.5. No tuning required [S1, unofficial].
*   **Max Displacement $r$:** Hyperparameter for positional offset bounds [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The Gumbel-Sigmoid approximation requires careful handling of numerical stability in the exponential term, especially with low $\tau$ [S1, unofficial].
*   **Boundaries:** Positional offsets are clamped to $[-r, r]$ to ensure primitives stay within voxels [S1, unofficial].
*   **Degenerate Input:** If no voxels exceed the 0.5 threshold, the fine stage produces no primitives [S1, unofficial].

## Subtopics

### LaRa
A feed-forward GS method that predicts Gaussians from features. VS-Splat differs by selecting voxels before fine generation, whereas LaRa selects existing primitives for refinement. [LaRa](52-lara.md)

### GenDen
Augments LaRa with a densification module using positional gradients. VS-Splat uses voxel-level selection without pseudo-labels, unlike GenDen which uses straight-through estimators. [GenDen](53-genden.md)

### SplatWeaver
Uses score-based selection with pseudo-labels. VS-Splat is trained solely via rendering objective. [SplatWeaver](54-splatweaver.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Uses a lightweight 3D U-Net variant and feature compression ($\tilde{C} \ll 2C$) to reduce memory and computational cost compared to transformer-based approaches [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** End-to-end learnable voxel selection without auxiliary supervision; coarse representation guides fine primitive placement; adaptive density control [S1, unofficial].
*   **Limitations:** Relies on a predefined voxel grid; fine stage architecture details are not specified [S1, unofficial].
*   **When to use:** Sparse-view 3D object reconstruction where adaptive density and feed-forward efficiency are prioritized [S1, unofficial].

## Background
VS-Splat was proposed in the paper "VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views" [S1, unofficial]. It builds on the background of 3D Gaussian Splatting [7] and voxel-aligned feed-forward models [11, 12], addressing limitations in selection strategies used by prior works like LaRa and GenDen [S1, unofficial].

## Sources
[S1] VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views, https://doi.org/10.48550/arxiv.2609.12343, fetched 2026-10-26 (unofficial)
