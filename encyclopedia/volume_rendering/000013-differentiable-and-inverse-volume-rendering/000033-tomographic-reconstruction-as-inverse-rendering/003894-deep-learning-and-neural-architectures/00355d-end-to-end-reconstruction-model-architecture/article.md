# End-to-End Reconstruction Model Architecture

Also called: Simultaneous Reconstruction and Denoising Model, End-to-end differentiable architecture, Sinogram-Based Tomography Reconstruction, g(r) model

## Overview
This method is an end-to-end differentiable deep learning architecture that directly maps a noisy sinogram into a denoised 3D reconstruction, combining the reconstruction and denoising steps to mitigate noise amplification caused by the ill-conditioning of the Radon transform [S1, unofficial]. It differs from traditional Filtered Back-Projection (FBP) by replacing the fixed linear filtration with a trainable neural network, and from pure post-processing methods by learning the physics of the acquisition geometry implicitly through the differentiable back-projection operator [S1, unofficial]. Unlike [Deep Image Prior Reconstruction](1221-deep-image-prior-reconstruction.md) which optimizes a static network's weights per instance, this model learns a generalizable mapping from sinogram to volume via joint optimization of its parameters [S1, unofficial].

## How it works
The model, denoted as $g(r)$, consists of two U-Net instances ($g_1$ and $g_2$) connected by the Radon back-projection operator $A^T$ [S1, unofficial]. The architecture is defined by the composition:

$$
g(r) = g_2 (A^T g_1 (r))
$$

where $r$ is the noisy sensor reading (sinogram), $g_1$ is a U-Net that processes the sinogram, $A^T$ is the transpose of the discrete Radon transform (back-projection), and $g_2$ is a U-Net that processes the back-projected images to produce the final reconstruction [S1, unofficial].

### Inputs and Outputs
*   **Input:** A noisy sinogram $r$.
*   **Output:** A denoised 3D reconstruction volume.

### Data Structures and Layout
The model comprises two U-Net architectures, $g_1$ and $g_2$. The specific configuration depends on the model size (XXS, XS, S, S-64, M, L, XL, XXL) [S1, unofficial].

**U-Net Block Structure:**
Each U-Net is composed of down-sampling and up-sampling blocks [S1, unofficial].
1.  **Down-sampling Block:** Consists of a strided convolution followed by a sequence of residual blocks [S1, unofficial].
2.  **Up-sampling Block:**
    *   Takes the output of the previous block.
    *   Doubles the resolution using bilinear up-sampling.
    *   Concatenates the result with the skip connection from the corresponding down-sampling block at the same resolution.
    *   Weights the concatenated channels using channel attention [S1, unofficial].
    *   Shrinks the number of channels via a $1 \times 1$ convolution.
    *   Processes the result via a sequence of residual blocks [S1, unofficial].

**Residual Block Channels:**
*   The number of channels in residual blocks is doubled every two strided convolutions [S1, unofficial].
*   The first block of XXS, XS, and S models uses 32 channels.
*   The first block of S-64, M, L, XL, and XXL models uses 64 channels [S1, unofficial].

**Model Sizes and Residual Blocks:**
The number of residual blocks per U-Net block depends on the resolution and model size. Let $N_{res}(size, resolution)$ be the number of residual blocks for a given model size and resolution:

| Resolution | XXS | XS | S | M | L | XL | XXL |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 256 | 1 | 1 | 1 | 2 | 2 | 3 | 5 |
| 128 | 1 | 1 | 1 | 1 | 2 | 3 | 5 |
| 64 | 2 | 1 | 1 | 1 | 2 | 3 | 5 |
| 32 | - | 2 | 2 | 1 | 2 | 3 | 5 |
| 16 | - | - | 2 | 1 | 2 | 3 | 5 |
| 8 | - | - | - | 2 | 3 | 5 | 9 |
| 16 | - | - | - | 2 | 3 | 4 | 6 |
| 32 | - | - | 2 | 2 | 3 | 4 | 6 |
| 64 | - | 2 | 2 | 2 | 3 | 4 | 6 |
| 128 | 2 | 2 | 2 | 2 | 3 | 4 | 6 |
| 256 | 1 | 2 | 2 | 2 | 4 | 4 | 6 |

