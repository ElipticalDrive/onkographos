# SegAnySplatting Feature Optimization

Also called: Scale-Aware Semantic Features, SegAnyGS

## Overview
SegAnySplatting Feature Optimization optimizes semantic features for 3D Gaussians by incorporating scale-awareness through a learned gate and feature smoothing [S1, unofficial]. It differs from [FaCT-GS](10199-fact-gs.md) by focusing on distilling semantic information (e.g., from CLIP or LSeg) into Gaussian attributes with scale-adaptive gating, rather than compact factorized color representations [S1, unofficial].

## How it works
**Inputs and Outputs:** The method takes a frozen pre-trained 3D Gaussian Splatting model and a training dataset containing semantic masks or features as input, and outputs an optimized semantic feature tensor attached to each Gaussian [S1, unofficial].

**Data Structures:** The semantic features are stored as a `torch.nn.Parameter` of shape `(n_gaussians, n_feature_dims)` and initialized with `torch.randn(...) * 1e-2` [S1, unofficial]. The scale gate is implemented as an `nn.Sequential` module containing `nn.Linear(1, n_feature_dims)` followed by `nn.Sigmoid()` [S1, unofficial]. The background color is stored as a `torch.nn.Parameter` of shape `(n_feature_dims,)`, initialized to zeros, and marked as non-trainable [S1, unofficial].

**Parameters:** The method uses `n_feature_dims` (int, default 32) for the number of semantic feature dimensions per Gaussian [S1, unofficial]. It uses `scale_aware_dim` (int) to control scale awareness, where values `<=0` or `>=32` trigger the adaptive scale gate [S1, unofficial]. It uses `smooth_K` (int, default 16) for the feature smoothing kernel size [S1, unofficial]. It uses `smooth_dropout` (float, default 0.5) for the feature smoothing dropout rate [S1, unofficial]. It uses `rfn` (float, default 1.0) as the feature normalization factor [S1, unofficial]. It uses `down_sample_factor` (int) to reduce the resolution of the rendered feature map and accelerate distillation [S1, unofficial].

