# Feature Extraction Network

Also called: Encoder Network for Feature Fusion, ResNet34-based Feature Extraction

## Overview
The Feature Extraction Network is a neural network component that maps 2D input images to multi-scale feature representations for use in generalizable neural radiance fields [S1]. It solves the problem of extracting robust, spatially aligned features from unposed or sparsely posed multi-view images by leveraging a ResNet34 backbone with specific up-convolutional fusion paths [S1]. It sets itself apart from sibling methods like the MELON Encoder by using a deeper residual structure with explicit multi-scale concatenation, and from the Multi-Scale Feature Extractor by relying on standard ResNet blocks rather than a Feature Pyramid Network (FPN) hierarchy [S1].

## How it works
**Inputs and Outputs:**
- Input: A set of 2D RGB images (dimensions not specified in sources) [S1].
- Output: A concatenated feature map containing multi-scale features extracted from the image [S1].

**Data Structures:**
The network consists of a backbone and two fusion paths.
1.  **Backbone:** Standard ResNet34 [S1]. The max-pooling layer after the first convolution is omitted in the specific architecture diagram [S1].
2.  **Path 1 (Deep Feature Path):**
    -   Three convolutional layers (`[Conv]×3`) [S1].
    -   Up-convolution (3x3 kernel, 256 channels) [S1].
    -   Up-convolution (3x3 kernel, 128 channels) [S1].
    -   Three convolutional layers (`[Conv]×3`, 3x3 kernel, 64 channels) [S1].
    -   Three convolutional layers (`[Conv]×3`, 3x3 kernel, 64 channels) [S1].
    -   Final convolution (7x7 kernel, 64 channels) [S1].
3.  **Path 2 (Shallow Feature Path):**
    -   Six convolutional layers (`[Conv]×6`) [S1].
    -   Three convolutional layers (`[Conv]×3`) [S1].
4.  **Residual Blocks:** Standard ResNet residual blocks are used within the backbone [S1].
5.  **Concatenation:** Features from Path 1 and Path 2 are concatenated. If spatial dimensions mismatch after up-convolution, bilinear interpolation is applied to rescale Path 1 features to match Path 2 features [S1].

**Numbered Steps:**
1.  Pass input images through the ResNet34 backbone (excluding the initial max-pooling layer) to extract hierarchical features [S1].
2.  Process features through Path 1: Apply three convolutions, then up-convolve (3x3, 256 ch), up-convolve (3x3, 128 ch), apply three convolutions (3x3, 64 ch), and apply three more convolutions (3x3, 64 ch) [S1].
3.  Process features through Path 2: Apply six convolutions, then three convolutions [S1].
4.  Align spatial dimensions: If the output of the up-convolution steps in Path 1 does not match the spatial shape of the corresponding features in Path 2, use bilinear interpolation to rescale Path 1 features [S1].
5.  Concatenate the aligned features from Path 1 and Path 2 [S1].
6.  Apply the final 7x7 convolution (64 channels) to the concatenated features to produce the output [S1].

**Formulas:**
-   Bilinear Interpolation: Used to rescale feature maps $F_{path1}$ to match $F_{path2}$ dimensions $H \times W$. The specific interpolation kernel is not defined in the sources [S1].

**Parameters:**
-   Kernel sizes: 3x3 for most convolutions, 7x7 for the final layer [S1].
-   Channels: 256, 128, 64 in Path 1; unspecified in Path 2 but implied to match Path 1 for concatenation [S1].
-   Depth sampling: 64 inverse-depth samples are used in the broader GeNeRF context, but this is a sampling parameter, not a network parameter [S1].