*Note: "-" indicates the resolution is not applicable or not used for that model size [S1, unofficial].*

**Channel Counts for Back-Projection:**
*   The smallest end-to-end model (XXS for $g_1$, S-64 for $g_2$) uses 16 channels for the Radon back-projection input.
*   Other end-to-end models use 32 channels [S1, unofficial].

### Processing Steps
1.  **Sinogram Embedding ($g_1$ Input):**
    *   The input sinogram $r$ passes through an "Embedding Conv 2D" layer.
    *   The output passes through a U-Net Block.
    *   This sequence is repeated for multiple U-Net Blocks to form the $g_1$ network. The specific number of blocks and their resolutions follow the U-Net architecture defined by the chosen model size (e.g., S-64) [S1, unofficial].
    *   The final convolution of $g_1$ reduces the number of channels to the back-projection channel count (16 or 32) [S1, unofficial].

2.  **Radon Back-Projection:**
    *   The multi-channel output of $g_1$ is transformed into images using the Radon back-projection operator $A^T$.
    *   Each channel of the output from $g_1$ is treated as a separate sinogram and back-projected into an image [S1, unofficial].

3.  **Positional Encoding:**
    *   The output of $A^T$ (shape $(ch, 256, 256)$) is concatenated with a 4-dimensional positional embedding (shape $(4, 256, 256)$) in the first layer of $g_2$.
    *   This positional embedding is processed by a "Pos Features Conv 2D" layer to encode pixel coordinates [S1, unofficial].

4.  **Reconstruction and Denoising ($g_2$):**
    *   The concatenated input (back-projected images + positional features) is processed by a sequence of U-Net Blocks to form the $g_2$ network.
    *   The architecture of $g_2$ follows the same U-Net structure as $g_1$, but with potentially different dimensions (e.g., S-64) [S1, unofficial].
    *   Blocks with multiple inputs (e.g., skip connections) up-sample each input to the largest resolution using bilinear up-sampling, then concatenate all inputs on the channel dimension [S1, unofficial].

### Training
The model is trained end-to-end by jointly optimizing the parameters of $g_1$ and $g_2$ to minimize an image reconstruction loss [S1, unofficial]. The differentiability of the Radon back-projection operator $A^T$ allows gradients to flow from the loss in the image space, through $g_2$, through $A^T$, through $g_1$, and back to the input sinogram $r$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ResidualBlock(nn.Module):
    def __init__(self, in_channels, out_channels, stride=1):
        super(ResidualBlock, self).__init__()
        self.conv1 = nn.Conv2d(in_channels, out_channels, kernel_size=3, stride=stride, padding=1, bias=False)
        self.bn1 = nn.BatchNorm2d(out_channels)
        self.conv2 = nn.Conv2d(out_channels, out_channels, kernel_size=3, stride=1, padding=1, bias=False)
        self.bn2 = nn.BatchNorm2d(out_channels)
        self.relu = nn.ReLU(inplace=True)
        self.shortcut = nn.Sequential()
        if stride != 1 or in_channels != out_channels:
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_channels, out_channels, kernel_size=1, stride=stride, bias=False),
                nn.BatchNorm2d(out_channels)
            )

    def forward(self, x):
        out = self.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        out += self.shortcut(x)
        out = self.relu(out)
        return out

