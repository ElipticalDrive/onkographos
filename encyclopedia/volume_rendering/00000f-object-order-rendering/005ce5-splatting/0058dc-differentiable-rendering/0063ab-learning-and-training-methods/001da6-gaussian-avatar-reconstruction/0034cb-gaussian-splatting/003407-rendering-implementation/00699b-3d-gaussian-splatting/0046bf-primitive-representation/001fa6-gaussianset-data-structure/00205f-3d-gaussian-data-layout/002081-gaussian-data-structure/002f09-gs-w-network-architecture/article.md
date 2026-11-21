# GS-W Network Architecture

Also called: Gaussian in the Wild model parameters

## Overview
The GS-W Network Architecture is the component of the Gaussian in the Wild (GS-W) method responsible for predicting per-Gaussian appearance attributes and visibility from a set of reference images. It solves the problem of adapting 3D Gaussian Splatting to unconstrained image collections by replacing static color attributes with learnable features that are dynamically sampled based on camera viewpoint. It sets itself apart from standard 3DGS by introducing a U-Net encoder-decoder structure to extract image features and a fusion network to combine these with intrinsic Gaussian attributes, enabling view-dependent appearance modeling without explicit spherical harmonics [S1].

## How it works
The architecture consists of three main parts: a U-Net for feature extraction, a Fusion Network ($M_f$) for combining features, and a Color MLP ($M_c$) for final color prediction. It also manages sampling coordinates for dynamic feature retrieval.

**Inputs:**
- A set of $K$ reference images (or their extracted feature maps).
- For each Gaussian point $i$:
  - Position $X_i \in \mathbb{R}^3$.
  - Intrinsic feature $s_{fi} \in \mathbb{R}^{48}$ [S1].
  - Dynamic appearance feature $d_{fi}$ (from previous training steps or initialization) [S1].
  - Sampling coordinates $sc_{ki} \in \mathbb{R}^2$ for $k=1 \dots K$ [S1].