**Edge Cases and Pitfalls:**
-   Shape Mismatch: After up-convolution, the resulting feature map shape may not match the shape of the feature map to be concatenated. This must be handled via bilinear interpolation [S1].
-   Max-Pooling: The max-pooling layer after the first convolution is explicitly omitted, which affects the spatial reduction rate compared to standard ResNet [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class FeatureExtractionNetwork(nn.Module):
    def __init__(self):
        super(FeatureExtractionNetwork, self).__init__()
        # Backbone: ResNet34 without the first max-pooling layer
        # Note: Standard ResNet34 implementation details are assumed from [S1]
        self.backbone = self._get_resnet34_backbone()
        
        # Path 1: Deep feature processing
        self.path1_conv1 = nn.Conv2d(512, 256, kernel_size=3, padding=1) # Assuming input from backbone
        self.path1_upconv1 = nn.ConvTranspose2d(256, 256, kernel_size=3, stride=2, padding=1, output_padding=1)
        self.path1_upconv2 = nn.ConvTranspose2d(256, 128, kernel_size=3, stride=2, padding=1, output_padding=1)
        self.path1_conv2 = nn.Conv2d(128, 64, kernel_size=3, padding=1)
        self.path1_conv3 = nn.Conv2d(64, 64, kernel_size=3, padding=1)
        self.path1_conv4 = nn.Conv2d(64, 64, kernel_size=3, padding=1)
        self.path1_conv5 = nn.Conv2d(64, 64, kernel_size=3, padding=1)
        self.path1_conv6 = nn.Conv2d(64, 64, kernel_size=3, padding=1)
        self.path1_conv7 = nn.Conv2d(64, 64, kernel_size=3, padding=1)
        self.path1_final = nn.Conv2d(64, 64, kernel_size=7, padding=3)
        
        # Path 2: Shallow feature processing
        # Note: Exact channel dimensions for Path 2 are not specified in [S1]
        # Assuming they align with Path 1 for concatenation
        self.path2_conv1 = nn.Conv2d(256, 256, kernel_size=3, padding=1) # Placeholder channel count
        self.path2_conv2 = nn.Conv2d(256, 256, kernel_size=3, padding=1)
        self.path2_conv3 = nn.Conv2d(256, 256, kernel_size=3, padding=1)
        self.path2_conv4 = nn.Conv2d(256, 256, kernel_size=3, padding=1)
        self.path2_conv5 = nn.Conv2d(256, 256, kernel_size=3, padding=1)
        self.path2_conv6 = nn.Conv2d(256, 256, kernel_size=3, padding=1)
        self.path2_conv7 = nn.Conv2d(256, 256, kernel_size=3, padding=1)
        self.path2_conv8 = nn.Conv2d(256, 256, kernel_size=3, padding=1)
        self.path2_conv9 = nn.Conv2d(256, 256, kernel_size=3, padding=1)

    def _get_resnet34_backbone(self):
        # Placeholder for standard ResNet34 implementation
        # The max-pooling layer after the first convolution is omitted
        return nn.Sequential(
            nn.Conv2d(3, 64, kernel_size=7, stride=2, padding=3, bias=False),
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True),
            # MaxPool2d is omitted as per [S1]
            # ... ResNet34 blocks ...
        )

    def forward(self, x):
        # Extract features from backbone
        # Note: The exact feature map to be split into Path 1 and Path 2 is not specified
        # Assuming a split occurs at a certain depth in ResNet34
        features = self.backbone(x)
        
        # Path 1
        p1 = self.path1_conv1(features)
        p1 = self.path1_upconv1(p1)
        p1 = self.path1_upconv2(p1)
        p1 = self.path1_conv2(p1)
        p1 = self.path1_conv3(p1)
        p1 = self.path1_conv4(p1)
        p1 = self.path1_conv5(p1)
        p1 = self.path1_conv6(p1)
        p1 = self.path1_conv7(p1)
        p1 = self.path1_final(p1)
        
        # Path 2
        # Note: Input to Path 2 is not explicitly defined in [S1]
        # Assuming it processes a different level of features or the same features
        p2 = self.path2_conv1(features)
        p2 = self.path2_conv2(p2)
        p2 = self.path2_conv3(p2)
        p2 = self.path2_conv4(p2)
        p2 = self.path2_conv5(p2)
        p2 = self.path2_conv6(p2)
        p2 = self.path2_conv7(p2)
        p2 = self.path2_conv8(p2)
        p2 = self.path2_conv9(p2)
        
        # Align shapes
        if p1.shape != p2.shape:
            p1 = F.interpolate(p1, size=p2.shape[2:], mode='bilinear', align_corners=False)
            
        # Concatenate
        out = torch.cat([p1, p2], dim=1)
        
        return out
```

## Subtopics

### [MELON Encoder Architecture](11900-melon-encoder-architecture.md)
The MELON Encoder is a neural network component that maps 2D input images to camera pose parameters (azimuth, elevation, and optionally roll and translation) for use in unposed image NeRF reconstruction. It distinguishes itself from sibling encoders by using a compact, fixed-depth convolutional stack with group normalization and SiLU activations, optimized for rapid convergence of pose estimates in low-view-count scenarios.

### [Multi-Scale Feature Extractor](15126-multi-scale-feature-extractor.md)
The Multi-Scale Feature Extractor is a neural network module designed to extract semantic-aware features from input images for 3D scene understanding. It solves the problem of capturing multi-scale semantic information by combining a deep convolutional backbone with a Feature Pyramid Network (FPN). It is distinct from its sibling, the MELON Encoder Architecture, and its alternative baseline, the Multi-Scale Tri-Plane Architecture, by relying on 2D image-based convolutional hierarchies rather than 3D implicit representations or self-attention convolutions for frequency separation.

## Operating characteristics
-   **Time Complexity:** Not given in the sources.
-   **Memory Complexity:** Not given in the sources.
-   **Measured Performance:** Not given in the sources.
-   **Scaling:** Not given in the sources.
-   **Strengths and Limitations:** The architecture uses a standard ResNet34 backbone, which is computationally efficient and well-understood. The omission of the initial max-pooling layer may preserve more spatial detail. The use of bilinear interpolation for shape alignment is a simple but effective way to handle dimension mismatches. However, the exact channel dimensions for Path 2 are not specified, which may lead to ambiguity in implementation.
-   **When to use it:** When a robust, multi-scale feature extractor is needed for generalizable neural radiance fields, particularly in the context of the DBARF method [S1].
-   **When not to use it:** When computational resources are extremely limited and a lighter backbone is required, or when pose estimation is the primary goal rather than feature extraction (in which case the MELON Encoder might be more suitable) [S1].

## Background
The Feature Extraction Network was introduced in the DBARF paper (Deep Bundle-Adjusting Generalizable Neural Radiance Fields) [S1]. It is based on the ResNet34 architecture [S1]. The method is part of the broader GeNeRF framework, which uses inverse-depth sampling [S1].

## Sources
[S1] DBARF: Deep Bundle-Adjusting Generalizable Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00011, fetched 2026-10-29
