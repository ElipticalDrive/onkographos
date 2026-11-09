# 3D Convolutional Neural Networks (3D CNNs)
Also called: 3D CNN

## Overview
3D CNNs extend the 2D sliding window operation to volumetric grids, stacking downsampling and upsampling stages to extract and reconstruct spatial features [S1, unofficial]. They are the dominant architectures for volume Super-Resolution (SR) and segmentation, where their local receptive fields naturally match the spatial locality of volumetric features [S1, unofficial]. Unlike 2D CNNs which process slices independently, 3D CNNs capture inter-slice dependencies, making them distinct from sibling methods like SSR-VFD which may rely on different feature propagation strategies.

## How it works
**Inputs and Outputs:**
- Input: A 3D volume tensor $V_{in} \in \mathbb{R}^{D \times H \times W}$ (or 2D slices if processing sequentially, but standard 3D CNNs expect 3D input).
- Output: A reconstructed 3D volume tensor $V_{out} \in \mathbb{R}^{D' \times H' \times W'}$ with higher resolution or refined features.

**Data Structures:**
- The core data structure is a 5D tensor for a batch of volumes: $B \times C \times D \times H \times W$, where $B$ is batch size, $C$ is channels, and $D, H, W$ are depth, height, and width.
- Convolutional layers maintain this tensor structure, changing $C$ and potentially $D, H, W$ via striding or padding.

**Algorithm Steps:**
1. **Input Projection:** The input volume is passed through an initial convolutional layer to map input channels to a hidden feature dimension $C_{hidden}$.
2. **Feature Extraction (Encoder):** A series of 3D convolutional layers extract hierarchical features. Each layer applies a kernel $K \in \mathbb{R}^{k_d \times k_h \times k_w}$ across the volume. The output at position $(d, h, w)$ for channel $c$ is:
   $$ y_{d,h,w,c} = \sum_{i,j,k} \sum_{c'} K_{i,j,k,c',c} \cdot x_{d+i, h+j, w+k, c'} + b_c $$
   where $x$ is the input feature map, $K$ is the kernel weights, and $b$ is the bias [S1, unofficial].
3. **Non-linearity:** An activation function (e.g., ReLU) is applied element-wise to introduce non-linearity.
4. **Upsampling (Decoder):** To achieve super-resolution, the feature maps are upsampled. This can be done via transposed convolutions (deconvolution) or interpolation followed by convolution. If using transposed convolution with stride $s$:
   $$ y_{d',h',w'} = \sum_{i,j,k} K_{i,j,k} \cdot x_{\lfloor d'/s \rfloor + i, \lfloor h'/s \rfloor + j, \lfloor w'/s \rfloor + k} $$
   [S1, unofficial].
5. **Reconstruction:** A final convolutional layer maps the high-dimensional feature maps back to the output channel count (e.g., 1 for grayscale volume).

**Parameters:**
- Kernel size: Typically $3 \times 3 \times 3$ or $5 \times 5 \times 5$.
- Stride: 1 for feature extraction, $>1$ for downsampling, or used in transposed convolutions for upsampling.
- Padding: 'Same' padding is common to maintain spatial dimensions during convolution.

**Edge Cases and Pitfalls:**
- **Memory Usage:** 3D convolutions are memory-intensive. The memory footprint scales with $D \times H \times W \times C_{in} \times C_{out} \times k_d \times k_h \times k_w$. Large volumes may require tiling or reduced batch sizes [G].
- **Boundary Effects:** Convolution near volume boundaries requires padding. Zero-padding can introduce artifacts; reflection or replication padding may be preferred [G].
- **Degenerate Input:** Input volumes with zero variance or constant values may lead to vanishing gradients if not properly normalized [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class Basic3DCNNBlock(nn.Module):
    def __init__(self, in_channels, out_channels, kernel_size=3, stride=1, padding=1):
        super(Basic3DCNNBlock, self).__init__()
        self.conv = nn.Conv3d(in_channels, out_channels, kernel_size=kernel_size, 
                              stride=stride, padding=padding, bias=False)
        self.bn = nn.BatchNorm3d(out_channels)
        self.relu = nn.ReLU(inplace=True)

    def forward(self, x):
        x = self.conv(x)
        x = self.bn(x)
        x = self.relu(x)
        return x

class UpsampleBlock(nn.Module):
    def __init__(self, in_channels, out_channels, scale_factor=2):
        super(UpsampleBlock, self).__init__()
        self.deconv = nn.ConvTranspose3d(in_channels, out_channels, 
                                         kernel_size=scale_factor, stride=scale_factor)
        self.bn = nn.BatchNorm3d(out_channels)
        self.relu = nn.ReLU(inplace=True)

    def forward(self, x):
        x = self.deconv(x)
        x = self.bn(x)
        x = self.relu(x)
        return x

class Simple3DCNN(nn.Module):
    def __init__(self, in_channels=1, base_channels=32, out_channels=1):
        super(Simple3DCNN, self).__init__()
        # Encoder
        self.enc1 = Basic3DCNNBlock(in_channels, base_channels)
        self.enc2 = Basic3DCNNBlock(base_channels, base_channels * 2, stride=2)
        self.enc3 = Basic3DCNNBlock(base_channels * 2, base_channels * 4, stride=2)
        
        # Bottleneck
        self.bottleneck = Basic3DCNNBlock(base_channels * 4, base_channels * 8)
        
        # Decoder
        self.dec3 = UpsampleBlock(base_channels * 8, base_channels * 4)
        self.dec3_conv = Basic3DCNNBlock(base_channels * 8, base_channels * 4) # Concat skip connection assumed
        
        self.dec2 = UpsampleBlock(base_channels * 4, base_channels * 2)
        self.dec2_conv = Basic3DCNNBlock(base_channels * 4, base_channels * 2)
        
        self.dec1 = UpsampleBlock(base_channels * 2, base_channels)
        self.dec1_conv = Basic3DCNNBlock(base_channels * 2, base_channels)
        
        # Output
        self.output_conv = nn.Conv3d(base_channels, out_channels, kernel_size=1)

    def forward(self, x):
        # Encoder
        e1 = self.enc1(x)
        e2 = self.enc2(e1)
        e3 = self.enc3(e2)
        
        # Bottleneck
        b = self.bottleneck(e3)
        
        # Decoder with skip connections (simplified, assuming concat)
        d3 = self.dec3(b)
        d3 = torch.cat([d3, e3], dim=1) # Skip connection
        d3 = self.dec3_conv(d3)
        
        d2 = self.dec2(d3)
        d2 = torch.cat([d2, e2], dim=1) # Skip connection
        d2 = self.dec2_conv(d2)
        
        d1 = self.dec1(d2)
        d1 = torch.cat([d1, e1], dim=1) # Skip connection
        d1 = self.dec1_conv(d1)
        
        out = self.output_conv(d1)
        return out
```

## Subtopics

### [U-Net](10523-u-net.md)
The U-Net variant is an encoder-decoder convolutional neural network architecture designed for volumetric segmentation and super-resolution. It extends standard 3D CNNs by adding skip connections between corresponding encoder and decoder stages, allowing the network to combine high-level semantic context with low-level spatial details. This structure distinguishes it from plain 3D CNNs by mitigating the loss of fine-grained spatial information during downsampling, making it particularly effective for tasks requiring precise boundary localization in 3D volumes.

### SRDenseNet
SRDenseNet is a deep network used as a super-resolution module to reconstruct high-resolution (HR) images from low-resolution (LR) inputs, selected for its generalization ability on medical images [S2, unofficial]. The SRDenseNet-All variant consists of four stages: low-level features, high-level features, deconvolution layers, and a reconstruction layer. The low-level feature stage receives an LR image. There are eight dense blocks learning high-level features. Each dense block has eight convolution layers, each producing 16 feature maps (128 feature maps per block). The bottleneck layer decreases feature maps to a compact model. A deconvolution layer upscales resolution from 128x128 to 512x512. A final convolutional layer with a 3x3 kernel reduces output channels to a single channel [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(D \cdot H \cdot W \cdot C_{in} \cdot C_{out} \cdot k_d \cdot k_h \cdot k_w)$ per layer. Deep networks with many layers and channels can be computationally expensive.
- **Memory Complexity:** High, due to the need to store intermediate feature maps for all spatial dimensions. Batch size is often limited by GPU memory.
- **Strengths:** Captures 3D spatial correlations effectively; dominant architecture for volume SR and segmentation [S1, unofficial].
- **Limitations:** High computational and memory cost; sensitive to input resolution and batch size; may oversmooth if optimized with MSE loss alone [S1, unofficial].
- **When to Use:** When 3D context is critical and computational resources are sufficient. Preferred for medical volume SR where generalization on unseen data classes is needed [S2, unofficial].
- **When Not to Use:** For very large volumes where memory is a bottleneck, or when 2D slice-wise processing is sufficient and 3D dependencies are weak.

## Background
3D CNNs were developed as an extension of 2D CNNs to handle volumetric data. They are widely used in scientific volume analysis, medical imaging, and video processing. The architecture is framed by the volumetric tasks it addresses, such as segmentation and super-resolution [S1, unofficial]. The use of 3D convolutions allows for the extraction of spatial features that respect the 3D nature of the data, unlike 2D methods that process slices independently.

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S2] Bournemouth University Research Online [BURO] - A Novel Implicit Neural Representation for Volume Data., https://eprints.bournemouth.ac.uk/38355/, fetched 2026-10-29 (unofficial)
