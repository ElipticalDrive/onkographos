# Voxel Super-Resolution Decoder

Also called: Voxel Super-Resolution

## Overview
A decoder architecture designed to upsample low-resolution voxel grids to higher resolutions for super-resolution tasks, serving as a fast alternative to querying implicit MLPs for higher resolution data [S1]. It distinguishes itself from sibling methods like [U-Net](10523-u-net.md) by utilizing a specific sequence of 3D convolutional layers with instance normalization and ReLU activations, rather than a symmetric encoder-decoder skip-connection structure [S1].

## How it works
**Inputs and Outputs:**
*   Input: A low-resolution voxel grid $V_{input}$ (e.g., $160^3$).
*   Output: A high-resolution voxel grid $V_{upsample}$ (e.g., $256^3$ or $384^3$).

**Data Structures:**
The input is a 3D tensor representing voxel values. The output is a 3D tensor of the same data type but with increased spatial dimensions.

**Algorithm:**
The decoder replaces standard convolution decoders with a specific stack of four 3D convolutional layers [S1].

1.  **Convolutional Stack:** Apply four consecutive 3D convolution layers.
2.  **Normalization and Activation:** After each of the four 3D convolution layers, apply Instance Normalization followed by a ReLU activation function [S1].
3.  **Primary Upsampling:** Upsample the feature map by a factor of 2 [S1].
4.  **Final Upsampling:** Apply a final upsampling operation. The factor depends on the target output resolution:
    *   For an output resolution of $256^3$, apply an upsampling factor of 1.6 [S1].
    *   For an output resolution of $384^3$, apply an upsampling factor of 2.4 [S1].
5.  **Final Convolution:** Apply a final 3D convolution layer to produce the output grid [S1].

**Loss Function:**
The network is trained using a masked color reconstruction loss, enforced against the ground-truth upsampled grid [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_super_resolution_decoder(V_input, target_resolution):
    """
    Upsamples a voxel grid using a 3D CNN decoder.
    
    Args:
        V_input: Input voxel grid tensor.
        target_resolution: Integer, the side length of the cubic output grid (e.g., 256 or 384).
        
    Returns:
        V_output: Upsampled voxel grid tensor.
    """
    # Step 1 & 2: Four 3D Conv layers with Instance Norm and ReLU
    x = V_input
    for i in range(4):
        x = Conv3D(x)
        x = InstanceNorm(x)
        x = ReLU(x)
    
    # Step 3: Primary upsampling by factor of 2
    x = Upsample(x, factor=2)
    
    # Step 4: Final upsampling based on target resolution
    if target_resolution == 256:
        factor = 1.6
    elif target_resolution == 384:
        factor = 2.4
    else:
        # Not given in the sources for other resolutions. [G]
        # Assuming linear scaling or nearest neighbor fallback for implementation safety.
        factor = target_resolution / (256 * 2) 
        
    x = Upsample(x, factor=factor)
    
    # Step 5: Final 3D convolution
    V_output = Conv3D(x)
    
    return V_output
```

**Parameters:**
*   Input resolution: Typically $160^3$ [S1].
*   Output resolution: Typically $256^3$ or $384^3$ [S1].

## Subtopics

### Voxel-grid Super-Resolution
Predicts an upsampled voxel grid of higher resolution from a lower resolution input grid using a U-Net style encoder-decoder network [S1]. It is chosen when a fast grid upsampling alternative to querying an implicit MLP is required [S1]. The operation is defined as $V_{upsample} = g(V_{input}, \text{resolution})$, where $g$ is the U-Net style network [S1]. Typical parameters include an input resolution of $160^3$ and an output resolution of $384^3$ [S1].

## Operating characteristics
*   **Time Complexity:** Faster than querying an implicit MLP for higher resolution, as it performs a direct forward pass through a fixed-size network rather than evaluating a continuous function at arbitrary coordinates [S1].
*   **Memory Complexity:** Requires memory proportional to the output voxel grid size and the network parameters (lightweight decoders are noted in the context of NeRF-MAE) [S1].
*   **Strengths:** Provides a fast method for grid upsampling [S1].
*   **Limitations:** Resolution is discrete and fixed by the network architecture and upsampling factors; it does not support arbitrary continuous resolution queries like implicit representations [S1].

## Background
The method originates from the NeRF-MAE paper, which introduces masked autoencoders for self-supervised 3D representation learning for Neural Radiance Fields [S1]. It was developed to efficiently handle downstream 3D OBB (Oriented Bounding Box) prediction tasks by providing high-resolution voxel features without the computational cost of implicit MLP queries [S1].

## Sources
[S1] NeRF-MAE: Masked AutoEncoders for Self-Supervised 3D Representation Learning for Neural Radiance Fields, https://arxiv.org/abs/2404.01300v3, fetched 2026-10-26
