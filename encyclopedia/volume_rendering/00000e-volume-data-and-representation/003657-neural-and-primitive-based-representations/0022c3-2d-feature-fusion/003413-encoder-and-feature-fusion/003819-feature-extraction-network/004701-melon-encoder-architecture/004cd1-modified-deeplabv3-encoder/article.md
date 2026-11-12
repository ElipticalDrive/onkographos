# Modified DeepLabV3 Encoder

Also called: DeepLabV3 feature extractor, F_low encoder

## Overview
The Modified DeepLabV3 Encoder extracts low-resolution canonical features ($F_{low}$) from a single input image by adapting the standard DeepLabV3 architecture to accept pixel coordinates and bypass batch normalization. It serves as the foundational feature extraction stage in the MELON Encoder Architecture, providing the semantic and structural base for subsequent global and high-resolution feature fusion. Unlike its sibling, the Hybrid Convolutional-Transformer Encoder, which focuses on processing $F_{low}$ into global features, this encoder is responsible for the initial spatial encoding of the raw image.

## How it works
**Inputs and Outputs:**
*   **Input:** A single RGB image $I$ of resolution $H \times W$.
*   **Output:** A low-resolution feature map $F_{low}$ of dimensions $H/8 \times W/8 \times C$, where $C$ is the channel count of the DeepLabV3 decoder output.

**Data Structures:**
The input tensor is constructed by concatenating the original RGB channels with 2D pixel coordinates.
*   **Input Tensor Layout:** 5 channels.
    *   Channel 0-2: Red, Green, Blue values (normalized to $[0, 1]$ or $[-1, 1]$ as per standard preprocessing).
    *   Channel 3: Normalized x-coordinates ($x/W$).
    *   Channel 4: Normalized y-coordinates ($y/H$).
*   **Internal State:** The architecture uses a ResNet34 backbone. All Batch Normalization layers are removed, and biases are reintroduced in all convolutional layers.

**Steps:**
1.  **Coordinate Generation:** For each pixel $(i, j)$ in the input image where $0 \le i < H$ and $0 \le j < W$, compute normalized coordinates:
    $$ x_{norm} = \frac{i}{H}, \quad y_{norm} = \frac{j}{W} $$
2.  **Input Assembly:** Construct the 5-channel input tensor $X_{in}$ by stacking the RGB channels and the coordinate maps:
    $$ X_{in} = [I_R, I_G, I_B, X_{map}, Y_{map}] $$
    where $X_{map}$ and $Y_{map}$ are $H \times W$ grids of $x_{norm}$ and $y_{norm}$ respectively.
3.  **Feature Extraction:** Pass $X_{in}$ through the modified DeepLabV3 encoder-decoder structure.
    *   The first convolutional layer accepts 5 input channels instead of the standard 3.
    *   The backbone is ResNet34.
    *   All Batch Normalization layers are disabled.
    *   Biases are enabled for all convolutional layers.
4.  **Feature Extraction Point:** The output $F_{low}$ is taken from the decoder layer immediately preceding the bilinear upsampling and segmentation head. This corresponds to the feature map at $1/8$th the resolution of the input.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def create_coordinate_maps(H, W, device):
    """
    Creates normalized x and y coordinate maps.
    """
    y_coords = torch.arange(H, device=device).float() / H
    x_coords = torch.arange(W, device=device).float() / W
    # Create meshgrid: y_coords is rows (height), x_coords is cols (width)
    Y_map, X_map = torch.meshgrid(y_coords, x_coords, indexing='ij')
    return X_map.unsqueeze(0), Y_map.unsqueeze(0)

def modify_deeplabv3_backbone():
    """
    Returns a DeepLabV3 model with ResNet34 backbone, modified for 5-channel input
    and without Batch Normalization.
    """
    # Note: In a real implementation, one would load a pretrained DeepLabV3
    # and modify its first conv layer and remove BN modules.
    # Here we describe the structural changes.
    model = load_deeplabv3_resnet34()
    
    # 1. Modify first conv layer to accept 5 channels
    first_conv = model.backbone.conv1
    old_weight = first_conv.weight # Shape: [out_channels, 3, kH, kW]
    new_weight = torch.cat([old_weight, torch.zeros_like(old_weight[:, :2, :, :])], dim=1)
    # Initialize new coordinate channels with small random weights or zeros
    # The source does not specify initialization, so we assume standard initialization
    # for the new channels.
    first_conv.weight = nn.Parameter(new_weight)
    first_conv.in_channels = 5
    
    # 2. Remove Batch Normalization and add biases
    for module in model.modules():
        if isinstance(module, nn.BatchNorm2d):
            # Replace BN with Identity or remove it. 
            # The source says "remove all instances... reintroducing biases".
            # This implies the BN layer is removed from the graph, 
            # and the subsequent Conv layer (if any) or the Conv layer itself 
            # must have bias=True.
            # Since BN is usually after Conv, we remove the BN module.
            # If the Conv layer had bias=False (because BN absorbed it), 
            # we must set bias=True.
            pass 
            
    # Specific handling for Conv layers: ensure bias=True
    for module in model.modules():
        if isinstance(module, nn.Conv2d):
            module.bias = nn.Parameter(torch.zeros(module.out_channels))
            module.bias.requires_grad = True
            
    return model

