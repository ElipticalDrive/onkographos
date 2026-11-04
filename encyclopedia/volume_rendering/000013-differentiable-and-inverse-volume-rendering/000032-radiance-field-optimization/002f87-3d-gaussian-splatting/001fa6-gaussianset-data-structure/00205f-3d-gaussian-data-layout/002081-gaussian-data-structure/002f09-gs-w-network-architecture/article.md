# GS-W Network Architecture

Also called: Gaussian in the Wild model parameters

## Overview
The GS-W Network Architecture is the component of the Gaussian in the Wild (GS-W) method responsible for predicting per-Gaussian appearance and visibility attributes from a set of reference images [S1]. It solves the problem of modeling view-dependent appearance and dynamic lighting in unconstrained image collections by replacing static color attributes with features extracted from a neural network [S1]. It differs from standard 3D Gaussian Splatting by introducing a U-Net encoder-decoder structure and a fusion network to map image features to Gaussian attributes, rather than using direct MLPs or spherical harmonics [S1].

## How it works
The architecture consists of three main parts: a U-Net for feature extraction and visibility prediction, a Fusion Network ($M_f$) for combining intrinsic and dynamic features, and a Color MLP ($M_c$) for final color decoding [S1].

### Inputs and Outputs
**Inputs:**
1. A set of $K$ reference images.
2. The position $\mathbf{X}_i \in \mathbb{R}^3$ of each Gaussian point $i$.
3. An intrinsic feature vector $\mathbf{s}_{fi} \in \mathbb{R}^{48}$ associated with each Gaussian point [S1].
4. A dynamic appearance feature vector $\mathbf{d}_{fi}$ (dimension not explicitly specified, but combined with $\mathbf{s}_{fi}$ in $M_f$) [S1].

**Outputs:**
1. A predicted color $\mathbf{c}_i \in \mathbb{R}^3$ for each Gaussian point.
2. A visibility map (scalar field) indicating the presence of the Gaussian in the scene [S1].

### Data Structures
The network parameters are stored as follows:
*   **U-Net Encoder:** Pre-trained ResNet-18 weights, specifically the first 8 subconvolution modules [S1].
*   **U-Net Decoder 1 (Feature Maps):** 4 up-sampling convolution modules. Each module uses transpose convolution. The number of channels for each feature map is fixed at 16. Batch normalization and ReLU activation are applied after each up-sampling [S1].
*   **U-Net Decoder 2 (Visibility Map):** 3 up-sampling convolution modules. Same structure as Decoder 1 (transpose convolution, 16 channels, batch norm, ReLU) [S1].
*   **Fusion Network ($M_f$):** Two sequential MLP modules.
    *   Layer 1: 3 hidden layers with unit counts $[128, 96, 64]$.
    *   Layer 2: 2 hidden layers with unit counts $[48, 48]$.
*   **Color MLP ($M_c$):** 1 hidden layer with 48 units [S1].

### Parameters and Hyperparameters
*   **Positional Encoding:** 10 frequencies are used to encode the Gaussian point position $\mathbf{X}_i$ [S1].
*   **Intrinsic Feature Dimension:** 48 [S1].
*   **Initialization:** Xavier initialization is used for all weight parameters in the MLPs ($M_f$ and $M_c$) [S1].
*   **Dropout:** Random dropout with probability 0.1 is applied to the dynamic appearance features $\mathbf{d}_{fi}$ during training to prevent overfitting [S1].

