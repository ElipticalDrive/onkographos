# Neural Reconstruction Network
Also called: Feature Pyramid Consolidation Network, U-Net Decoder for Volume Rendering

## Overview
The Neural Reconstruction Network (NRN) is a compact neural network that consolidates a feature image pyramid of $n$ layers into a single full-resolution image, resembling a U-Net decoder [S1]. It builds on [Multi Resolution Alpha Blending](12118-multi-resolution-alpha-blending.md) by taking the blended output from multiple splatting resolutions and fusing them to produce the final view. Unlike hole-filling networks, the NRN is significantly smaller and more efficient because the underlying renderer autonomously fills gaps and generates smooth output via trilinear splatting, allowing the network to focus on high-quality texture reconstruction and minimal outlier removal [S1].

## How it works
**Inputs and Outputs:**
- Input: A feature image pyramid comprising $n$ layers, where each layer corresponds to a different resolution level of the rasterized scene [S1].
- Output: A single full-resolution RGB image [S1, S2].

**Data Structures:**
- The input is a list of $n$ feature maps. The exact channel dimensions are not specified, but the internal feature size is fixed at 32 [S1].
- The output is a tensor of shape $(H, W, 3)$ for RGB, where $H, W$ are the highest resolution dimensions [S2].

**Steps:**
1. Initialize the network with a U-Net decoder-like architecture [S1].
2. For each layer $l$ in the pyramid (from coarsest to finest, excluding the final layer):
   a. Apply a bilinear upsampling operation to the current feature map to match the resolution of the next finer layer [S1].
   b. Merge the upsampled output with the subsequent (finer) level's features via concatenation or addition (standard U-Net skip connection behavior implied by "merging") [S1].
   c. Process the merged features through a single gated convolution block [S1].
3. For the final layer:
   a. Process the features through a single gated convolution block without subsequent upsampling [S1].
4. Output the resulting full-resolution image [S1].

**Gated Convolution Block Details:**
- Each layer incorporates a single gated convolution [YLY* 19] [S1].
- The block includes a self-bypass connection (residual connection) [S1].
- The feature size (number of channels) is 32 [S1].
- The gated convolution mechanism typically splits the output channels into two halves: one passed through a sigmoid activation (gate) and the other multiplied element-wise [S1, YLY* 19].