**Outputs:**
- Predicted color $C_i \in \mathbb{R}^3$.
- Visibility map $V_i$ (scalar or mask) [S1].
- Updated intrinsic feature $s_{fi}$ (implicit in the network's forward pass if treated as a latent, but typically $s_{fi}$ is an attribute stored per Gaussian) [S1].

**Data Structures:**
1.  **U-Net Encoder:** Uses the first 8 subconvolution modules of a pre-trained ResNet-18 [S1].
2.  **U-Net Decoder 1 (Feature Maps):** 4 up-sampling convolution modules. Outputs $K+1$ feature maps. Each feature map has 16 channels [S1].
3.  **U-Net Decoder 2 (Visibility):** 3 up-sampling convolution modules. Outputs the visibility map. No skip connections from the encoder [S1].
4.  **Fusion Network ($M_f$):** Two MLP modules.
    -   $M_{f1}$: 3 hidden layers with units $[128, 96, 64]$ [S1].
    -   $M_{f2}$: 2 hidden layers with units $[48, 48]$ [S1].
5.  **Color MLP ($M_c$):** 1 hidden layer with 48 units [S1].
6.  **Sampling Coordinates:** $K$ learnable 2D coordinates per Gaussian, initialized via linear transformation of position $X_i$ [S1].

**Parameters:**
-   **Positional Encoding:** 10 frequencies used for encoding position $X_i$ [S1].
-   **Batch Normalization & Activation:** Applied in both U-Net decoders [S1].
-   **Dropout:** 0.1 probability applied to $d_{fi}$ during training [S1].
-   **Initialization:** Xavier initialization for MLP weights [S1].

**Numbered Steps:**

1.  **Feature Extraction:** Pass the reference images through the U-Net encoder (ResNet-18 first 8 modules) to extract image features [S1].
2.  **Feature Map Generation:** Pass encoder features through U-Net Decoder 1 (4 up-sampling convs, batch norm, ReLU, transpose conv, 16 channels) to generate $K+1$ feature maps [S1].
3.  **Visibility Prediction:** Pass encoder features through U-Net Decoder 2 (3 up-sampling convs, batch norm, ReLU, transpose conv, 16 channels) to generate the visibility map. Note: Skip connections from encoder to this decoder are omitted to reduce computation [S1].
4.  **Sampling Coordinate Initialization (Pre-training):** For each Gaussian $i$, initialize $K$ sampling coordinates $sc_{ki}$ by applying a linear transformation to its 3D position $X_i$ [S1].
5.  **Feature Sampling:** For each Gaussian $i$ and each of the $K$ feature maps, sample the feature value at the corresponding coordinate $sc_{ki}$ from the generated feature maps. This yields $K$ sampled features [S1].
6.  **Fusion:** Concatenate the sampled features, the intrinsic feature $s_{fi}$, and the positional encoding of $X_i$. Pass this concatenated vector through Fusion Network $M_f$ (MLP 1: $[128, 96, 64]$; MLP 2: $[48, 48]$) to produce a fused appearance feature [S1].
7.  **Color Prediction:** Pass the fused appearance feature through Color MLP $M_c$ (1 hidden layer, 48 units) to predict the final color $C_i$ [S1].
8.  **Training Updates:**
    -   Update U-Net weights with learning rate decaying from $2 \times 10^{-3}$ to $2 \times 10^{-5}$ [S1].
    -   Update MLP weights with fixed learning rate $5 \times 10^{-4}$ [S1].
    -   Update Gaussian positions with learning rate decaying from $1.6 \times 10^{-4}$ to $1.6 \times 10^{-7}$ [S1].
    -   Densify Gaussians between iterations 500 and 15,000 with a gradient threshold of $4 \times 10^{-4}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GS_W_Network:
    def __init__(self, K, num_gaussians):
        """
        Initialize the GS-W network components.
        :param K: Number of reference feature maps to sample from.
        :param num_gaussians: Number of Gaussian points.
        """
        self.K = K
        
        # U-Net Encoder: Pre-trained ResNet-18 first 8 subconvolution modules
        # Note: In implementation, load ResNet-18 and slice the first 8 conv blocks
        self.encoder = self._load_resnet18_encoder()
        
        # U-Net Decoder 1: Generate K+1 feature maps
        # 4 up-sampling convolution modules, batch norm, ReLU, transpose conv, 16 channels
        self.decoder_features = nn.Sequential(
            # Up-sampling conv 1
            nn.ConvTranspose2d(256, 16, kernel_size=4, stride=2, padding=1), # Example channel assumption
            nn.BatchNorm2d(16),
            nn.ReLU(),
            # Up-sampling conv 2
            nn.ConvTranspose2d(16, 16, kernel_size=4, stride=2, padding=1),
            nn.BatchNorm2d(16),
            nn.ReLU(),
            # Up-sampling conv 3
            nn.ConvTranspose2d(16, 16, kernel_size=4, stride=2, padding=1),
            nn.BatchNorm2d(16),
            nn.ReLU(),
            # Up-sampling conv 4
            nn.ConvTranspose2d(16, 16, kernel_size=4, stride=2, padding=1),
            nn.BatchNorm2d(16),
            nn.ReLU()
        )
        
        # U-Net Decoder 2: Generate visibility map
        # 3 up-sampling convolution modules, NO skip connections from encoder
        self.decoder_visibility = nn.Sequential(
            # Up-sampling conv 1
            nn.ConvTranspose2d(256, 16, kernel_size=4, stride=2, padding=1),
            nn.BatchNorm2d(16),
            nn.ReLU(),
            # Up-sampling conv 2
            nn.ConvTranspose2d(16, 16, kernel_size=4, stride=2, padding=1),
            nn.BatchNorm2d(16),
            nn.ReLU(),
            # Up-sampling conv 3
            nn.ConvTranspose2d(16, 16, kernel_size=4, stride=2, padding=1),
            nn.BatchNorm2d(16),
            nn.ReLU()
        )
        
        # Fusion Network Mf
        # MLP 1: [128, 96, 64]
        self.mf1 = nn.Sequential(
            nn.Linear(128, 96), nn.ReLU(),
            nn.Linear(96, 64), nn.ReLU()
        )
        # MLP 2: [48, 48]
        self.mf2 = nn.Sequential(
            nn.Linear(64, 48), nn.ReLU(),
            nn.Linear(48, 48)
        )
        
        # Color MLP Mc
        # 1 hidden layer with 48 units
        self.mc = nn.Sequential(
            nn.Linear(48, 48), nn.ReLU(),
            nn.Linear(48, 3) # Output color RGB
        )
        
        # Xavier initialization for MLPs
        self._init_weights()
        
        # Sampling coordinates initialization
        # Linear transformation of position Xi to K sampling coordinates
        # sc_i = W * Xi + b
        self.sc_linear = nn.Linear(3, 2 * K)
        self._init_sampling_coords()

    def _load_resnet18_encoder(self):
        # Placeholder for loading pre-trained ResNet-18 and extracting first 8 modules
        # This is a structural representation. Actual implementation requires PyTorch torchvision models.
        pass

    def _init_weights(self):
        for m in [self.mf1, self.mf2, self.mc]:
            for layer in m:
                if isinstance(layer, nn.Linear):
                    nn.init.xavier_uniform_(layer.weight)

    def _init_sampling_coords(self):
        # Initialize sampling coordinates based on position
        # This is done per Gaussian during initialization, not as a single linear layer for all
        pass

    def forward(self, images, gaussians):
        """
        Forward pass for GS-W network.
        :param images: Tensor of reference images [B, C, H, W]
        :param gaussians: Dict containing Gaussian attributes
            - positions: [N, 3]
            - s_fi: [N, 48] intrinsic features
            - d_fi: [N, D] dynamic appearance features
            - sc: [N, K, 2] sampling coordinates
        :return: colors [N, 3], visibility [N, 1]
        """
        N = gaussians['positions'].shape[0]
        
        # Step 1: Feature Extraction
        encoder_features = self.encoder(images)
        
        # Step 2: Feature Map Generation
        # Output: [B, 16, H', W'] for each of K+1 maps? 
        # The source says "K+1 feature maps" are generated. 
        # Assuming the decoder outputs a tensor that can be split or indexed.
        # Let's assume the decoder outputs [B, 16*(K+1), H', W'] or similar structure.
        # For simplicity, we assume the decoder produces a tensor of shape [B, 16, H', W'] 
        # and we replicate or split it K+1 times, or the decoder has K+1 heads.
        # Given "number of channels for each feature map fixed at 16", 
        # and "K+1 feature maps", it's likely the output is [B, 16, H', W'] for each map.
        # We will assume the decoder outputs a tensor of shape [B, 16, H', W'] 
        # and we use it for all K+1 maps (or the architecture implies K+1 parallel decoders).
        # However, standard U-Net decoders output one tensor. 
        # Let's assume the decoder output is used to sample K+1 maps by some mechanism, 
        # or the decoder is replicated K+1 times. 
        # Given the text "decoder for generating K+1 feature maps comprises 4 up-sampling...", 
        # it implies one decoder structure produces these maps. 
        # We will assume the output is [B, 16, H', W'] and we treat it as the base feature map 
        # from which we sample, or that the decoder output is [B, 16*(K+1), H', W'].
        # Let's assume the latter for clarity in sampling.
        feature_maps_tensor = self.decoder_features(encoder_features) # [B, 16*(K+1), H', W']
        
        # Step 3: Visibility Prediction
        visibility_tensor = self.decoder_visibility(encoder_features) # [B, 16, H', W']
        
        # Step 4 & 5: Feature Sampling
        # Sample from feature_maps_tensor using sc
        # sc shape: [N, K, 2]
        # feature_maps_tensor shape: [B, 16*(K+1), H', W']
        # We need to sample K maps for each Gaussian.
        # Reshape feature_maps_tensor to [B, K+1, 16, H', W']
        feature_maps_reshaped = feature_maps_tensor.view(feature_maps_tensor.shape[0], -1, 16, feature_maps_tensor.shape[2], feature_maps_tensor.shape[3])
        
        sampled_features = []
        for k in range(self.K):
            # Select k-th feature map
            fmap_k = feature_maps_reshaped[:, k, :, :, :] # [B, 16, H', W']
            # Sample at coordinates sc[:, k, :]
            # Normalize coordinates to [-1, 1] if necessary
            coords = gaussians['sc'][:, k, :] # [N, 2]
            # Grid sample
            # Assuming coords are in [0, 1] or need normalization
            # F.grid_sample expects normalized coordinates [-1, 1]
            # Let's assume coords are normalized
            grid = coords.unsqueeze(1).unsqueeze(1) # [N, 1, 1, 2]
            # Expand fmap_k to batch size N (assuming B=1 for simplicity in this step, or B=N)
            # In practice, B might be 1 (single image set) and N is number of Gaussians
            # If B > 1, we need to handle batch dimensions carefully.
            # Assuming B=1 for the reference images
            fmap_k_expanded = fmap_k.expand(N, -1, -1, -1)
            sampled = F.grid_sample(fmap_k_expanded, grid, align_corners=True) # [N, 16, 1, 1]
            sampled_features.append(sampled.squeeze(-1).squeeze(-1)) # [N, 16]
        
        sampled_features = torch.stack(sampled_features, dim=1) # [N, K, 16]
        
        # Step 6: Fusion
        # Concatenate sampled features, intrinsic feature s_fi, and positional encoding
        # Positional encoding: 10 frequencies
        pos_enc = self._positional_encoding(gaussians['positions'], num_frequencies=10) # [N, 20]
        
        # Concat: [N, K*16] + [N, 48] + [N, 20] = [N, 16K + 68]
        # Note: The source doesn't specify the exact concatenation order or dimensions of d_fi in the fusion input.
        # It mentions "dynamic appearance features d_fi" are used with dropout.
        # Let's assume d_fi is concatenated as well.
        # d_fi dimension is not explicitly given, but let's assume it's part of the input.
        # The source says "random dropout ... applied to d_fi".
        # Let's assume the input to Mf is [sampled_features, s_fi, pos_enc, d_fi]
        # We need d_fi. Let's assume it's passed in.
        d_fi = gaussians['d_fi'] # [N, D]
        
        # Apply dropout to d_fi
        d_fi_dropped = F.dropout(d_fi, p=0.1, training=self.training)
        
        # Concatenate
        fusion_input = torch.cat([sampled_features.view(N, -1), gaussians['s_fi'], pos_enc, d_fi_dropped], dim=1)
        
        # Pass through Mf
        fused_feature = self.mf1(fusion_input)
        fused_feature = self.mf2(fused_feature) # [N, 48]
        
        # Step 7: Color Prediction
        colors = self.mc(fused_feature) # [N, 3]
        
        # Step 8: Visibility
        # Sample visibility map at Gaussian positions?
        # The source says "decoder ... for producing the visibility map".
        # It doesn't explicitly say how visibility is used per Gaussian.
        # It might be a global visibility or per-Gaussian.
        # Given it's a "visibility map", it's likely spatial.
        # We sample the visibility map at the projected 2D positions of Gaussians.
        # This requires projection, which is part of the rendering pipeline, not just the network.
        # The network outputs the visibility map tensor.
        # The actual visibility value for a Gaussian is obtained by sampling this map at its 2D screen position.
        # This step is outside the pure network forward pass if it requires projection.
        # However, the network outputs the map.
        # Let's return the visibility map tensor.
        visibility_map = visibility_tensor # [B, 16, H', W']
        
        return colors, visibility_map

    def _positional_encoding(self, positions, num_frequencies):
        # Standard positional encoding
        # positions: [N, 3]
        # Output: [N, 2 * num_frequencies]
        # This is a placeholder for the actual PE function
        pass
```

## Subtopics

### [Wave Parameter Estimation Network](13001-wave-parameter-estimation-network.md)
The Wave Parameter Estimation Network (WPEN) estimates the constituent wave parameters (amplitude, angular frequency, and propagation direction) of a dynamic ocean from a time sequence of reconstructed geometric data, specifically surface normals and microfacet roughness. It distinguishes itself from direct regression methods by employing a differentiable ocean model as a physics-based loss function, allowing the network to learn by minimizing the geometric discrepancy between predicted and input ocean states. This approach enables self-supervised training where the network iteratively optimizes wave parameters to match real-world geometry reconstructed by preceding stages.

## Operating characteristics
- **Time Complexity:** The U-Net encoder uses a pre-trained ResNet-18, which is efficient. The decoders are small (4 and 3 up-sampling convs). The MLPs are small (max 128 units). The main cost is sampling $K$ coordinates per Gaussian from $K+1$ feature maps.
- **Memory Complexity:** Stores $K+1$ feature maps (16 channels each) and the visibility map. Gaussian attributes include $s_{fi}$ (48 dims), $d_{fi}$, and $sc$ ($K \times 2$ dims).
- **Performance:** On Brandenburg, Sacre, and Trevi scenes, GS-W achieves PSNR of 24.70, SSIM of 0.8655, and LPIPS of 0.1242, with 50.7 FPS and 122.7 MB size [S1]. This outperforms global feature (PSNR 22.43) and K-planes-hybrid (PSNR 22.40) methods [S1].
- **Strengths:** Captures local highlights and avoids artifacts from transient objects [S1]. Adjusts scene appearance in a manner aligned with human perception by weighting features from reference images [S1].
- **Limitations:** Requires reference images. Computationally more complex than static 3DGS due to the U-Net and sampling.
- **When to use:** Unconstrained image collections where view-dependent appearance is important and reference images are available.
- **When not to use:** Real-time applications with strict latency constraints where FPS > 50 is required, or when no reference images are available.

## Background
The GS-W Network Architecture was introduced in the "Gaussian in the Wild" paper, which extends 3D Gaussian Splatting to unconstrained image collections [S1]. It was developed to handle the challenges of dynamic appearance and transient objects in real-world image sets, which standard 3DGS struggles with due to its static color attributes [S1]. The method builds on the Gaussian Data Structure and integrates a neural network to predict appearance features dynamically [S1].

## Sources
[S1] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
