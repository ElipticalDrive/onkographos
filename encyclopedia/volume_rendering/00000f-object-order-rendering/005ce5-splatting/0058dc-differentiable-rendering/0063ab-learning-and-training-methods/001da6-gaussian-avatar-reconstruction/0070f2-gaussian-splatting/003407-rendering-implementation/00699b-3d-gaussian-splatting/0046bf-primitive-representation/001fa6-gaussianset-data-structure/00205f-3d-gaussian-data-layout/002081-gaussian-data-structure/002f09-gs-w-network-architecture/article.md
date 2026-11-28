# GS-W Network Architecture

Also called: Gaussian in the Wild model parameters

## Overview
The GS-W Network Architecture is the component of the Gaussian in the Wild (GS-W) method responsible for predicting per-Gaussian appearance attributes and visibility from a set of reference images [S1]. It solves the problem of adapting 3D Gaussian Splatting to unconstrained image collections by replacing static color attributes with learnable features that are dynamically sampled based on camera viewpoint [S1]. It sets itself apart from standard 3DGS by introducing a U-Net encoder-decoder structure to extract image features and a fusion network to combine these with intrinsic Gaussian attributes, enabling view-dependent appearance modeling without explicit spherical harmonics [S1].

## How it works
**Inputs:**
- A set of $K$ reference images (or their extracted feature maps) [S1].
- For each Gaussian point $i$:
  - Position $X_i \in \mathbb{R}^3$ [S1].
  - Intrinsic feature $s_{fi} \in \mathbb{R}^{48}$ [S1].
  - Dynamic appearance feature $d_{fi}$ [S1].
  - Sampling coordinates $sc_{ki} \in \mathbb{R}^2$ for $k=1 \dots K$ [S1].

**Outputs:**
- Predicted color $C_i \in \mathbb{R}^3$ [S1].
- Visibility map $V$ (spatial tensor) [S1].

**Data Structures:**
1. **U-Net Encoder:** Uses the first 8 subconvolution modules of a pre-trained ResNet-18 [S1].
2. **U-Net Decoder 1 (Feature Maps):** 4 up-sampling convolution modules. Outputs $K+1$ feature maps. Each feature map has 16 channels [S1].
3. **U-Net Decoder 2 (Visibility):** 3 up-sampling convolution modules. Outputs the visibility map. No skip connections from the encoder [S1].
4. **Fusion Network ($M_f$):** Two MLP modules. $M_{f1}$ has 3 hidden layers with units $[128, 96, 64]$. $M_{f2}$ has 2 hidden layers with units $[48, 48]$ [S1].
5. **Color MLP ($M_c$):** 1 hidden layer with 48 units [S1].
6. **Sampling Coordinates:** $K$ learnable 2D coordinates per Gaussian, initialized via linear transformation of position $X_i$ [S1].

**Parameters:**
- **Positional Encoding:** 10 frequencies used for encoding position $X_i$ [S1].
- **Batch Normalization & Activation:** Applied in both U-Net decoders [S1].
- **Dropout:** 0.1 probability applied to $d_{fi}$ during training [S1].
- **Initialization:** Xavier initialization for MLP weights [S1].

