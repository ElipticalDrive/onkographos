# Voxel Grid Decoders

Also called: Convolutional Voxel Decoders, Bottleneck Voxel Decoders

## Overview
Voxel Grid Decoders model the volume function $V(\mathbf{x}; \mathbf{z})$ as a discrete 3D grid of voxels, producing an explicit 3D voxel grid as the output tensor of a neural network [S1]. This approach avoids the prohibitive computational requirements of evaluating deep MLPs at every ray-marching step, enabling real-time rendering speeds [S1]. It sets itself apart from implicit MLP decoders by trading memory for speed, storing values in a structured grid rather than computing them on the fly [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A latent code vector $\mathbf{z}$ (typically derived from an encoder network processing input images).
*   **Output:** A volume tensor $\mathbf{Y} \in \mathbb{R}^{C \times D \times D \times D}$ representing the decoded volume, where $C$ is the number of channels (typically 4 for RGB$\alpha$) and $D$ is the spatial resolution [S1].

**Data Structures:**
*   **Tensor $\mathbf{Y}$:** A 4D tensor with dimensions $C \times D \times D \times D$.
    *   $C$: Channel dimension. In experiments, $C=4$ [S1].
    *   $D$: Spatial resolution. In experiments, $D \in \{32, 128\}$ [S1].
*   **Interpolation Function $S(\cdot; \mathbf{Y})$:** Maps continuous 3D coordinates to tensor values via trilinear interpolation [S1].

**Formulas:**
The volume decoder function is defined as [S1]:
$$
\mathbf{V}(\mathbf{x}; \mathbf{z}) = S\left(\frac{\mathbf{x} - \mathbf{x}_0}{W/2}; \mathbf{g}(\mathbf{z})\right)
$$
Where:
*   $\mathbf{x}$: The 3D query point in world space.
*   $\mathbf{x}_0$: The center of the 3D cube bounding the volume.
*   $W$: The side length of the cube.
*   $\mathbf{g}(\mathbf{z})$: The neural network that maps $\mathbf{z}$ to the tensor $\mathbf{Y}$ of size $4 \times D \times D \times D$ [S1].
*   $S(\cdot; \mathbf{Y})$: An interpolation function that scales continuous values in the range $[-1, 1]$ to the grid indices $[1, D]$ along each dimension, followed by trilinear interpolation [S1].

**Parameters:**
*   **Resolution $D$:** Typically 32 or 128 [S1].
*   **Channels $C$:** Typically 4 (RGB$\alpha$) [S1].
*   **Latent Dimension:** Varies by architecture (e.g., 1024 for convolutional variants, 128 for bottleneck) [S1].

**Architectures for $\mathbf{g}(\mathbf{z})$:**
Three variants are described in the source [S1]:

1.  **Convolutional Decoder (32³):**
    *   Apply a fully-connected layer and non-linearity to transform $\mathbf{z}$ into a 1024-dimensional representation.
    *   Reinterpret the resulting vector as a $1 \times 1 \times 1$ cube with 1024 channels.
    *   Apply 5 transposed convolutions to upsample to a final size of $32^3$.
2.  **Convolutional Decoder (128³):**
    *   Same initialization as the 32³ variant.
    *   Apply 7 transposed convolutions to upsample to a final size of $128^3$.
3.  **Bottleneck Decoder (128³):**
    *   Use 3 fully-connected layers with output sizes 128, 4, and $128^3 \times 4$ respectively.
    *   The last layer represents the decoded volume directly.

**Post-processing:**
*   Apply the softplus function to the RGB$\alpha$ values in the output tensor to ensure they are non-negative [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class VoxelGridDecoder(nn.Module):
    def __init__(self, decoder_type='conv_32', D=32, C=4):
        super().__init__()
        self.decoder_type = decoder_type
        self.D = D
        self.C = C
        
        if decoder_type == 'conv_32':
            # Variant 1: 1024-dim latent -> 1x1x1x1024 -> 32x32x32x4
            self.fc = nn.Linear(1024, 1024) # Assuming input z is 1024-dim
            self.relu = nn.ReLU()
            # 5 transposed convolutions to go from 1x1x1x1024 to 32x32x32x4
            # Note: Exact kernel sizes/strides not specified in source, 
            # assuming standard upsampling logic.
            self.conv1 = nn.ConvTranspose3d(1024, 512, kernel_size=2, stride=2)
            self.conv2 = nn.ConvTranspose3d(512, 256, kernel_size=2, stride=2)
            self.conv3 = nn.ConvTranspose3d(256, 128, kernel_size=2, stride=2)
            self.conv4 = nn.ConvTranspose3d(128, 64, kernel_size=2, stride=2)
            self.conv5 = nn.ConvTranspose3d(64, 4, kernel_size=2, stride=2)
            
        elif decoder_type == 'conv_128':
            # Variant 2: 1024-dim latent -> 1x1x1x1024 -> 128x128x128x4
            self.fc = nn.Linear(1024, 1024)
            self.relu = nn.ReLU()
            # 7 transposed convolutions
            self.conv1 = nn.ConvTranspose3d(1024, 512, kernel_size=2, stride=2)
            self.conv2 = nn.ConvTranspose3d(512, 256, kernel_size=2, stride=2)
            self.conv3 = nn.ConvTranspose3d(256, 128, kernel_size=2, stride=2)
            self.conv4 = nn.ConvTranspose3d(128, 64, kernel_size=2, stride=2)
            self.conv5 = nn.ConvTranspose3d(64, 32, kernel_size=2, stride=2)
            self.conv6 = nn.ConvTranspose3d(32, 16, kernel_size=2, stride=2)
            self.conv7 = nn.ConvTranspose3d(16, 4, kernel_size=2, stride=2)
            
        elif decoder_type == 'bottleneck':
            # Variant 3: 3 FC layers: 128 -> 4 -> 128^3 * 4
            self.fc1 = nn.Linear(1024, 128)
            self.fc2 = nn.Linear(128, 4)
            self.fc3 = nn.Linear(4, 128 * 128 * 128 * 4)
            
        else:
            raise ValueError("Unknown decoder type")

    def forward(self, z):
        if self.decoder_type.startswith('conv'):
            # FC layer to 1024-dim
            x = self.fc(z)
            x = self.relu(x)
            # Reshape to 1x1x1x1024 (N, C, D, H, W) -> (1, 1024, 1, 1, 1)
            x = x.view(x.size(0), 1024, 1, 1, 1)
            
            if self.decoder_type == 'conv_32':
                x = self.relu(self.conv1(x))
                x = self.relu(self.conv2(x))
                x = self.relu(self.conv3(x))
                x = self.relu(self.conv4(x))
                x = self.conv5(x)
            elif self.decoder_type == 'conv_128':
                x = self.relu(self.conv1(x))
                x = self.relu(self.conv2(x))
                x = self.relu(self.conv3(x))
                x = self.relu(self.conv4(x))
                x = self.relu(self.conv5(x))
                x = self.relu(self.conv6(x))
                x = self.conv7(x)
                
            # Softplus for non-negativity
            x = F.softplus(x)
            
        elif self.decoder_type == 'bottleneck':
            x = self.fc1(z)
            x = self.fc2(x)
            x = self.fc3(x)
            # Reshape to (N, 4, 128, 128, 128)
            x = x.view(x.size(0), 4, self.D, self.D, self.D)
            # Softplus for non-negativity
            x = F.softplus(x)
            
        return x

def sample_volume(tensor_Y, query_points, x0, W):
    """
    Samples the voxel grid tensor Y at query points.
    tensor_Y: (N, C, D, D, D)
    query_points: (N, num_samples, 3) in world coordinates
    x0: center of the volume (3,)
    W: side length of the volume
    """
    # Normalize query points to [-1, 1] range relative to the volume
    # (x - x0) / (W/2)
    normalized_points = (query_points - x0) / (W / 2.0)
    
    # Clamp to [-1, 1] to handle points outside the volume
    normalized_points = torch.clamp(normalized_points, -1.0, 1.0)
    
    # Reshape for grid_sample: (N * num_samples, 1, 1, 1, 3)
    # grid_sample expects input (N, C, D, H, W) and grid (N, ..., 3)
    # We need to permute tensor_Y to (N, C, D, D, D) which it already is.
    # But grid_sample expects the last dimension to be coordinates.
    
    # PyTorch grid_sample expects grid of shape (*, 3) where * matches batch dims
    # We reshape normalized_points to (N * num_samples, 1, 1, 1, 3)
    batch_size = normalized_points.size(0)
    num_samples = normalized_points.size(1)
    
    # grid_sample requires the grid coordinates to be in the range [-1, 1]
    # which we have.
    
    # Reshape tensor_Y for grid_sample: (N, C, D, D, D)
    # Reshape grid: (N * num_samples, 1, 1, 1, 3)
    grid = normalized_points.view(batch_size * num_samples, 1, 1, 1, 3)
    
    # Perform trilinear interpolation
    # Note: grid_sample expects input (N, C, D, H, W)
    # tensor_Y is (N, C, D, D, D). This matches.
    samples = F.grid_sample(
        tensor_Y, 
        grid, 
        mode='trilinear', 
        padding_mode='zeros', 
        align_corners=True
    )
    
    # Reshape output: (N * C, num_samples, 1, 1) -> (N, C, num_samples)
    samples = samples.view(batch_size, tensor_Y.size(1), num_samples)
    
    return samples
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The decoder function is only evaluated inside the volume it covers. Interpolation should handle points outside the $[-1, 1]$ range, typically by clamping or returning zero [S1].
*   **Non-negativity:** RGB$\alpha$ values must be non-negative. The softplus function ensures this [S1].
*   **Memory:** High-resolution grids ($128^3$) consume significant memory. The bottleneck decoder offers a trade-off between speed and memory by using fully connected layers [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the decoder architecture. Convolutional decoders involve upsampling operations. Bottleneck decoders involve large fully connected layers.
*   **Memory Complexity:** Proportional to $C \times D^3$. For $D=128, C=4$, this is $4 \times 128^3 \times 4$ bytes $\approx 33$ MB per view (assuming float32).
*   **Measured Performance:**
    *   Convolutional 32³: 90Hz [S1].
    *   Convolutional 128³: 22Hz [S1].
    *   Bottleneck 128³: 90Hz [S1].
*   **Strengths:** Real-time rendering speeds achievable with lower resolutions or bottleneck architectures. Explicit representation allows for efficient ray marching.
*   **Limitations:** Voxel grids are limited to representing details as small as a single voxel. They are computationally expensive and wasteful for scenes with large empty spaces compared to sparse representations [S1].
*   **When to Use:** When real-time performance is critical and the scene can be adequately represented at the chosen voxel resolution.
*   **When Not to Use:** For scenes requiring high-frequency details beyond the voxel resolution or with large empty regions where sparse representations would be more efficient.

## Background
Voxel Grid Decoders were introduced in the "Neural Volumes: Learning Dynamic Renderable Volumes from Images" paper [S1]. They were proposed as an alternative to implicit MLP decoders to address the computational bottleneck of evaluating MLPs at every ray-marching step, which restricts real-time application performance [S1]. The method builds on the concept of using neural networks to generate explicit 3D representations from 2D images.

## Sources
[S1] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
