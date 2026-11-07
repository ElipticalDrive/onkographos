# RenderNet Architecture

Also called: RenderNet

## Overview
RenderNet is a deep convolutional network designed for the differentiable rendering of voxelized 3D shapes [S1, S2, unofficial]. It solves the problem of synthesizing novel views by mapping a 3D input volume directly to a 2D image [S1, S2, unofficial]. What sets it apart from its siblings, such as [Voxel Grid Decoders](2219-voxel-grid-decoders.md) and [GRU Volume Fusion](7543-gru-volume-fusion.md), is its explicit architectural separation of 3D feature extraction and 2D image decoding, connected by a projection unit that aggregates features along the viewing ray using an MLP [S1, S2, unofficial]. Unlike methods that extract features in object space independent of camera parameters, RenderNet performs view-space resampling at the beginning of the pipeline, making the features inherently dependent on the camera pose [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid representing the shape [S1, S2, unofficial].
*   **Output:** A 2D rendered image (RGB or grayscale) and optionally other modalities (e.g., albedo, normals) [S2, unofficial].

**Data Structures and Layout:**
The network consists of four main stages: 3D Input Encoder, 3D Residual Blocks, Projection Unit, and 2D Decoder [S2, unofficial].

1.  **3D Input Encoder:** A sequence of 3D convolutions [S2, unofficial].
    *   Layer 1: Channels $\{8\}$, Kernel $\{5\}$, Stride $\{2\}$ [S2, unofficial].
    *   Layer 2: Channels $\{16\}$, Kernel $\{3\}$, Stride $\{2\}$ [S2, unofficial].
    *   Layer 3: Channels $\{16\}$, Kernel $\{3\}$, Stride $\{1\}$ [S2, unofficial].
2.  **3D Residual Blocks:** Ten 3D residual blocks follow the encoder [S2, unofficial]. Each block consists of two 3D convolutions with a shortcut connection adding the input to the output [S2, unofficial].
3.  **Projection Unit:**
    *   Resizes the tensor from dimensions $W \times H \times 32 \times 16$ to $W \times H \times (32 \cdot 16)$ [S2, unofficial].
    *   Applies a $1 \times 1$ convolution with the same number of channels ($32 \cdot 16 = 512$) [S2, unofficial].
    *   This projects all voxels along the X-axis (ray direction) to a single high-dimensional feature vector, analogous to an accumulation function [S1, unofficial].
4.  **2D Decoder:**
    *   **2D Residual Blocks:** Ten 2D residual blocks follow the projection unit [S2, unofficial]. Similar to 3D blocks but using 2D convolutions [S2, unofficial].
    *   **Intermediate Conv:** A $4 \times 4$ convolution with $32 \cdot 8 = 256$ channels [S2, unofficial].
    *   **Final 2D Residual Blocks:** Five 2D residual blocks [S2, unofficial].
    *   **Output Convs:** A series of 2D convolutions with channels $\{32 \cdot 4, 32 \cdot 2, 32, 16, 3 \text{ (or } 1 \text{ for greyscale)}\}$, kernel sizes $\{4, 4, 4, 4, 4\}$, and strides $\{1, 2, 2, 2, 1\}$ [S2, unofficial].

**Steps:**
1.  **Resampling:** Resample the input 3D volume to view space using a perspective camera transform [S1, unofficial]. The X-axis of the resampled volume corresponds to ray directions, and the YZ-axes correspond to image coordinates [S1, unofficial].
2.  **3D Feature Extraction:** Process the resampled volume with the 3D Input Encoder and 3D Residual Blocks to extract semantic features, reducing spatial dimensions [S1, unofficial].
3.  **Ray Projection:** Use the Projection Unit to project all voxels along the X-axis to a single high-dimensional feature vector (512-channel 2D projection) [S1, unofficial].
4.  **2D Upsampling:** Upsample the 2D projection to the target resolution using the 2D Decoder (2D Residual Blocks and Output Convs) [S1, unofficial].
5.  **Multi-modal Output (Optional):** To generate additional modalities (e.g., normal maps), create a branch of 2D convolution layers starting at the first strided up-convolution layer (Layer 2 of Output Convs) and train it jointly with the main branch [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_net(input_volume_3d, camera_params):
    # Step 1: Resample to view space
    view_space_volume = resample_to_view_space(input_volume_3d, camera_params)
    # X-axis: ray directions, YZ-axes: image coordinates

    # Step 2: 3D Input Encoder
    x = conv3d(view_space_volume, out_channels=8, kernel_size=5, stride=2)
    x = conv3d(x, out_channels=16, kernel_size=3, stride=2)
    x = conv3d(x, out_channels=16, kernel_size=3, stride=1)

    # Step 3: 3D Residual Blocks (10 blocks)
    for _ in range(10):
        x = residual_block_3d(x)

    # Step 4: Projection Unit
    # Resize from WxHx32x16 to WxHx(32*16)
    x = resize_tensor(x, new_shape=(x.shape[0], x.shape[1], 32 * 16))
    # 1x1 Convolution with the same number of channels
    x = conv1x1(x, out_channels=32 * 16)

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
    x = conv2d(x, out_channels=128, kernel_size=4, stride=1)
    x = conv2d(x, out_channels=64, kernel_size=4, stride=2)
    # Multi-modal branch can be attached here
    x = conv2d(x, out_channels=32, kernel_size=4, stride=2)
    x = conv2d(x, out_channels=16, kernel_size=4, stride=2)
    rgb_output = conv2d(x, out_channels=3, kernel_size=4, stride=1)

    return rgb_output

# Helper definitions
def resample_to_view_space(volume, camera_params):
    # Resamples 3D volume to view space using perspective camera transform
    # X-axis aligns with ray directions, YZ-axes align with image coordinates
    pass

def conv3d(x, out_channels, kernel_size, stride):
    # Applies a 3D convolution with specified output channels, kernel size, and stride
    pass

def resize_tensor(x, new_shape):
    # Resizes tensor dimensions to new_shape (e.g., flattening last two dimensions)
    pass

def conv1x1(x, out_channels):
    # Applies a 1x1 convolution with specified output channels
    pass

def conv2d(x, out_channels, kernel_size, stride):
    # Applies a 2D convolution with specified output channels, kernel size, and stride
    pass

def residual_block_3d(x):
    # Applies two 3D convolutions with a shortcut connection adding input to output
    pass

def residual_block_2d(x):
    # Applies two 2D convolutions with a shortcut connection adding input to output
    pass
```

**Parameters:**
*   Projection output channels: 512 [S1, unofficial].
*   Input volume dimensions: Not given in the sources.
*   Target image resolution: Not given in the sources.

**Edge Cases and Pitfalls:**
*   The architecture supports arbitrary input and output dimensions during and after training, but the specific dimensions provided in the notes are for the training dataset [S1, unofficial].
*   The projection unit assumes a specific tensor shape ($W \times H \times 32 \times 16$) before resizing, which may require padding or cropping if the input volume dimensions do not align with the expected feature map size after the 3D encoder [S2, unofficial].

## Subtopics

### Alternative Architecture EC
**Definition:** An alternative model for comparison against RenderNet [S2, unofficial].
**Layout:** The 3D input encoder consists of 3D convolutions with channels $\{64, 128, 256, 512\}$, kernel sizes $\{4, 4, 4, 4\}$, and strides $\{2, 2, 2, 2\}$, using parametric ReLU [S2, unofficial]. This is followed by a fully-connected layer mapping tensors to a 200-dimensional vector and a sigmoid activation [S2, unofficial]. Lighting and pose parameters are directly concatenated to the shape latent vector [S2, unofficial]. The decoder consists of 2D convolutions with 4×4 kernels, channels $\{512, 512, 256, 256, 128, 128, 64, 64, 32, 32, 16, 1\}$, and strides $\{2, 1, 2, 1, 2, 1, 2, 1, 2, 1, 2, 1\}$ [S2, unofficial].
**When to choose:** Use when comparing against the standard RenderNet pipeline or when explicit pose/lighting conditioning via latent vector concatenation is required [S2, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Enables learning different rendering styles; effective for synthesizing novel viewpoints for faces captured from a single viewpoint; allows multi-modal output sharing high-level information [S1, S2, unofficial].
*   **Limitations:** View-space resampling at the beginning makes features dependent on camera parameters, unlike object-space feature extraction methods [S1, unofficial].
*   **When to use:** When differentiable rendering of voxelized 3D shapes is required, particularly for novel view synthesis of faces or bodies [S1, unofficial].
*   **When not to use:** When object-space feature independence is preferred, or when explicit modeling of occlusion through absorption coefficient and alpha blending is required [S1, unofficial].

## Background
RenderNet was introduced as a deep convolutional network for differentiable rendering from 3D shapes [S2, unofficial]. The architecture enables differentiable rendering of voxelized 3D shapes by combining a 3D and a 2D convolutional part connected by a projection unit [S1, unofficial]. It has been applied to scene relighting, novel view synthesis, facial and body reenactment, and photorealistic avatars [S1, unofficial]. The projection unit has also been extended to controllable neural voxel rendering for handling high-frequency and complex textures [S1, unofficial].

## Sources
[S1] Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images, https://doi.org/10.48550/arxiv.2106.05429, fetched 2026-10-26 (unofficial)
[S2] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26 (unofficial)