**Numbered Steps:**
1. **Feature Extraction:** Pass the reference images through the U-Net encoder (ResNet-18 first 8 modules) to extract image features [S1].
2. **Feature Map Generation:** Pass encoder features through U-Net Decoder 1 (4 up-sampling convs, batch norm, ReLU, transpose conv, 16 channels) to generate $K+1$ feature maps [S1].
3. **Visibility Prediction:** Pass encoder features through U-Net Decoder 2 (3 up-sampling convs, batch norm, ReLU, transpose conv, 16 channels) to generate the visibility map. Skip connections from the encoder to this decoder are omitted to reduce computation [S1].
4. **Sampling Coordinate Initialization (Pre-training):** For each Gaussian $i$, initialize $K$ sampling coordinates $sc_{ki}$ by applying a linear transformation to its 3D position $X_i$ [S1].
5. **Feature Sampling:** For each Gaussian $i$ and each of the $K$ feature maps, sample the feature value at the corresponding coordinate $sc_{ki}$ from the generated feature maps. This yields $K$ sampled features [S1].
6. **Fusion:** Concatenate the sampled features, the intrinsic feature $s_{fi}$, the positional encoding of $X_i$, and the dynamic appearance feature $d_{fi}$. Apply 0.1 dropout to $d_{fi}$ during training. Pass this concatenated vector through Fusion Network $M_f$ (MLP 1: $[128, 96, 64]$; MLP 2: $[48, 48]$) to produce a fused appearance feature [S1].
7. **Color Prediction:** Pass the fused appearance feature through Color MLP $M_c$ (1 hidden layer, 48 units) to predict the final color $C_i$ [S1].
8. **Training Updates:** Update U-Net weights with learning rate decaying from $2 \times 10^{-3}$ to $2 \times 10^{-5}$ [S1]. Update MLP weights with fixed learning rate $5 \times 10^{-4}$ [S1]. Update Gaussian positions with learning rate decaying from $1.6 \times 10^{-4}$ to $1.6 \times 10^{-7}$ [S1]. Densify Gaussians between iterations 500 and 15,000 with a gradient threshold of $4 \times 10^{-4}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GS_W_Network:
    def __init__(self, K, encoder_channels=256, d_fi_dim=64):
        self.K = K
        # U-Net Encoder: Pre-trained ResNet-18 first 8 subconvolution modules
        self.encoder = self._load_pretrained_resnet18_encoder()
        
        # U-Net Decoder 1: Generate K+1 feature maps
        # 4 up-sampling convolution modules, batch norm, ReLU, transpose conv, 16 channels
        self.decoder_features = nn.Sequential(
            nn.ConvTranspose2d(encoder_channels, 16, kernel_size=4, stride=2, padding=1), nn.BatchNorm2d(16), nn.ReLU(),
            nn.ConvTranspose2d(16, 16, kernel_size=4, stride=2, padding=1), nn.BatchNorm2d(16), nn.ReLU(),
            nn.ConvTranspose2d(16, 16, kernel_size=4, stride=2, padding=1), nn.BatchNorm2d(16), nn.ReLU(),
            nn.ConvTranspose2d(16, 16 * (K + 1), kernel_size=4, stride=2, padding=1), nn.BatchNorm2d(16 * (K + 1)), nn.ReLU()
        )
        
        # U-Net Decoder 2: Generate visibility map
        # 3 up-sampling convolution modules, NO skip connections from encoder
        self.decoder_visibility = nn.Sequential(
            nn.ConvTranspose2d(encoder_channels, 16, kernel_size=4, stride=2, padding=1), nn.BatchNorm2d(16), nn.ReLU(),
            nn.ConvTranspose2d(16, 16, kernel_size=4, stride=2, padding=1), nn.BatchNorm2d(16), nn.ReLU(),
            nn.ConvTranspose2d(16, 1, kernel_size=4, stride=2, padding=1)
        )
        
        # Fusion Network Mf
        # Input: K sampled features (16*K) + s_fi (48) + pos_enc (20) + d_fi (d_fi_dim)
        fusion_input_dim = 16 * K + 48 + 20 + d_fi_dim
        self.mf1 = nn.Sequential(nn.Linear(fusion_input_dim, 128), nn.ReLU(),
                                 nn.Linear(128, 96), nn.ReLU(),
                                 nn.Linear(96, 64), nn.ReLU())
        self.mf2 = nn.Sequential(nn.Linear(64, 48), nn.ReLU(),
                                 nn.Linear(48, 48))
        
        # Color MLP Mc
        self.mc = nn.Sequential(nn.Linear(48, 48), nn.ReLU(),
                                nn.Linear(48, 3))
        
        # Xavier initialization for MLPs
        self._init_weights()
        
        # Sampling coordinates initialization
        self.sc_linear = nn.Linear(3, 2 * K)

    def _load_pretrained_resnet18_encoder(self):
        # Returns the first 8 subconvolution modules of a pre-trained ResNet-18
        # Implementation depends on the specific framework; here we assume it is loaded externally.
        pass

    def _init_weights(self):
        for m in [self.mf1, self.mf2, self.mc]:
            for layer in m:
                if isinstance(layer, nn.Linear):
                    nn.init.xavier_uniform_(layer.weight)

    def forward(self, images, gaussians):
        N = gaussians['positions'].shape[0]
        
        # Step 1: Feature Extraction
        encoder_features = self.encoder(images)
        
        # Step 2: Feature Map Generation
        feature_maps_tensor = self.decoder_features(encoder_features) # [B, 16*(K+1), H', W']
        
        # Step 3: Visibility Prediction
        visibility_tensor = self.decoder_visibility(encoder_features) # [B, 1, H'', W'']
        
        # Step 4 & 5: Feature Sampling
        # Reshape feature_maps_tensor to [B, K+1, 16, H', W']
        feature_maps_reshaped = feature_maps_tensor.view(feature_maps_tensor.shape[0], -1, 16, feature_maps_tensor.shape[2], feature_maps_tensor.shape[3])
        
        sampled_features = []
        # Normalize sampling coordinates to [-1, 1] for grid_sample
        coords = gaussians['sc'] # [N, K, 2]
        coords_norm = 2.0 * coords - 1.0 # Assuming coords are in [0, 1]
        coords_norm = coords_norm.unsqueeze(1).unsqueeze(1) # [N, 1, 1, K, 2]
        
        for k in range(self.K):
            fmap_k = feature_maps_reshaped[:, k, :, :, :] # [B, 16, H', W']
            # Expand to match N gaussians
            fmap_k_expanded = fmap_k.unsqueeze(0).expand(N, -1, -1, -1, -1) if fmap_k.shape[0] == 1 else fmap_k.unsqueeze(0)
            # grid_sample expects grid [B, H_out, W_out, 2]
            # We sample at single points, so H_out=1, W_out=1
            grid = coords_norm[:, :, :, k, :] # [N, 1, 1, 2]
            sampled = F.grid_sample(fmap_k_expanded, grid, align_corners=True) # [N, 16, 1, 1]
            sampled_features.append(sampled.squeeze(-1).squeeze(-1)) # [N, 16]
        
        sampled_features = torch.stack(sampled_features, dim=1) # [N, K, 16]
        
        # Step 6: Fusion
        pos_enc = self._positional_encoding(gaussians['positions'], num_frequencies=10) # [N, 20]
        d_fi = gaussians['d_fi'] # [N, d_fi_dim]
        d_fi_dropped = F.dropout(d_fi, p=0.1, training=self.training)
        
        fusion_input = torch.cat([sampled_features.view(N, -1), gaussians['s_fi'], pos_enc, d_fi_dropped], dim=1)
        fused_feature = self.mf1(fusion_input)
        fused_feature = self.mf2(fused_feature) # [N, 48]
        
        # Step 7: Color Prediction
        colors = self.mc(fused_feature) # [N, 3]
        
        # Step 8: Visibility
        visibility_map = visibility_tensor # [B, 1, H'', W'']
        
        return colors, visibility_map

    def _positional_encoding(self, positions, num_frequencies):
        # Standard positional encoding
        # positions: [N, 3]
        # Output: [N, 2 * num_frequencies]
        freq_bands = torch.cat([positions * (10000 ** (2 * (i // 2) / 3)) for i in range(num_frequencies)], dim=1)
        return torch.cat([torch.sin(freq_bands), torch.cos(freq_bands)], dim=1)
```

## Subtopics

### [Wave Parameter Estimation Network](13001-wave-parameter-estimation-network.md)
The Wave Parameter Estimation Network (WPEN) estimates the constituent wave parameters (amplitude, angular frequency, and propagation direction) of a dynamic ocean from a time sequence of reconstructed geometric data, specifically surface normals and microfacet roughness. It distinguishes itself from direct regression methods by employing a differentiable ocean model as a physics-based loss function, allowing the network to learn by minimizing the geometric discrepancy between predicted and input ocean states. This approach enables self-supervised training where the network iteratively optimizes wave parameters to match real-world geometry reconstructed by preceding stages.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Stores $K+1$ feature maps (16 channels each) and the visibility map. Gaussian attributes include $s_{fi}$ (48 dims), $d_{fi}$, and $sc$ ($K \times 2$ dims) [S1].
- **Performance:** On Brandenburg, Sacre, and Trevi scenes, GS-W achieves PSNR of 24.70, SSIM of 0.8655, and LPIPS of 0.1242, with 50.7 FPS and 122.7 MB size [S1]. This outperforms global feature (PSNR 22.43) and K-planes-hybrid (PSNR 22.40) methods [S1].
- **Strengths:** Captures local highlights and avoids artifacts from transient objects [S1]. Adjusts scene appearance in a manner aligned with human perception by weighting features from reference images [S1].
- **Limitations:** Not given in the sources.
- **When to use:** Unconstrained image collections where view-dependent appearance is important and reference images are available [S1].
- **When not to use:** Not given in the sources.

## Background
The GS-W Network Architecture was introduced in the "Gaussian in the Wild" paper, which extends 3D Gaussian Splatting to unconstrained image collections [S1]. It was developed to handle the challenges of dynamic appearance and transient objects in real-world image sets, which standard 3DGS struggles with due to its static color attributes [S1]. The method builds on the Gaussian Data Structure and integrates a neural network to predict appearance features dynamically [S1].

## Sources
[S1] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
