# RenderNet Architecture

Also called: RenderNet

## Overview
RenderNet is a deep convolutional network designed for the differentiable rendering of voxelized 3D shapes [S1, S2, unofficial]. It solves the problem of synthesizing novel views by mapping a 3D input volume directly to a 2D image, learning visual feature mappings from exemplary images [S1, unofficial]. What sets it apart from its siblings, such as [Voxel Grid Decoders](2219-voxel-grid-decoders.md), is its explicit architectural separation of 3D feature extraction and 2D image decoding, connected by a projection unit that aggregates features along the viewing ray using an MLP [S1, unofficial]. Unlike methods that extract features in object space independent of camera parameters, RenderNet performs view-space resampling at the beginning of the pipeline, making the features inherently dependent on the camera pose [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid representing the shape.
*   **Output:** A 2D rendered image (RGB or grayscale) and optionally other modalities (e.g., albedo, normals) [S2, unofficial].

**Data Structures and Layout:**
The network consists of four main stages: 3D Input Encoder, 3D Residual Blocks, Projection Unit, and 2D Decoder [S2, unofficial].

1.  **3D Input Encoder:** A sequence of 3D convolutions.
    *   Layer 1: Channels $\{8\}$, Kernel $\{5\}$, Stride $\{2\}$.
    *   Layer 2: Channels $\{16\}$, Kernel $\{3\}$, Stride $\{2\}$.
    *   Layer 3: Channels $\{16\}$, Kernel $\{3\}$, Stride $\{1\}$.
2.  **3D Residual Blocks:** Ten 3D residual blocks follow the encoder. Each block consists of two 3D convolutions with a shortcut connection adding the input to the output [S2, unofficial].
3.  **Projection Unit:**
    *   Resizes the tensor from dimensions $W \times H \times 32 \times 16$ to $W \times H \times (32 \cdot 16)$ [S2, unofficial].
    *   Applies a $1 \times 1$ convolution with the same number of channels ($512$) [S2, unofficial].
    *   This projects all voxels along the X-axis (ray direction) to a single high-dimensional feature vector, analogous to an accumulation function [S1, unofficial].
4.  **2D Decoder:**
    *   **2D Residual Blocks:** Ten 2D residual blocks follow the projection unit. Similar to 3D blocks but using 2D convolutions [S2, unofficial].
    *   **Intermediate Conv:** A $4 \times 4$ convolution with $32 \cdot 8 = 256$ channels [S2, unofficial].
    *   **Final 2D Residual Blocks:** Five 2D residual blocks [S2, unofficial].
    *   **Output Convs:** A series of 2D convolutions:
        *   Layer 1: Channels $\{32 \cdot 4 = 128\}$, Kernel $\{4\}$, Stride $\{1\}$.
        *   Layer 2: Channels $\{32 \cdot 2 = 64\}$, Kernel $\{4\}$, Stride $\{2\}$.
        *   Layer 3: Channels $\{32\}$, Kernel $\{4\}$, Stride $\{2\}$.
        *   Layer 4: Channels $\{16\}$, Kernel $\{4\}$, Stride $\{2\}$.
        *   Layer 5: Channels $\{3\}$ (or $\{1\}$ for grayscale), Kernel $\{4\}$, Stride $\{1\}$ [S2, unofficial].

**Steps:**
1.  **Resampling:** Resample the input 3D volume to view space using a perspective camera transform. The X-axis of the resampled volume corresponds to ray directions, and the YZ-axes correspond to image coordinates [S1, unofficial].
2.  **3D Feature Extraction:** Process the resampled volume with the 3D Input Encoder and 3D Residual Blocks to extract semantic features, reducing spatial dimensions [S1, unofficial].
3.  **Ray Projection:** Use the Projection Unit to project all voxels along the X-axis to a single high-dimensional feature vector (512-channel 2D projection) [S1, unofficial].
4.  **2D Upsampling:** Upsample the 2D projection to the target resolution using the 2D Decoder (2D Residual Blocks and Output Convs) [S1, unofficial].
5.  **Multi-modal Output (Optional):** To generate additional modalities (e.g., normal maps), create a branch of 2D convolution layers starting at the first strided up-convolution layer (Layer 2 of Output Convs) and train it jointly with the main branch [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_net(input_volume_3d, camera_params):
    # Step 1: Resample to view space
    # X-axis: ray directions, YZ-axes: image coordinates
    view_space_volume = resample_to_view_space(input_volume_3d, camera_params)
    
    # Step 2: 3D Input Encoder
    # Conv3d(8, kernel=5, stride=2)
    x = conv3d(view_space_volume, out_channels=8, kernel_size=5, stride=2)
    # Conv3d(16, kernel=3, stride=2)
    x = conv3d(x, out_channels=16, kernel_size=3, stride=2)
    # Conv3d(16, kernel=3, stride=1)
    x = conv3d(x, out_channels=16, kernel_size=3, stride=1)
    
    # Step 3: 3D Residual Blocks (10 blocks)
    for _ in range(10):
        x = residual_block_3d(x)
        
    # Step 4: Projection Unit
    # Resize from WxHx32x16 to WxHx512
    # Note: 32*16 = 512
    x = resize_tensor(x, new_shape=(x.shape[0], x.shape[1], 512))
    # 1x1 Convolution with 512 channels
    x = conv1x1(x, out_channels=512)
    
    # Step 5: 2D Decoder
    # 2D Residual Blocks (10 blocks)
    for _ in range(10):
        x = residual_block_2d(x)
        
    # Intermediate Conv: 4x4, 256 channels
    x = conv2d(x, out_channels=256, kernel_size=4, stride=1)
    
    # Final 2D Residual Blocks (5 blocks)
    for _ in range(5):
        x = residual_block_2d(x)
        
    # Output Convs
    # Conv2d(128, kernel=4, stride=1)
    x = conv2d(x, out_channels=128, kernel_size=4, stride=1)
    
    # Conv2d(64, kernel=4, stride=2) -> Start of branching point for multi-modal
    x = conv2d(x, out_channels=64, kernel_size=4, stride=2)
    
    # Conv2d(32, kernel=4, stride=2)
    x = conv2d(x, out_channels=32, kernel_size=4, stride=2)
    
    # Conv2d(16, kernel=4, stride=2)
    x = conv2d(x, out_channels=16, kernel_size=4, stride=2)
    
    # Conv2d(3, kernel=4, stride=1) -> RGB Output
    rgb_output = conv2d(x, out_channels=3, kernel_size=4, stride=1)
    
    return rgb_output
```

**Parameters:**
*   Projection output channels: 512 [S1, unofficial].
*   Input volume dimensions: Not given in the sources.
*   Target image resolution: Not given in the sources.

**Edge Cases and Pitfalls:**
*   The architecture supports arbitrary input and output dimensions during and after training, but the specific dimensions provided in the notes are for the training dataset [S1, unofficial].
*   The projection unit assumes a specific tensor shape ($W \times H \times 32 \times 16$) before resizing, which may require padding or cropping if the input volume dimensions do not align with the expected feature map size after the 3D encoder [S2, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Enables learning different rendering styles; effective for synthesizing novel viewpoints for faces captured from a single viewpoint; allows multi-modal output sharing high-level information [S1, S2, unofficial].
*   **Limitations:** View-space resampling at the beginning makes features dependent on camera parameters, unlike object-space feature extraction methods [S1, unofficial].
*   **When to use:** When differentiable rendering of voxelized 3D shapes is required, particularly for novel view synthesis of faces or bodies [S1, unofficial].
*   **When not to use:** When object-space feature independence is preferred, or when explicit modeling of occlusion through absorption coefficient and alpha blending is required [S1, unofficial].

## Background
RenderNet was introduced by [S2, unofficial] as a deep convolutional network for differentiable rendering from 3D shapes. It was designed to address the challenges of rendering complex 3D shapes by learning visual feature mappings from exemplary images [S1, unofficial]. The architecture has been used in applications such as scene relighting, novel view synthesis, facial and body reenactment, and photorealistic avatars [S1, unofficial]. It has also been extended to controllable neural voxel rendering for handling high-frequency and complex textures [S1, unofficial].

## Sources
[S1] Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images, https://doi.org/10.48550/arxiv.2106.05429, fetched 2026-10-26 (unofficial)
[S2] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26 (unofficial)