### Numbered Steps
1.  **Feature Extraction:** Pass the $K$ reference images through the U-Net encoder (first 8 subconvolution modules of pre-trained ResNet-18) to extract image features [S1].
2.  **Visibility Prediction:** Pass the encoded features through the U-Net decoder with 3 up-sampling modules to produce the visibility map. Note: Skip connections between the encoder and this specific decoder are omitted to reduce computation [S1].
3.  **Feature Map Generation:** Pass the encoded features through the U-Net decoder with 4 up-sampling modules to generate $K+1$ feature maps (channels fixed at 16) [S1].
4.  **Positional Encoding:** Encode the 3D position $\mathbf{X}_i$ of each Gaussian point using 10 frequencies [S1].
5.  **Feature Fusion:** Concatenate the intrinsic feature $\mathbf{s}_{fi}$, the dynamic appearance feature $\mathbf{d}_{fi}$, and the encoded position. Pass this concatenated vector through the Fusion Network $M_f$ (two MLP modules with layers $[128, 96, 64]$ and $[48, 48]$) [S1].
6.  **Color Decoding:** Pass the output of $M_f$ through the Color MLP $M_c$ (1 hidden layer, 48 units) to predict the final color $\mathbf{c}_i$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GS_W_Network(nn.Module):
    def __init__(self, num_ref_images, intrinsic_dim=48, pos_freqs=10):
        super().__init__()
        self.pos_freqs = pos_freqs
        
        # 1. U-Net Encoder: First 8 subconvolution modules of pre-trained ResNet-18
        # [G] In implementation, load ResNet-18 and remove the final avgpool/linear layers.
        # The encoder outputs features from the first 8 conv blocks.
        self.encoder = self._get_resnet18_encoder()
        
        # 2. U-Net Decoder for Feature Maps (K+1 maps)
        # 4 up-sampling convolution modules, channels fixed at 16
        self.decoder_features = nn.Sequential(
            nn.ConvTranspose2d(in_channels=..., out_channels=16, kernel_size=3, stride=2, padding=1, output_padding=1),
            nn.BatchNorm2d(16),
            nn.ReLU(),
            # ... repeat 4 times with appropriate channel adjustments if input dim varies
            # Note: Exact input channels to decoder depend on encoder output.
            # Sources state "number of channels for each feature map fixed at 16"
            # implying the output of the decoder is 16 channels.
            # The intermediate channel sizes are not explicitly given, but standard U-Net
            # upsampling reduces channels or keeps them. We assume standard transpose conv.
            # [G] Placeholder for exact intermediate channel counts if not 16.
            # However, source says "number of channels for each feature map fixed at 16".
            # This likely refers to the output feature maps.
        )
        
        # 3. U-Net Decoder for Visibility Map
        # 3 up-sampling convolution modules, NO skip connections from encoder
        self.decoder_visibility = nn.Sequential(
            nn.ConvTranspose2d(in_channels=..., out_channels=16, kernel_size=3, stride=2, padding=1, output_padding=1),
            nn.BatchNorm2d(16),
            nn.ReLU(),
            # ... repeat 3 times
            nn.ConvTranspose2d(in_channels=16, out_channels=1, kernel_size=3, stride=2, padding=1, output_padding=1),
            # Output is a scalar visibility map
        )
        
        # 4. Fusion Network Mf
        # Two MLP modules
        # Layer 1: [128, 96, 64]
        # Layer 2: [48, 48]
        input_dim_mf = intrinsic_dim + intrinsic_dim + (pos_freqs * 2 * 3) # Assuming sin/cos encoding
        self.mf_layer1 = nn.Sequential(
            nn.Linear(input_dim_mf, 128),
            nn.ReLU(),
            nn.Linear(128, 96),
            nn.ReLU(),
            nn.Linear(96, 64),
            nn.ReLU()
        )
        self.mf_layer2 = nn.Sequential(
            nn.Linear(64, 48),
            nn.ReLU(),
            nn.Linear(48, 48),
            nn.ReLU()
        )
        
        # 5. Color MLP Mc
        # One hidden layer with 48 units
        self.mc = nn.Sequential(
            nn.Linear(48, 48),
            nn.ReLU(),
            nn.Linear(48, 3) # Output RGB
        )
        
        # Xavier initialization for MLPs
        self._init_weights()

    def _get_resnet18_encoder(self):
        # [G] Implementation detail: Load torchvision models.resnet18(pretrained=True)
        # and return the sequential module containing the first conv and 8 subconvolution blocks.
        pass

    def _init_weights(self):
        for m in self.modules():
            if isinstance(m, nn.Linear):
                nn.init.xavier_uniform_(m.weight)

    def positional_encoding(self, x):
        # Encode position with 10 frequencies
        # [G] Standard sinusoidal positional encoding
        pass

    def forward(self, images, intrinsic_features, dynamic_features, positions):
        # images: [B, K, H, W]
        # intrinsic_features: [N, 48]
        # dynamic_features: [N, D]
        # positions: [N, 3]
        
        # Step 1: Extract features
        # [G] images need to be processed by the encoder.
        # Assuming encoder outputs a feature map of size [B, C, H', W']
        encoded_features = self.encoder(images)
        
        # Step 2: Generate Visibility Map
        # No skip connections
        visibility_map = self.decoder_visibility(encoded_features)
        
        # Step 3: Generate Feature Maps (K+1 maps)
        feature_maps = self.decoder_features(encoded_features)
        
        # Step 4 & 5: Fusion
        # Encode positions
        pos_encoded = self.positional_encoding(positions) # [N, 60]
        
        # Concatenate intrinsic, dynamic, and encoded position
        # [G] Dynamic features dimension D is not given. Assuming it matches intrinsic or is provided.
        # Source says "random dropout ... applied to dynamic appearance features dfi"
        # We assume dynamic_features is available.
        fusion_input = torch.cat([intrinsic_features, dynamic_features, pos_encoded], dim=-1)
        
        # Apply dropout to dynamic features during training
        if self.training:
            fusion_input[..., :dynamic_features.shape[-1]] = F.dropout(dynamic_features, p=0.1)
            
        fused_features = self.mf_layer1(fusion_input)
        fused_features = self.mf_layer2(fused_features)
        
        # Step 6: Color Decoding
        colors = self.mc(fused_features)
        
        return colors, visibility_map, feature_maps
```

### Edge Cases and Pitfalls
*   **Skip Connections:** The visibility map decoder explicitly omits skip connections from the encoder to reduce computation. Implementing standard U-Net skip connections here would be incorrect for this method [S1].
*   **Dynamic Feature Dropout:** The dropout is applied *only* to the dynamic appearance features $\mathbf{d}_{fi}$ during training, not to the intrinsic features or position [S1].
*   **Initialization:** The U-Net encoder uses pre-trained ResNet-18 weights. The MLPs use Xavier initialization. Other components (like the dynamic sampling coordinates mentioned in the notes) have separate initialization procedures not detailed in this section [S1].

## Operating characteristics
*   **Performance:** On the Brandenburg, Sacre, and Trevi scenes, GS-W achieves a PSNR of 24.70, SSIM of 0.8655, and LPIPS of 0.1242 [S1].
*   **Speed:** The method runs at 50.7 FPS [S1].
*   **Size:** The model size is 122.7 MB [S1].
*   **Comparison:** It outperforms "global feature" and "K-planes-hybrid" methods in PSNR and SSIM, and is significantly faster than K-planes-hybrid (50.7 FPS vs 0.67 FPS) [S1].
*   **Strengths:** Captures local highlights and avoids artifacts caused by transient objects [S1].
*   **Limitations:** Not given in the sources.

## Background
The GS-W Network Architecture was introduced in the paper "Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections" [S1]. It extends 3D Gaussian Splatting to handle unconstrained image collections by weighting features extracted from reference images, aligning scene appearance adjustments with human perception [S1].

## Sources
[S1] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