class ChannelAttention(nn.Module):
    def __init__(self, in_channels, reduction=16):
        super(ChannelAttention, self).__init__()
        self.avg_pool = nn.AdaptiveAvgPool2d(1)
        self.fc = nn.Sequential(
            nn.Linear(in_channels, in_channels // reduction, bias=False),
            nn.ReLU(inplace=True),
            nn.Linear(in_channels // reduction, in_channels, bias=False),
            nn.Sigmoid()
        )

    def forward(self, x):
        b, c, _, _ = x.size()
        y = self.avg_pool(x).view(b, c)
        y = self.fc(y).view(b, c, 1, 1)
        return x * y.expand_as(x)

class UNetBlock(nn.Module):
    def __init__(self, in_channels, out_channels, num_res_blocks, is_downsampling=False):
        super(UNetBlock, self).__init__()
        self.is_downsampling = is_downsampling
        if is_downsampling:
            self.conv = nn.Conv2d(in_channels, out_channels, kernel_size=3, stride=2, padding=1, bias=False)
            self.bn = nn.BatchNorm2d(out_channels)
            self.relu = nn.ReLU(inplace=True)
            res_blocks = []
            for _ in range(num_res_blocks):
                res_blocks.append(ResidualBlock(out_channels, out_channels))
            self.res_blocks = nn.Sequential(*res_blocks)
        else:
            self.up = nn.Upsample(scale_factor=2, mode='bilinear', align_corners=False)
            self.attention = ChannelAttention(out_channels * 2)
            self.conv = nn.Conv2d(out_channels * 2, out_channels, kernel_size=1, bias=False)
            self.bn = nn.BatchNorm2d(out_channels)
            self.relu = nn.ReLU(inplace=True)
            res_blocks = []
            for _ in range(num_res_blocks):
                res_blocks.append(ResidualBlock(out_channels, out_channels))
            self.res_blocks = nn.Sequential(*res_blocks)

    def forward(self, x, skip=None):
        if self.is_downsampling:
            out = self.relu(self.bn(self.conv(x)))
            out = self.res_blocks(out)
            return out
        else:
            out = self.up(x)
            if skip is not None:
                # Concatenate with skip connection
                out = torch.cat([out, skip], dim=1)
                # Channel attention
                out = self.attention(out)
                # Shrink channels
                out = self.relu(self.bn(self.conv(out)))
            out = self.res_blocks(out)
            return out

class RadonBackProjection(nn.Module):
    def __init__(self, in_channels, img_size=256):
        super(RadonBackProjection, self).__init__()
        self.in_channels = in_channels
        self.img_size = img_size
        # Note: Actual Radon back-projection requires specific geometry parameters
        # For implementation, this would call a differentiable Radon back-projection function
        # Here we assume a placeholder for the differentiable operator A^T
        pass

    def forward(self, sinogram):
        # sinogram shape: (batch, channels, height, width)
        # Output shape: (batch, channels, img_size, img_size)
        # This step performs the transpose of the discrete Radon transform
        # Implementation detail: Use a library or custom CUDA kernel for differentiable back-projection
        # For the purpose of this article, we assume the operator is differentiable
        return back_project_radon(sinogram, self.img_size)

class EndToEndReconstructionModel(nn.Module):
    def __init__(self, g1_config, g2_config, back_proj_channels=32, img_size=256):
        super(EndToEndReconstructionModel, self).__init__()
        self.g1 = self._build_g1(g1_config, back_proj_channels)
        self.back_proj = RadonBackProjection(back_proj_channels, img_size)
        self.g2 = self._build_g2(g2_config, img_size)

    def _build_g1(self, config, out_channels):
        # g1 processes sinogram
        # Input: Sinogram r
        # Step 1: Embedding Conv 2D
        self.g1_embedding = nn.Conv2d(1, config['init_channels'], kernel_size=3, padding=1)
        
        # Step 2: Sequence of U-Net Blocks
        # The architecture involves multiple blocks with decreasing resolution
        # For simplicity, we assume a standard U-Net structure with skip connections
        # The exact number of blocks and channels is defined by config
        # This is a simplified representation; actual implementation requires detailed block stacking
        self.g1_blocks = nn.ModuleList()
        # ... build blocks based on config ...
        
        # Final convolution to reduce channels to back_proj_channels
        self.g1_final_conv = nn.Conv2d(config['last_channels'], out_channels, kernel_size=1)
        return self.g1_blocks

    def _build_g2(self, config, img_size):
        # g2 processes back-projected images
        # Input: Back-projected images + Positional Embedding
        # Step 1: Pos Features Conv 2D on concatenated input
        # Input shape: (ch, 256, 256) + (4, 256, 256) -> (ch+4, 256, 256)
        self.g2_pos_conv = nn.Conv2d(config['init_channels'] + 4, config['init_channels'], kernel_size=3, padding=1)
        
        # Step 2: Sequence of U-Net Blocks
        self.g2_blocks = nn.ModuleList()
        # ... build blocks based on config ...
        
        return self.g2_blocks

    def forward(self, r):
        # r: noisy sinogram
        # 1. g1 processing
        x = self.g1_embedding(r)
        # ... pass through g1 blocks ...
        x = self.g1_final_conv(x)
        
        # 2. Radon Back-Projection
        # x shape: (batch, back_proj_channels, sinogram_h, sinogram_w)
        # Output shape: (batch, back_proj_channels, img_size, img_size)
        images = self.back_proj(x)
        
        # 3. Positional Embedding
        # Create positional embedding (4 channels)
        # This is a placeholder for the actual positional encoding generation
        pos_embed = self._generate_positional_embedding(images.shape[0], images.shape[1], images.shape[2], images.shape[3])
        
        # Concatenate images and pos_embed
        x = torch.cat([images, pos_embed], dim=1)
        
        # 4. g2 processing
        x = self.g2_pos_conv(x)
        # ... pass through g2 blocks ...
        
        return x

    def _generate_positional_embedding(self, batch_size, channels, height, width):
        # Generate 4-channel positional embedding
        # This is a simplified placeholder; actual implementation should encode pixel coordinates
        # For example, using sine/cosine functions or learned embeddings
        # Shape: (batch_size, 4, height, width)
        return torch.zeros(batch_size, 4, height, width)
```

## Subtopics

### [ResNet-Based CT Reconstruction](8264-resnet-based-ct-reconstruction.md)
ResNet-Based CT Reconstruction is a learned post-processing method that maps noisy filtered backprojection (FBP) reconstructions directly to ground truth images, learning to suppress artifacts and noise. It differs from pure post-processing methods by using the physics of the acquisition geometry to guide the denoising and artifact suppression. Unlike Gradient-Aware ResNet Reconstruction, it treats the input as a static noisy image rather than conditioning on the explicit Radon matrix.

## Operating characteristics
*   **Time and Memory Complexity:** The complexity is dominated by the U-Net architectures and the Radon back-projection operator. The number of parameters is less than existing approaches, leading to faster inference and training times [S1, unofficial]. The exact complexity depends on the model size (XXS to XXL) and the resolution of the input sinogram.
*   **Measured Performance:** The method produces more accurate reconstructions than existing approaches while using fewer parameters and less time [S1, unofficial].
*   **Strengths:** Combines reconstruction and denoising to mitigate noise amplification from the ill-conditioning of the Radon transform. The multi-channel output of $g_1$ allows richer information flow. End-to-end training leverages the differentiability of the back-projection operator.
*   **Limitations:** Requires training data for supervised learning. The performance depends on the quality and quantity of the training data. The method assumes a specific acquisition geometry for the Radon transform.
*   **When to Use:** Suitable for tomographic reconstruction tasks where noise reduction is critical and computational efficiency is desired.
*   **When Not to Use:** Not suitable for unsupervised settings or when the acquisition geometry differs significantly from the training data.

## Background
The method was proposed by Matteo Ronchetti and Davide Bacciu from the University of Pisa in the paper "Generative Tomography Reconstruction" (arXiv:2010.14933v2, 25 Nov 2020) [S1, unofficial]. It addresses the prototypical linear inverse problem of reconstructing Computed Tomography (CT) images. The approach is inspired by Filtered Back-Projection (FBP) but replaces the fixed linear filtration with trainable neural networks to handle the ill-conditioning of the Radon transform and reduce noise amplification [S1, unofficial].

## Sources
[S1] Generative Tomography Reconstruction, https://core.ac.uk/works/91148923, fetched 2026-10-31 (unofficial)