**Parameters:**
- Feature size: 32 [S1].
- Number of pyramid layers: $n$ (variable, determined by the renderer's multi-resolution setup) [S1].

**Edge Cases and Pitfalls:**
- The network assumes the renderer has already produced a pyramid with sufficient coverage. If the renderer fails to fill gaps, the NRN may still produce artifacts, though it is designed to handle minor holes [S1].
- Precision issues in bilinear upsampling should be managed using standard floating-point operations [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neural_reconstruction_network(pyramid_layers):
    """
    Consolidates a feature image pyramid into a single full-resolution image.
    
    Args:
        pyramid_layers: List of feature maps [L_1, L_2, ..., L_n] from coarsest to finest.
        
    Returns:
        Full-resolution RGB image.
    """
    # Assume pyramid_layers has n elements
    n = len(pyramid_layers)
    
    # Start with the coarsest layer
    current_features = pyramid_layers[0]
    
    # Process intermediate layers
    for i in range(n - 1):
        # Bilinear upsampling to match the resolution of the next layer
        # Note: Standard bilinear interpolation
        upscaled = bilinear_upsample(current_features, target_size=pyramid_layers[i+1].shape)
        
        # Merge with the subsequent finer level
        # Assuming concatenation along channel dimension as per typical U-Net decoder
        merged = concatenate(upscaled, pyramid_layers[i+1], axis=-1)
        
        # Apply gated convolution with self-bypass
        current_features = gated_conv_block(merged, feature_size=32)
        
    # Process the final layer
    # No upsampling for the final layer
    final_output = gated_conv_block(current_features, feature_size=32)
    
    return final_output

def gated_conv_block(features, feature_size=32):
    """
    Applies a single gated convolution with a self-bypass connection.
    
    Args:
        features: Input feature map.
        feature_size: Number of channels in the output.
        
    Returns:
        Output feature map with residual connection.
    """
    # Standard gated convolution implementation
    # Split output into two halves: linear and gate
    # Typically: conv -> split -> sigmoid -> multiply
    # For simplicity, we assume a standard gated conv block exists
    # The block has feature_size channels
    
    # Apply convolution to get 2 * feature_size channels
    conv_output = convolution_2d(features, out_channels=2 * feature_size, kernel_size=3, padding=1)
    
    # Split into linear and gate parts
    linear_part, gate_part = split(conv_output, axis=-1)
    
    # Apply sigmoid to gate part
    gate = sigmoid(gate_part)
    
    # Element-wise multiplication
    gated_output = linear_part * gate
    
    # Self-bypass (residual) connection
    # Note: If input and output dimensions differ, a 1x1 conv might be needed for the skip
    # Assuming dimensions match for simplicity as per "self-bypass"
    output = gated_output + features
    
    return output

def bilinear_upsample(features, target_size):
    """
    Performs bilinear upsampling to the target size.
    
    Args:
        features: Input feature map.
        target_size: Tuple (height, width) for the output.
        
    Returns:
        Upsampled feature map.
    """
    # Standard bilinear interpolation
    # Implementation depends on framework, but logically:
    # Resize using bilinear kernel
    return resize_bilinear(features, target_size)

def concatenate(a, b, axis=-1):
    """
    Concatenates two tensors along the specified axis.
    """
    return torch.cat([a, b], dim=axis)

def convolution_2d(features, out_channels, kernel_size, padding):
    """
    Applies a 2D convolution.
    """
    # Standard 2D convolution
    # Assuming input channels are inferred from features
    return torch.nn.functional.conv2d(features, weight, bias, padding=padding)

def sigmoid(x):
    """
    Sigmoid activation function.
    """
    return 1 / (1 + torch.exp(-x))

def split(tensor, axis=-1):
    """
    Splits a tensor into two equal halves along the specified axis.
    """
    mid = tensor.shape[axis] // 2
    return tensor[..., :mid], tensor[..., mid:]

def resize_bilinear(features, target_size):
    """
    Resizes features using bilinear interpolation.
    """
    # Standard bilinear resize
    return torch.nn.functional.interpolate(features, size=target_size, mode='bilinear', align_corners=False)
```

## Subtopics

### U-Net Synthesizer with Gated Convolutions
A variant that explicitly integrates adversarial training. It processes the pyramid of rasterized raw images to generate a full-resolution RGB image in one pass, fusing coarse and fine details to cover bleeding surfaces [S2]. Unlike the basic NRN, it renders views at down-sampled resolutions in addition to the highest resolution to pair with a multi-scale discriminator, enabling joint adversarial-rasterization training for better generalization and artifact reduction [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources. However, the network is described as "significantly smaller and more efficient" than hole-filling networks [S1].
- **Memory Complexity:** Not given in the sources. The restrained number of features (32) and convolutional operations suggests low memory usage [S1].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Efficient due to small size; leverages the renderer's ability to fill gaps autonomously; focuses on high-quality texture reconstruction [S1]. The adversarial variant handles bleeding surfaces and generates realistic outputs even with insufficient point cloud information [S2].
- **Limitations:** Relies on the quality of the input pyramid; if the renderer produces poor coverage, the network may struggle [S1].
- **When to use:** When real-time or efficient rendering is required, and the underlying splatting renderer provides good initial coverage [S1]. Use the adversarial variant when high visual fidelity and generalization to unseen inputs are critical [S2].
- **When not to use:** When the renderer fails to produce a coherent pyramid or when computational resources are extremely limited and even the small network is too heavy [G].

## Background
The Neural Reconstruction Network was introduced in the TRIPS paper (Trilinear Point Splatting for Real-Time Radiance Field Rendering) [S1]. It was designed to replace traditional hole-filling networks with a more efficient decoder-like architecture, capitalizing on the smooth output generated by trilinear point splatting [S1]. The U-Net Synthesizer variant was later proposed in CE-NPBG (Connectivity Enhanced Neural Point-Based Graphics for Novel View Synthesis in Autonomous Driving Scenes) to address bleeding surfaces and improve realism through adversarial training [S2].

## Sources
[S1] TRIPS: Trilinear Point Splatting for Real‐Time Radiance Field Rendering, https://doi.org/10.1111/cgf.15012, fetched 2026-10-27
[S2] CE-NPBG: Connectivity Enhanced Neural Point-Based Graphics for Novel View Synthesis in Autonomous Driving Scenes, https://doi.org/10.1109/cvprw67362.2025.00240, fetched 2026-10-29
