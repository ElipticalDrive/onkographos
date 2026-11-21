# Voxel Grid Decoders

Also called: Convolutional Volume Decoder, Bottleneck Volume Decoder

## Overview
Voxel Grid Decoders model the volume function $V(\mathbf{x}; \mathbf{z})$ as a discrete 3D grid of voxels, producing an explicit 3D voxel grid as the output tensor of a neural network [S1]. This approach avoids the prohibitive computational and memory requirements of evaluating large Multi-Layer Perceptrons (MLPs) at every step along each ray in the ray-marching process, which is necessary for real-time applications [S1]. It serves as the explicit representation component within Neural Volumes, replacing the implicit coordinate-based MLP [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A latent code vector $\mathbf{z}$ (typically from an encoder network).
*   **Output:** A tensor $\mathbf{Y} \in \mathbb{R}^{C \times D \times D \times D}$ representing a $D \times D \times D$ grid of values with $C$ channels [S1]. In standard experiments, $C=4$ (representing RGB$\alpha$) [S1].

**Data Structures:**
*   **Voxel Grid Tensor:** A 4D tensor with dimensions $C \times D \times D \times D$.
*   **Interpolation Function $S$:** Samples from grid $\mathbf{Y}$ by scaling continuous values in the range $[-1, 1]$ to the grid indices $[1, D]$ along each dimension, followed by trilinear interpolation [S1].

**Parameters:**
*   $D$: Spatial resolution. Typical values are $32$ or $128$ [S1].
*   $W$: Side size of the 3D cube covered by the volume [S1].
*   $\mathbf{x}_0$: Center of the 3D cube [S1].
*   $\mathbf{z}$: Latent representation vector [S1].

**Formulas:**
The volume decoder function is defined as [S1]:
$$
\mathbf{V}(\mathbf{x}; \mathbf{z}) = S\left(\frac{\mathbf{x} - \mathbf{x}_0}{W/2}; \mathbf{g}(\mathbf{z})\right)
$$
where $\mathbf{g}(\mathbf{z})$ is the neural network that produces the tensor $\mathbf{Y}$ of size $4 \times D \times D \times D$ [S1]. The function $S$ performs the coordinate transformation and interpolation [S1].

**Implementation Variants:**
The function $\mathbf{g}(\mathbf{z})$ can be implemented using either a convolutional architecture or a bottleneck fully-connected architecture [S1].

1.  **Convolutional Decoder (Variant 1):**
    *   Apply a fully-connected layer and non-linearity to transform $\mathbf{z}$ into a 1024-dimensional representation [S1].
    *   Reinterpret the resulting vector as a $1 \times 1 \times 1$ cube with 1024 channels [S1].
    *   Apply 5 transposed convolutions to achieve a final size of $32^3$ [S1].
    *   Performance: Runs at 90Hz [S1].

2.  **Convolutional Decoder (Variant 2):**
    *   Same initialization as Variant 1 [S1].
    *   Apply 7 transposed convolutions to achieve a final size of $128^3$ [S1].
    *   Performance: Runs at 22Hz [S1].

3.  **Bottleneck Decoder:**
    *   Use 3 fully-connected layers with output sizes of 128, 4, and $128^3 \times 4$ respectively [S1].
    *   The last layer represents the decoded volume [S1].
    *   Performance: Runs at 90Hz [S1].

**Post-processing:**
After decoding the volume, apply a softplus function to the RGB$\alpha$ values to ensure they are non-negative [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class VoxelGridDecoderConv32(nn.Module):
    """Convolutional decoder variant 1: 32^3 resolution."""
    def __init__(self, latent_dim):
        super().__init__()
        # Transform latent z into 1024-dimensional representation
        self.fc1 = nn.Linear(latent_dim, 1024)
        self.relu = nn.ReLU()
        
        # Reinterpret as 1x1x1 cube with 1024 channels
        # Apply 5 transposed convolutions to achieve 32^3
        # Note: Specific kernel sizes/strides not given in sources [G]
        # Assuming standard upsampling factors to reach 32 from 1
        self.conv1 = nn.ConvTranspose3d(1024, 512, kernel_size=4, stride=2, padding=1)
        self.conv2 = nn.ConvTranspose3d(512, 256, kernel_size=4, stride=2, padding=1)
        self.conv3 = nn.ConvTranspose3d(256, 128, kernel_size=4, stride=2, padding=1)
        self.conv4 = nn.ConvTranspose3d(128, 64, kernel_size=4, stride=2, padding=1)
        self.conv5 = nn.ConvTranspose3d(64, 4, kernel_size=4, stride=2, padding=1)

    def forward(self, z):
        x = self.fc1(z)
        x = self.relu(x)
        # Reshape to 4D tensor for ConvTranspose3d: (N, C, D, H, W)
        # Here N=1, C=1024, D=1, H=1, W=1
        x = x.view(x.size(0), 1024, 1, 1, 1)
        
        x = self.relu(self.conv1(x))
        x = self.relu(self.conv2(x))
        x = self.relu(self.conv3(x))
        x = self.relu(self.conv4(x))
        # Final output is 4 channels (RGBa)
        x = self.conv5(x)
        
        # Apply softplus to ensure non-negative values
        x = F.softplus(x)
        return x

class VoxelGridDecoderBottleneck(nn.Module):
    """Bottleneck decoder variant: 128^3 resolution."""
    def __init__(self, latent_dim):
        super().__init__()
        # 3 fully-connected layers with output sizes 128, 4, and 128^3 * 4
        self.fc1 = nn.Linear(latent_dim, 128)
        self.fc2 = nn.Linear(128, 4)
        self.fc3 = nn.Linear(4, 128**3 * 4)

    def forward(self, z):
        x = self.fc1(z)
        x = torch.relu(x)
        x = self.fc2(x)
        x = torch.relu(x)
        x = self.fc3(x)
        
        # Reshape to 4D tensor: C x D x D x D
        # C=4, D=128
        x = x.view(4, 128, 128, 128)
        
        # Apply softplus
        x = F.softplus(x)
        return x

def sample_voxel_grid(grid_tensor, x, x0, W):
    """
    Sample from the voxel grid tensor using trilinear interpolation.
    grid_tensor: C x D x D x D tensor
    x: Input 3D coordinate
    x0: Center of the volume
    W: Side size of the volume
    """
    # Scale continuous values in [-1, 1] to grid [1, D]
    # Formula: (x - x0) / (W/2)
    scaled_x = (x - x0) / (W / 2.0)
    
    # Clamp to [-1, 1] to stay within bounds
    scaled_x = torch.clamp(scaled_x, -1.0, 1.0)
    
    # Perform trilinear interpolation
    # Note: PyTorch grid_sample expects normalized coordinates in [-1, 1]
    # and input shape (N, C, D, H, W). We assume grid_tensor is already
    # in the correct format for interpolation.
    # This is a conceptual implementation; actual grid_sample usage
    # requires reshaping and specific coordinate grid generation.
    # [G] Specific grid_sample coordinate generation omitted for brevity,
    # but the core logic is scaling and interpolation.
    
    # Placeholder for actual interpolation step
    # y = grid_sample(grid_tensor.unsqueeze(0), scaled_x.unsqueeze(0).unsqueeze(0).unsqueeze(0))
    # return y.squeeze()
    raise NotImplementedError("Grid sampling implementation depends on specific framework details not provided in sources.")
```

## Subtopics

### Semantic Volume Decoder
A decoder network that outputs a semantic volume $G_S$ from encoded features, used for 3D semantic labeling [S2]. It differs from the standard RGB$\alpha$ decoder by incorporating skip connections from the input grid and using a specific loss function for classification [S2].
*   **Structure:** Replace the initial depth-1 connection $D_1$ with an additional skip connection incorporating the encoded output from the input grid ($G$) [S2]. Followed by a final convolutional output block to generate the semantic volume [S2].
*   **Parameters:** Spatial dimensions $W = H = D = 160$ [S2]. Channel count $C$ is 18 for the Front3D dataset and 20 for the Habitat-Matterport3D dataset [S2].
*   **Loss:** Weighted masked cross-entropy loss, balanced by the inverse log propensity $p_c$ of each class, where propensity is the frequency of the class in the training dataset [S2].
*   **Link:** [NeRF-MAE](https://arxiv.org/abs/2404.01300v3) [S2].

## Operating characteristics
*   **Performance:** The Convolutional 32³ variant runs at 90Hz [S1]. The Convolutional 128³ variant runs at 22Hz [S1]. The Bottleneck 128³ variant runs at 90Hz [S1].
*   **Memory:** The memory requirement is proportional to $C \times D^3$. For $D=128$ and $C=4$, this is $4 \times 128^3$ values [S1].
*   **Limitations:** Voxel grids are limited to representing details as small as a single voxel and are computationally expensive to evaluate and store at high resolution [S1]. They are also wasteful in scenes with much empty space [S1].
*   **Strengths:** Avoids the prohibitive size requirements of MLPs for high-quality reconstructions [S1]. Enables real-time applications by reducing the complexity of per-ray evaluation [S1].

## Background
Voxel Grid Decoders were introduced in the "Neural Volumes" paper [S1]. The method was developed to address the computational bottleneck of evaluating MLPs at every step along each ray in the ray-marching process, which imposes a restrictive upper bound on MLP complexity for real-time applications [S1]. The approach assumes the volume function can be modeled as a discrete 3D grid of voxels, producing an explicit 3D voxel grid as the output tensor of a neural network [S1].

## Sources
[S1] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
[S2] NeRF-MAE: Masked AutoEncoders for Self-Supervised 3D Representation Learning for Neural Radiance Fields, https://arxiv.org/abs/2404.01300v3, fetched 2026-10-26
