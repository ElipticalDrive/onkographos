# Voxel Super-Resolution Decoder


## Overview
A decoder architecture that upsamples low-resolution voxel grids to higher resolutions for super-resolution tasks [S1]. It replaces standard convolution decoders with a specific sequence of 3D convolutions and upsampling operations [S1].

## How it works
**Inputs and Outputs:**
- Input: A low-resolution voxel grid (3D tensor).
- Output: A high-resolution voxel grid matching the target resolution ($256^3$ or $384^3$) [S1].

**Data Structures:**
- Voxel Grid: A 3D tensor representing volumetric data. The specific bit-packing or storage format is not detailed in the sources [S1].

**Steps:**
1. **Initial Convolutional Blocks:** Apply four 3D convolution layers. After each layer, apply Instance Normalization followed by a ReLU activation [S1].
2. **Primary Upsampling:** Upsample the feature map by a factor of 2 [S1].
3. **Final Upsampling:** Apply a final upsampling operation with a non-integer factor depending on the target resolution: 1.6 for $256^3$ output or 2.4 for $384^3$ output [S1].
4. **Final Convolution:** Apply a final 3D convolution layer to produce the output grid [S1].

**Loss Function:**
The model is trained using a masked color reconstruction loss enforced against the ground-truth upsampled grid [S1]. The broader framework uses a weighted masked cross-entropy loss balanced by the inverse log propensity of each class, where propensity is the frequency of each class in the training dataset [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch.nn.functional as F

def voxel_super_resolution_decoder(input_grid, target_resolution):
    """
    Upsamples a low-resolution voxel grid to a higher resolution.
    
    Args:
        input_grid: Tensor of shape (B, C, H, W, D) representing the low-res voxel grid.
        target_resolution: Integer, either 256 or 384, defining the output side length.
        
    Returns:
        Tensor of shape (B, C_out, target_resolution, target_resolution, target_resolution).
    """
    x = input_grid
    
    # Step 1: Four 3D convolution layers with Instance Norm and ReLU
    for _ in range(4):
        x = F.conv3d(x, weight=..., padding=...) # Parameters not specified in sources [S1]
        x = F.instance_norm(x)
        x = F.relu(x)
        
    # Step 2: Primary upsampling by factor of 2
    x = F.interpolate(x, scale_factor=2)
    
    # Step 3: Final upsampling based on target resolution
    scale = 1.6 if target_resolution == 256 else (2.4 if target_resolution == 384 else None)
    if scale is None:
        raise ValueError("Target resolution must be 256 or 384")
    x = F.interpolate(x, scale_factor=scale)
    
    # Step 4: Final 3D convolution
    output = F.conv3d(x, weight=..., padding=...) # Parameters not specified in sources [S1]
    
    return output
```

**Parameters:**
- Number of initial convolutional blocks: 4 [S1].
- Primary upsampling factor: 2 [S1].
- Final upsampling factor: 1.6 for $256^3$ output, 2.4 for $384^3$ output [S1].
- Activation: ReLU [S1].
- Normalization: Instance Norm [S1].
- Loss: Masked color reconstruction loss [S1].

**Edge Cases and Pitfalls:**
- Non-integer upsampling factors (1.6 and 2.4) require interpolation-based upsampling rather than standard integer-grid operations [S1].
- Kernel sizes, filter counts, and strides for the convolution layers are not provided in the sources [S1].

## Subtopics

## Operating characteristics
- **Time and Memory Complexity:** Not given in the sources [S1].
- **Measured Performance:** The decoder is part of the NeRF-MAE architecture, which uses a 70 million parameter network [S1]. It is described as "lightweight" compared to other decoders [S1].
- **Strengths:** Not specified in the sources for this specific decoder.
- **Limitations:** Specific limitations are not detailed in the sources [S1].
- **When to use:** When fast super-resolution of voxel grids is needed [S1].
- **When not to use:** Not specified [S1].

## Background
The Voxel Super-Resolution Decoder is part of the NeRF-MAE framework, which employs masked autoencoders for self-supervised 3D representation learning for Neural Radiance Fields [S1]. It was introduced in the paper "NeRF-MAE: Masked AutoEncoders for Self-Supervised 3D Representation Learning for Neural Radiance Fields" [S1].

## Sources
[S1] NeRF-MAE: Masked AutoEncoders for Self-Supervised 3D Representation Learning for Neural Radiance Fields, https://arxiv.org/abs/2404.01300v3, fetched 2026-10-26