**Steps:**
1. Load the pre-trained Gaussian model and freeze its geometric and color parameters [S1, unofficial].
2. Gather all scale values from the training dataset to compute `upper_bound_scale` [S1, unofficial].
3. If `scale_aware_dim` is not in the range $[1, 31]$, initialize a quantile transformer (`q_trans`) with a uniform output distribution to normalize the scale values [S1, unofficial].
4. If `scale_aware_dim` is in the range $[1, 31]$, initialize a fixed scale gate tensor `fixed_scale_gate` of shape `(scale_aware_dim + 1, 32)` where row $i$ contains 1s in the first $32 - \text{scale\_aware\_dim} + i$ positions and 0s elsewhere [S1, unofficial].
5. Preprocess masks by sorting them by scale in descending order and sampling indices for the largest, smallest, and random intermediate scales [S1, unofficial].
6. Render semantic features from the Gaussians, downsample the feature map using `down_sample_factor`, compute the distillation loss against ground-truth semantic features, apply feature smoothing with kernel size `smooth_K` and dropout `smooth_dropout`, and update `Semantic Features` and `Scale Gate` via gradient descent [S1, unofficial].
7. After distillation, visualize the rendered feature map using the provided viewer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SegAnySplattingFeatureOptimization:
    def __init__(self, n_gaussians, n_feature_dims=32, scale_aware_dim=0, 
                 smooth_K=16, smooth_dropout=0.5, rfn=1.0):
        self.n_gaussians = n_gaussians
        self.n_feature_dims = n_feature_dims
        self.scale_aware_dim = scale_aware_dim
        self.smooth_K = smooth_K
        self.smooth_dropout = smooth_dropout
        self.rfn = rfn
        
        # Semantic Features: (n_gaussians, n_feature_dims)
        self.semantic_features = nn.Parameter(torch.randn(n_gaussians, n_feature_dims) * 1e-2)
        # Background Color: (n_feature_dims,), not trainable
        self.background_color = nn.Parameter(torch.zeros(n_feature_dims), requires_grad=False)
        
        # Scale Gate initialization
        if not (1 <= scale_aware_dim <= 31):
            self.scale_gate = nn.Sequential(nn.Linear(1, n_feature_dims), nn.Sigmoid())
            self.fixed_scale_gate = None
        else:
            self.fixed_scale_gate = torch.zeros(scale_aware_dim + 1, 32)
            for i in range(scale_aware_dim + 1):
                self.fixed_scale_gate[i, :32 - scale_aware_dim + i] = 1.0
            self.scale_gate = None
            
        self.q_trans = None

    def _quantile_normalize(self, scales):
        # [G] Standard quantile normalization mapping to uniform distribution
        sorted_scales, _ = torch.sort(scales)
        ranks = torch.searchsorted(sorted_scales, scales)
        return ranks / len(scales)

    def preprocess_scales(self, scales):
        upper_bound_scale = torch.max(scales)
        if self.scale_gate is not None:
            self.q_trans = self._quantile_normalize(scales)
        return upper_bound_scale

    def preprocess_masks(self, masks, scales):
        sorted_indices = torch.argsort(scales, descending=True)
        sorted_masks = [masks[i] for i in sorted_indices]
        n = len(sorted_masks)
        indices = [0, n - 1]
        if n > 2:
            mid_indices = torch.randint(1, n - 1, (max(1, n // 4),))
            indices.extend(mid_indices.tolist())
        return [sorted_masks[i] for i in indices]

    def smooth_features(self, features):
        # [G] Apply spatial smoothing via average pooling and dropout
        if self.smooth_dropout > 0:
            features = F.dropout(features, p=self.smooth_dropout, training=True)
        k = self.smooth_K
        if features.dim() == 2:
            features = F.avg_pool2d(features.unsqueeze(0).unsqueeze(0), kernel_size=k, stride=1).squeeze()
        return features

    def render_and_optimize(self, gaussians, semantic_targets, optimizer):
        optimizer.zero_grad()
        # [G] Standard 3DGS rasterization accumulating semantic features
        rendered_features = self._render_features(gaussians)
        # Downsample for speed
        rendered_features = F.avg_pool2d(rendered_features, kernel_size=2, stride=2)
        # Loss
        loss = F.mse_loss(rendered_features, semantic_targets)
        # Smoothing
        smoothed = self.smooth_features(rendered_features)
        # [G] Loss typically computed on smoothed or original features; here on original per standard distillation
        loss.backward()
        optimizer.step()

    def _render_features(self, gaussians):
        # [G] Standard 3DGS projection and accumulation
        scales = gaussians.scales  # (n_gaussians, 3)
        scalar_scales = torch.max(scales, dim=1).values
        if self.scale_gate is not None:
            gate = self.scale_gate(scalar_scales.unsqueeze(1))
            features = self.semantic_features * gate
        else:
            # Map scalar scale to fixed gate row index
            scale_norm = scalar_scales / (torch.max(scalar_scales) + 1e-8)
            row_idx = torch.clamp((scale_norm * self.scale_aware_dim).long(), 0, self.scale_aware_dim)
            gate = self.fixed_scale_gate[row_idx]
            features = self.semantic_features * gate
        # [G] Rasterize features to image space
        return features  # Simplified return for pseudocode structure
```

**Edge Cases and Pitfalls:** If `scale_aware_dim` falls outside $[1, 31]$, the adaptive gate is automatically selected [S1, unofficial]. The `down_sample_factor` must be $\ge 1$ to prevent invalid tensor shapes during distillation [S1, unofficial]. Degenerate scale values (e.g., all zeros) should be handled by the quantile transformer or clamped to a minimum positive value [G]. Precision issues with the random initialization are mitigated by the $10^{-2}$ scaling factor [S1, unofficial].

## Operating characteristics
Time and memory complexity: Not given in the sources. Scaling: The method uses `down_sample_factor` to reduce the resolution of the rendered feature map and accelerate distillation [S1, unofficial]. Strengths, limitations, and usage recommendations: Not given in the sources.

## Background
The method originates from the `gaussian-splatting-lightning` repository [S1, unofficial]. It extends 3D Gaussian Splatting to support semantic feature optimization, building on appearance modeling and visibility map concepts from earlier sections [S1, unofficial].

## Sources
[S1] yzslab/gaussian-splatting-lightning, https://github.com/yzslab/gaussian-splatting-lightning, fetched 2026-10-27 (unofficial)
