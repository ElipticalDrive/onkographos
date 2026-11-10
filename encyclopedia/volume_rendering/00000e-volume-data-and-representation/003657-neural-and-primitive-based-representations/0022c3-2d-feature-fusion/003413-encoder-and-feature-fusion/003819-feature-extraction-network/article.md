# Feature Extraction Network

Also called: Encoder Network for Feature Fusion, ResNet34-based Feature Extraction

## Overview
The Feature Extraction Network is a neural architecture designed to extract spatial features from input images for downstream volume rendering tasks, specifically serving as the backbone for feature fusion in methods like DBARF [S1]. It builds on [ResNet Architecture](8378-resnet-architecture.md) to provide robust feature representations that are subsequently up-sampled and concatenated to support multi-scale density prediction. It differs from [CNN-based 2D Feature Extraction](8649-cnn-based-2d-feature-extraction.md) by its specific integration of up-convolutional layers for feature fusion and from [Attention-based Multi-View Density Prediction](10674-attention-based-multi-view-density-prediction.md) by relying on spatial convolutional hierarchies rather than attention mechanisms for feature aggregation.

## How it works
The network processes input images to produce feature maps that are fused across scales. The implementation relies on a ResNet34 backbone followed by a decoder-like structure with up-convolutions.

### Inputs and Outputs
- **Input**: A set of 2D images. In the context of the MELON variant, these are mapped to view directions [S2].
- **Output**: Feature maps of varying resolutions, ready for concatenation with other feature streams or for density prediction.

### Data Structures
The network maintains feature maps as 4D tensors $(N, C, H, W)$, where $N$ is batch size, $C$ is channels, $H$ is height, and $W$ is width.

### Steps
1. **Backbone Extraction**: Pass the input image through a ResNet34 backbone [S1]. The architecture consists of an initial convolutional layer, followed by max-pooling (omitted in some diagrams but standard in ResNet), and a series of residual blocks [S1].
2. **Feature Path 1 (Up-sampling)**:
   - Apply three convolutional layers [S1].
   - Apply an up-convolution (UpConv) with a $3 \times 3$ kernel and 256 output channels [S1].
   - Apply a second up-convolution with a $3 \times 3$ kernel and 128 output channels [S1].
   - Apply three convolutional layers with $3 \times 3$ kernels and 64 output channels [S1].
   - Apply three more convolutional layers with $3 \times 3$ kernels and 64 output channels [S1].
   - Apply a final convolution with a $7 \times 7$ kernel and 64 output channels [S1].
3. **Feature Path 2**:
   - Apply six convolutional layers [S1].
   - Apply three convolutional layers [S1].
4. **Concatenation and Resizing**:
   - Concatenate features from Path 1 and Path 2 at appropriate stages [S1].
   - If the spatial dimensions of the feature maps do not match after up-convolution, apply bilinear interpolation to rescale them to the same shape before concatenation [S1].

### Parameters
- **Input Resolution ($L$)**: The resolution of the encoder input. Typical values range from $32 \times 32$ to $128 \times 128$, chosen per scene in primary experiments [S2].
- **Feature Count ($F$)**: The number of features. Optimal value observed at 32 [S2].
- **Depth ($d$)**: Encoder depth. Optimal value observed at 4 layers [S2].

