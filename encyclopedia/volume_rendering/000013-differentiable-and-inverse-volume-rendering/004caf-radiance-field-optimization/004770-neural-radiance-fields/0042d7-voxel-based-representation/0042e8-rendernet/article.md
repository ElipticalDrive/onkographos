# RenderNet

Also called: Differentiable Rendering Convolutional Network

## Overview
RenderNet is a differentiable rendering convolutional neural network that generates 2D images from 3D voxel grids by jointly learning visibility computation and pixel color calculation (shading) [S1, unofficial]. It solves the problem of non-differentiability in traditional rendering pipelines (rasterization and ray tracing) caused by discrete operations like z-buffering and ray-object intersection, which prevents explicit correlation between rendering parameters and the resulting image for inverse rendering tasks [S1, unofficial]. Unlike mesh-based differentiable renderers, RenderNet handles erroneous or low-resolution voxel inputs robustly and generalizes well to unseen categories, whereas alternatives like EC/EC-Deep fail to generalize due to fully connected layers and require clean high-quality meshes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid $V$ of dimensions $H_V \times W_V \times D_V \times C_V$ (height, width, depth, channel) [S1, unofficial]. Typical input resolution is $64 \times 64 \times 64$ voxels [S1, unofficial].
*   **Parameters:** Camera pose (azimuth $\theta$, elevation $\phi$, distance $R$), light position, and optionally texture coefficients [S1, unofficial].
*   **Output:** A 2D RGB image $I$ of dimensions $H_I \times W_I \times C_I$ (height, width, channel) [S1, unofficial]. Typical output resolution is $512 \times 512$ pixels [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 4D tensor representing the 3D shape.
*   **Squeezed Tensor:** A 3D tensor $V'$ with dimensions $W \times H \times (D \cdot C)$, created by collapsing the depth dimension with the feature maps [S1, unofficial].
*   **Normal Map:** A 2D tensor where RGB channels encode the normal vector $\tilde{n}$ [S1, unofficial].
*   **Albedo Map:** A 2D tensor representing surface color [S1, unofficial].

**Numbered Steps:**

1.  **Coordinate Transformation:** Apply a rigid-body transformation to the input voxel grid $V$ to convert from world coordinates to camera coordinates. The camera is assumed to be axis-aligned, looking along the negative z-axis of the volumetric grid [S1, unofficial]. The transformation is parameterized by:
    *   Rotation around the y-axis (azimuth).
    *   Rotation around the z-axis (elevation).
    *   Distance $R$ (scaling factor) [S1, unofficial].
    *   The input voxel grid is embedded into a larger grid to prevent the object from being cut off after rotation [S1, unofficial].

2.  **Trilinear Sampling:** Trilinearly sample the transformed input voxel grid to obtain continuous feature values at arbitrary positions [S1, unofficial].

3.  **3D Convolutional Encoding:** Pass the sampled input through a sequence of 3D convolutions and PReLU activations to encode geometry. The specific architecture is:
    *   Conv3d
    *   PReLU
    *   Conv3d
    *   PReLU
    *   PReLU
    *   Down-conv3d [S1, unofficial]

4.  **Projection Unit (Visibility & Projection):** This is the core differentiable component that bridges the 3D input and 2D output. It consists of two sub-steps:
    *   **Reshaping:** Collapse the depth dimension with the feature maps to map the incoming 4D tensor to a 3D squeezed tensor $V'$ with dimension $W \times H \times (D \cdot C)$ [S1, unofficial].
    *   **MLP (1x1 Convolution):** Apply a Multilayer Perceptron (implemented as a 1x1 convolution) on each $(D \cdot C)$ vector of the squeezed tensor to produce a 3D tensor with $K$ channels. This allows the network to learn visibility of different parts of the 3D input along the depth dimension, unlike max pooling which only creates a silhouette map [S1, unofficial]. The operation is defined as:
        $$ I_{i,j,k} = \sum_{d_c} w_{k,d_c} \cdot V'_{i,j,d_c} + b_k $$
        where $i, j$ are pixel coordinates, $k$ is the image channel, $d_c$ is the squeezed depth channel (combining original depth $d$ and channel $c$), $w$ are weights, and $b$ are biases [S1, unofficial].

5.  **2D Convolutional Shading:** Apply 2D convolutions and PReLU activations to the projected features to generate the final RGB image. The specific architecture is:
    *   Conv2d
    *   PReLU
    *   Conv2d
    *   PReLU
    *   Up-conv2d
    *   PReLU
    *   PReLU
    *   Up-conv2d [S1, unofficial]
    *   The network can optionally output a Normal Map and a Final Output (depth/visibility) alongside the RGB image [S1, unofficial].

6.  **Shading Application (Optional/Explicit):** If Phong illumination is desired, the network renders Normal Maps and Albedo Maps. The shaded color $S$ is calculated as:
    $$ S = \max(0, \tilde{l} \cdot \tilde{n} + a) $$
    where $\tilde{l}$ is the unit light direction vector, $\tilde{n}$ is the normal vector (RGB channels of the normal map), and $a$ is an ambient constant. The final image $I$ is created by combining the albedo map $A$ and shading $S$:
    $$ I = A \cdot S $$
    [S1, unofficial]

7.  **Texture Mapping (Optional):** A texture-mapping network maps a 1D texture vector (e.g., PCA coefficients) to a 3D texture representation. This is concatenated with the input 3D shape before being given to RenderNet to render the albedo map [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class RenderNet(nn.Module):
    def __init__(self, input_res=64, output_res=512, channels=1):
        super(RenderNet, self).__init__()
        # 3D Convolutional Encoder
        self.conv3d_1 = nn.Conv3d(channels, 16, kernel_size=3, padding=1)
        self.conv3d_2 = nn.Conv3d(16, 32, kernel_size=3, padding=1)
        self.conv3d_3 = nn.Conv3d(32, 64, kernel_size=3, padding=1)
        self.conv3d_down = nn.Conv3d(64, 128, kernel_size=3, stride=2, padding=1)
        
        # Projection Unit: Reshape + 1x1 Conv (MLP)
        # Input: H x W x D x C -> Output: H x W x K
        # The 1x1 conv operates on the flattened depth-channel dimension
        self.projection_conv = nn.Conv2d(128 * 64, 512, kernel_size=1) # Assuming D=64, C=128 after downsample? 
        # Note: The paper describes collapsing D and C. 
        # If input to projection is H' x W' x D' x C', output is H' x W' x K.
        # Implemented as 1x1 Conv2d on the squeezed tensor.
        
        # 2D Convolutional Decoder
        self.conv2d_1 = nn.Conv2d(512, 256, kernel_size=3, padding=1)
        self.conv2d_2 = nn.Conv2d(256, 128, kernel_size=3, padding=1)
        self.conv2d_up1 = nn.ConvTranspose2d(128, 64, kernel_size=4, stride=2, padding=1)
        self.conv2d_up2 = nn.ConvTranspose2d(64, 3, kernel_size=4, stride=2, padding=1)
        
        self.prelu = nn.PReLU()

    def forward(self, voxel_grid, pose_params):
        """
        voxel_grid: (B, C, H, W, D)
        pose_params: (B, 3) [azimuth, elevation, distance]
        """
        # Step 1 & 2: Transformation and Sampling
        # [G] In a real implementation, this would involve grid_sample with 
        # a transformation matrix derived from pose_params.
        # For this pseudocode, we assume a function `transform_and_sample` exists
        # that handles the rigid-body transform and trilinear sampling.
        sampled_features = transform_and_sample(voxel_grid, pose_params)
        
        # Step 3: 3D Convolutional Encoding
        x = self.prelu(self.conv3d_1(sampled_features))
        x = self.prelu(self.conv3d_2(x))
        x = self.prelu(self.conv3d_3(x))
        x = self.conv3d_down(x) # Down-samples spatially
        
        # Step 4: Projection Unit
        # Reshape: (B, C', H', W', D') -> (B, C'*D', H', W')
        B, C, H, W, D = x.shape
        x_squeezed = x.view(B, C * D, H, W)
        
        # Apply 1x1 Conv (MLP) to learn visibility and projection
        # Output: (B, K, H, W)
        x_projected = self.projection_conv(x_squeezed)
        
        # Step 5: 2D Convolutional Shading
        x = self.prelu(self.conv2d_1(x_projected))
        x = self.prelu(self.conv2d_2(x))
        x = self.prelu(self.conv2d_up1(x))
        x = self.prelu(self.conv2d_up1(x)) # Note: Paper lists two PReLUs before second Up-conv
        x = self.conv2d_up2(x)
        
        # Output: (B, 3, H_out, W_out)
        return torch.sigmoid(x) # Assuming normalized RGB output

def transform_and_sample(voxel_grid, pose_params):
    """
    Applies rigid-body transformation and trilinear sampling.
    [G] This function would construct the transformation matrix from azimuth, 
    elevation, and distance, then use torch.nn.functional.grid_sample 
    to sample the voxel grid.
    """
    # Placeholder for the actual geometric transformation logic
    pass
```

**Parameters:**
*   **Optimizer:** Adam with learning rate 0.00001 [S1, unofficial].
*   **Loss Function:** Mean Squared Error (MSE) for colored images; Binary Cross Entropy for grayscale images [S1, unofficial].
*   **Training Strategy:** Patch training. Random spatially cropped samples (width/height) are used, keeping depth/channel intact. Patch size starts at 1/8 of the full grid and increases to 1/2 at the end of training. Full-sized grid is used only during inference [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Noisy/Low-Resolution Input:** RenderNet handles low-resolution or noisy voxel inputs better than mesh renderers [S1, unofficial].
*   **Inverse Rendering Instability:** Directly minimizing reconstruction loss for shape reconstruction leads to noisy, unstable results. This is solved by using a shape prior (a pre-trained 3D auto-encoder) to regularize the process, optimizing the latent representation $z'$ instead of the shape $z$ directly [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On the Phong-shaded chair dataset, RenderNet achieved PSNR scores of 25.39 (Phong), 24.21 (Contour), 20.88 (Cartoon), 19.70 (AO), and 17.77 (Face). For comparison, EC model scores were 22.37 (Phong) and 27.43 (EC-Deep Phong) [S1, unofficial].
*   **Strengths:**
    *   Handles occlusion and complex shaders (ambient occlusion, contour shading) without explicit geometric visibility tests [S1, unofficial].
    *   Robust to erroneous, low-resolution, or noisy voxel inputs [S1, unofficial].
    *   Generalizes well to unseen categories (e.g., trained on chairs, renders Stanford Bunny) [S1, unofficial].
    *   Can learn different shaders with the same network architecture [S1, unofficial].
*   **Limitations:**
    *   Requires voxel representation of 3D shapes, which can be memory inefficient compared to other formats (though octrees can mitigate this) [S1, unofficial].
    *   Assumes axis-aligned camera looking along the negative z-axis (though perspective views are possible via 3D convolutions) [S1, unofficial].
*   **When to use:** When differentiable rendering from voxel data is required, especially for inverse rendering tasks (shape, pose, lighting, texture estimation) where robustness to input quality and generalization to unseen categories are important.
*   **When not to use:** When high-resolution mesh-based rendering is required, or when memory efficiency of the input representation is critical and octrees are not an option.

## Background
RenderNet was introduced by Thu Nguyen-Phuoc, Chuan Li, Stephen Balaban, and Yong-Liang Yang from the University of Bath and Lambda Labs. The paper "RenderNet: A deep convolutional network for differentiable rendering from 3D shapes" was published on arXiv in 2018 (v3 on 1 Apr 2019) [S1, unofficial]. It addresses the challenge of inverse rendering by framing the rendering process as a feed-forward CNN, allowing the network to learn visibility and shading jointly. It was demonstrated on tasks such as novel-view synthesis, pose prediction, and image-based 3D shape reconstruction [S1, unofficial].

## Sources
[S1] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26 (unofficial)