def extract_f_low(image_tensor, model):
    """
    Extracts F_low from image_tensor.
    image_tensor: [B, 3, H, W]
    """
    B, C, H, W = image_tensor.shape
    
    # Generate coordinate maps
    X_map, Y_map = create_coordinate_maps(H, W, image_tensor.device)
    X_map = X_map.expand(B, 1, H, W)
    Y_map = Y_map.expand(B, 1, H, W)
    
    # Concatenate RGB and Coordinates
    # Input is [B, 5, H, W]
    input_tensor = torch.cat([image_tensor, X_map, Y_map], dim=1)
    
    # Forward pass
    # The source specifies taking the output of the decoder layer 
    # before bilinear upsampling and segmentation head.
    # In DeepLabV3, this is typically the output of the ASPP module 
    # followed by the decoder's final conv, but before the final upsampling.
    # We assume the model's forward method returns this intermediate feature map
    # or we access it via an internal hook.
    features = model(input_tensor) 
    
    return features
```

**Parameters:**
*   **Input Channels:** 5 (RGB + 2 coordinates).
*   **Backbone:** ResNet34.
*   **Batch Normalization:** Disabled.
*   **Biases:** Enabled in all Conv layers.

**Edge Cases and Pitfalls:**
*   **Precision:** Coordinate normalization must be consistent with the training data distribution.
*   **Boundaries:** The coordinate maps cover the full image extent $[0, 1]$.
*   **Degenerate Input:** If the image is not square, the coordinate normalization handles aspect ratio correctly by dividing by $H$ and $W$ respectively.

## Subtopics

### High-Resolution Encoder
A variant encoder that produces high-resolution features ($F_{high}$). It differs from the standard $F_{low}$ encoder by taking the second layer output features of DeepLabV3 (rather than the raw conditioning image) along with stacked pixel coordinates as input. This allows it to capture high-frequency details that are lost in the low-resolution $F_{low}$ stream.

### Triplane Encoder for Single-Image Portrait View Synthesis
This is not a variant of the encoder itself, but the broader pipeline context in which the Modified DeepLabV3 Encoder is used. The encoder extracts $F_{low}$, which is then processed by a Vision Transformer (ViT) block from Segformer to create global features $F$, and concatenated with high-resolution features $F_{high}$ to form the final triplane representation. Key parameters in this context include $\lambda_1$ (adversarial loss weight: 0.1 for reference, 0.025 for multiview) and $\lambda_2$ (category loss weight: 1 for human faces, 0 for cat faces).

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels in the input image, $O(H \cdot W)$, due to the convolutional nature of the ResNet34 backbone.
*   **Memory Complexity:** Proportional to the memory footprint of the ResNet34 model and the intermediate feature maps. The removal of Batch Normalization reduces memory overhead associated with running mean/variance buffers.
*   **Strengths:** Provides robust low-resolution semantic features. The inclusion of pixel coordinates allows the network to learn spatial priors directly, which is crucial for canonicalizing 3D structures from 2D images.
*   **Limitations:** The low-resolution output ($1/8$ scale) loses high-frequency details, necessitating the fusion with $F_{high}$ in the full MELON pipeline.
*   **When to use:** When a feedforward, low-resolution semantic encoding of an image is required for downstream 3D representation learning, particularly in single-image portrait view synthesis.

## Background
The Modified DeepLabV3 Encoder originates from the work by Hew Chan et al. (2023) on "Real-Time Radiance Fields for Single-Image Portrait View Synthesis" [S1]. It adapts the DeepLabV3 architecture [Chen et al. 2017], originally designed for semantic segmentation, to serve as a feature extractor for 3D neural rendering. The modification of removing Batch Normalization and adding pixel coordinates is a specific adaptation to enable the network to learn spatially aware canonical features from unposed images, avoiding the need for expensive GAN inversion.

## Sources
[S1] Real-Time Radiance Fields for Single-Image Portrait View Synthesis, https://doi.org/10.1145/3592460, fetched 2026-10-27