### Edge Cases and Pitfalls
- **Shape Mismatch**: After up-convolution, the resulting feature map shape may not match the shape of the feature map to be concatenated. Bilinear interpolation must be explicitly applied to resolve this [S1].
- **MaxPooling Omission**: In the specific diagram provided in the source, the max-pooling layer after the first convolution is omitted, but standard ResNet blocks imply its presence or equivalent stride handling [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class FeatureExtractionNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        # Backbone: ResNet34
        self.backbone = self._get_resnet34()
        
        # Path 1: Up-sampling and convolutional blocks
        # [Conv]x3 -> UpConv(256) -> UpConv(128) -> [Conv]x3(64) -> [Conv]x3(64) -> 7x7(64)
        self.path1_conv1 = nn.Sequential(
            nn.Conv2d(512, 256, 3, padding=1), # Assuming input from ResNet stage 4
            nn.ReLU(inplace=True),
            nn.Conv2d(256, 256, 3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(256, 256, 3, padding=1),
            nn.ReLU(inplace=True)
        )
        self.path1_upconv1 = nn.ConvTranspose2d(256, 256, 3, stride=2, padding=1, output_padding=1)
        self.path1_upconv2 = nn.ConvTranspose2d(256, 128, 3, stride=2, padding=1, output_padding=1)
        
        self.path1_conv2 = nn.Sequential(
            nn.Conv2d(128, 64, 3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(64, 64, 3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(64, 64, 3, padding=1),
            nn.ReLU(inplace=True)
        )
        
        self.path1_conv3 = nn.Sequential(
            nn.Conv2d(64, 64, 3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(64, 64, 3, padding=1),
            nn.ReLU(inplace=True),
            nn.Conv2d(64, 64, 3, padding=1),
            nn.ReLU(inplace=True)
        )
        
        self.path1_final = nn.Conv2d(64, 64, 7, padding=3)

        # Path 2: [Conv]x6 -> [Conv]x3
        self.path2_conv1 = nn.Sequential(
            nn.Conv2d(512, 256, 3, padding=1), nn.ReLU(inplace=True),
            nn.Conv2d(256, 256, 3, padding=1), nn.ReLU(inplace=True),
            nn.Conv2d(256, 256, 3, padding=1), nn.ReLU(inplace=True),
            nn.Conv2d(256, 256, 3, padding=1), nn.ReLU(inplace=True),
            nn.Conv2d(256, 256, 3, padding=1), nn.ReLU(inplace=True),
            nn.Conv2d(256, 256, 3, padding=1), nn.ReLU(inplace=True)
        )
        self.path2_conv2 = nn.Sequential(
            nn.Conv2d(256, 128, 3, padding=1), nn.ReLU(inplace=True),
            nn.Conv2d(128, 128, 3, padding=1), nn.ReLU(inplace=True),
            nn.Conv2d(128, 128, 3, padding=1), nn.ReLU(inplace=True)
        )

    def _get_resnet34(self):
        # Placeholder for actual ResNet34 implementation
        # Returns features from stage 4 (512 channels)
        pass

    def forward(self, x):
        # x: (N, 3, H, W)
        feat = self.backbone(x) # (N, 512, H/32, W/32)
        
        # Path 1
        p1 = self.path1_conv1(feat)
        p1 = self.path1_upconv1(p1)
        p1 = self.path1_upconv2(p1)
        p1 = self.path1_conv2(p1)
        p1 = self.path1_conv3(p1)
        p1 = self.path1_final(p1)
        
        # Path 2
        p2 = self.path2_conv1(feat)
        p2 = self.path2_conv2(p2)
        
        # Concatenation with resizing if needed
        # Note: Exact concatenation points depend on specific fusion strategy not fully detailed in text
        # Assuming final fusion of p1 and p2
        if p1.shape != p2.shape:
            p2 = F.interpolate(p2, size=p1.shape[2:], mode='bilinear', align_corners=False)
        
        fused = torch.cat([p1, p2], dim=1)
        return fused
```

## Subtopics

### MELON Encoder Architecture
The MELON variant uses a different encoder structure compared to the ResNet34-based DBARF approach. It consists of convolutional layers interlaced with SiLU activation functions, max-pooling layers, and group normalization layers [S2]. It maps three of the encoder outputs to an azimuth/elevation pair $(\theta, \phi)$ using spherical coordinates [S2]. It supports optional prediction of camera roll $\rho$ for real datasets and can add three output dimensions to predict $T_i$, the location of the origin in the camera reference frame [S2]. The optimal configuration for MELON uses an input resolution $L$ of 64, feature count $F$ of 32, and depth $d$ of 4 layers [S2].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: The MELON variant achieves competitive PSNR, SSIM, and LPIPS scores on NeRF Synthetic scenes (Lego, Hotdog, Chair, Drums, Mic, Ship) compared to GNeRF and VMRF [S2].
- **Strengths**: Robust feature extraction via ResNet34; handles shape mismatches via bilinear interpolation [S1].
- **Limitations**: Specific architectural details for the ResNet34 variant are limited to the appendix description [S1].

## Background
The Feature Extraction Network is based on ResNet34 [S1]. It was introduced in the context of DBARF (Deep Bundle-Adjusting Generalizable Neural Radiance Fields) [S1]. The MELON variant was introduced in "MELON: NeRF with Unposed Images in SO(3)" [S2].

## Sources
[S1] DBARF: Deep Bundle-Adjusting Generalizable Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00011, fetched 2026-10-29
[S2] MELON: NeRF with Unposed Images in SO(3), https://doi.org/10.1109/3dv62453.2024.00084, fetched 2026-10-30
