# Volumetric Ambient Occlusion prediction

Also called: Deep Volumetric Ambient Occlusion, Engel and Ropinski

## Overview
Deep Volumetric Ambient Occlusion (DVAO) is a deep learning-based technique that predicts per-voxel ambient occlusion (AO) volumes for direct volume rendering (DVR) [S1]. It solves the problem of computing realistic volumetric illumination efficiently by training a 3D convolutional neural network to map input scalar field data and transfer functions to an AO volume [S1]. Unlike traditional numerical integration methods, DVAO predicts the AO field once, which is then sampled via texture lookup during rendering, supporting real-time interaction when the transfer function changes [S1]. It is distinguished from sibling methods like [Local Ambient Occlusion (LAO)](link-to-lao) by its use of a global encoder-decoder architecture that incorporates transfer function information implicitly or explicitly, rather than computing occlusion locally per ray [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field volume $V$ of resolution $128 \times 128 \times 128$ and a transfer function $T$ [S1]. The transfer function can be represented implicitly (as an opacity volume derived from $V$) or explicitly (as a low-dimensional feature vector extracted by a TF Extractor) [S1].
*   **Output:** An ambient occlusion volume $A$ of resolution $128 \times 128 \times 128$, where each voxel contains a scalar value in $[0, 1]$ representing the occlusion factor [S1].

**Data Structures:**
1.  **Input Volume:** A 3D tensor of shape $(128, 128, 128, 1)$ containing normalized intensity values [S1].
2.  **Transfer Function (Implicit):** A 3D tensor of shape $(128, 128, 128, 1)$ representing opacity, computed by applying $T$ to the input volume [S1].
3.  **Transfer Function (Explicit):** A 1D feature vector of fixed length, output by the TF Extractor [S1].
4.  **Network Weights:** Parameters of the 3D ConvBlocks and TF Extractor [S1].

**Ground Truth Generation (Training Data):**
To train the network, ground truth AO volumes are generated using Monte Carlo ray casting [S1].
1.  For each voxel center $\mathbf{x}$, cast $N_r = 512$ rays in random directions $\boldsymbol{\omega} \sim \Omega$ [S1].
2.  Limit ray length to $D = 0.1 \times \text{volume diameter}$ to avoid fully shadowed regions [S1].
3.  Compute occlusion using the emission and absorption model [S1]:
    $$
    \text{AO}(\mathbf{x}) = \int_{\Omega} p(\boldsymbol{\omega}) \left( 1 - \int_{\mathbf{x}+\boldsymbol{\omega}\varepsilon}^{\mathbf{x}+\boldsymbol{\omega}D} \tau(t) e^{-\int_{\mathbf{x}}^{t} \tau(s) ds} dt \right) d\boldsymbol{\omega}
    $$
    where $\tau$ is the extinction coefficient (opacity), $p(\boldsymbol{\omega})$ is the probability of the sampled angle, $\varepsilon$ is a small offset from the voxel center to prevent self-occlusion, and $q(s)$ is the light contribution (assumed uniform/constant in the simplified formulation provided in notes, effectively integrating opacity) [S1]. Note: The source formula [S1] Eq. 1 uses $\tau(t)$ as the integrand for the inner integral's exponential term and the outer term, implying standard volumetric occlusion integration.

**Neural Network Architecture:**
The core network is a 3D U-Net-like encoder-decoder [S1].

1.  **Encoder:**
    *   Input: $128^3 \times 1$.
    *   Consists of 7 ConvBlocks [S1].
    *   Each ConvBlock contains: 3D Convolution $\rightarrow$ Instance Normalization (IN) $\rightarrow$ Mish Activation $\rightarrow$ 3D Convolution $\rightarrow$ IN $\rightarrow$ Mish Activation [S1].
    *   Between ConvBlocks, the feature volume is downsampled by half in each spatial dimension using trilinear interpolation, and the number of filters is doubled [S1].
    *   Filter counts: 16, 32, 64, 128, 256, 512, 512 [S1].
    *   The latent representation is $1^3 \times 512$ [S1].

2.  **Decoder:**
    *   Input: Latent $1^3 \times 512$.
    *   Consists of 7 ConvBlocks [S1].
    *   Upsampling is performed using trilinear interpolation before each ConvBlock [S1].
    *   Filter counts: 512, 256, 128, 64, 32, 16, 1 [S1].
    *   Skip connections concatenate encoder features to decoder features at each resolution level [S1].

3.  **Transfer Function Extractor (Explicit Representation Only):**
    *   Input: Discretized 1D texture $T$ of resolution $R=4096$ [S1].
    *   Consists of three strided 1D Convolution layers (with normalization and activation), an average pooling layer, and an MLP [S1].
    *   Output: A fixed-length TF descriptor vector [S1].

**Training:**
*   **Dataset:** CQ500 (CT scans of human heads) [S1]. Volumes are cropped to the largest non-transparent subvolume and resized to $128^3$ [S1].
*   **Transfer Function Generation:** Random piece-wise linear opacity transfer functions using 1-5 non-overlapping trapezoids centered around volume histogram peaks [S1]. Trapezoid parameters are sampled from uniform distributions: $h_{top} \sim U(h_{min}, h_{max})$, $w_{outer} \sim U(w_{min}, w_{max})$, $h_{bottom} \sim U(h_{min}, h_{top})$, $w_{inner} \sim U(w_{min}, w_{outer})$ with $h_{min}=0.1, h_{max}=0.9, w_{min}=0.01, w_{max}=0.1$ [S1].
*   **Loss Function:** Structural Similarity Index (SSIM) combined with Mean Squared Error (MSE) or Mean Absolute Error (MAE) [S1]. 2D image-based SSIM is used for efficiency [S1].
*   **Precision:** Mixed precision (16-bit floating point) is recommended [S1].

**Inference Pipeline:**
1.  Pre-compute the AO volume $A$ using the trained network given the current volume $V$ and transfer function $T$ [S1].
2.  Perform standard Direct Volume Ray Casting on the full-resolution volume for color and opacity [S1].
3.  Sample the pre-computed AO volume $A$ (at $128^3$ resolution) during ray casting to modulate the final color [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class Mish(nn.Module):
    def forward(self, x):
        return x * torch.tanh(F.softplus(x))

class ConvBlock(nn.Module):
    def __init__(self, in_channels, out_channels):
        super().__init__()
        self.conv1 = nn.Conv3d(in_channels, out_channels, kernel_size=3, padding=1)
        self.norm1 = nn.InstanceNorm3d(out_channels)
        self.conv2 = nn.Conv3d(out_channels, out_channels, kernel_size=3, padding=1)
        self.norm2 = nn.InstanceNorm3d(out_channels)
        self.act = Mish()

    def forward(self, x):
        x = self.conv1(x)
        x = self.norm1(x)
        x = self.act(x)
        x = self.conv2(x)
        x = self.norm2(x)
        x = self.act(x)
        return x

class TFExtractor(nn.Module):
    def __init__(self, tf_resolution=4096, descriptor_dim=64):
        super().__init__()
        self.conv1 = nn.Conv1d(1, 32, kernel_size=5, stride=2, padding=2)
        self.norm1 = nn.InstanceNorm1d(32)
        self.conv2 = nn.Conv1d(32, 64, kernel_size=5, stride=2, padding=2)
        self.norm2 = nn.InstanceNorm1d(64)
        self.conv3 = nn.Conv1d(64, 128, kernel_size=5, stride=2, padding=2)
        self.norm3 = nn.InstanceNorm1d(128)
        self.pool = nn.AdaptiveAvgPool1d(1)
        self.mlp = nn.Sequential(
            nn.Linear(128, 64),
            nn.ReLU(),
            nn.Linear(64, descriptor_dim)
        )

    def forward(self, tf_texture):
        x = self.conv1(tf_texture)
        x = self.norm1(x)
        x = torch.relu(x)
        x = self.conv2(x)
        x = self.norm2(x)
        x = torch.relu(x)
        x = self.conv3(x)
        x = self.norm3(x)
        x = torch.relu(x)
        x = self.pool(x)
        x = x.view(x.size(0), -1)
        x = self.mlp(x)
        return x

class DVAO(nn.Module):
    def __init__(self, use_explicit_tf=False, tf_descriptor_dim=512):
        super().__init__()
        self.use_explicit_tf = use_explicit_tf
        self.tf_extractor = None
        if use_explicit_tf:
            self.tf_extractor = TFExtractor(descriptor_dim=tf_descriptor_dim)
        
        self.enc1 = ConvBlock(1, 16)
        self.enc2 = ConvBlock(16, 32)
        self.enc3 = ConvBlock(32, 64)
        self.enc4 = ConvBlock(64, 128)
        self.enc5 = ConvBlock(128, 256)
        self.enc6 = ConvBlock(256, 512)
        self.enc7 = ConvBlock(512, 512)
        
        self.dec7 = ConvBlock(512 + 512, 512)
        self.dec6 = ConvBlock(512 + 256, 256)
        self.dec5 = ConvBlock(256 + 128, 128)
        self.dec4 = ConvBlock(128 + 64, 64)
        self.dec3 = ConvBlock(64 + 32, 32)
        self.dec2 = ConvBlock(32 + 16, 16)
        self.dec1 = ConvBlock(16 + 1, 1)
        
        self.sigmoid = nn.Sigmoid()

    def forward(self, volume, tf_representation=None):
        tf_descriptor = None
        if self.use_explicit_tf and tf_representation is not None:
            tf_descriptor = self.tf_extractor(tf_representation)
            tf_proj = tf_descriptor.view(tf_descriptor.size(0), 512, 1, 1, 1)
        
        x1 = self.enc1(volume)
        x = F.interpolate(x1, scale_factor=0.5, mode='trilinear', align_corners=False)
        x2 = self.enc2(x)
        x = F.interpolate(x2, scale_factor=0.5, mode='trilinear', align_corners=False)
        x3 = self.enc3(x)
        x = F.interpolate(x3, scale_factor=0.5, mode='trilinear', align_corners=False)
        x4 = self.enc4(x)
        x = F.interpolate(x4, scale_factor=0.5, mode='trilinear', align_corners=False)
        x5 = self.enc5(x)
        x = F.interpolate(x5, scale_factor=0.5, mode='trilinear', align_corners=False)
        x6 = self.enc6(x)
        x = F.interpolate(x6, scale_factor=0.5, mode='trilinear', align_corners=False)
        x7 = self.enc7(x)
        
        if tf_descriptor is not None:
            x7 = x7 + tf_proj
        
        x = F.interpolate(x7, scale_factor=2, mode='trilinear', align_corners=False)
        x = torch.cat([x, x6], dim=1)
        x = self.dec7(x)
        
        x = F.interpolate(x, scale_factor=2, mode='trilinear', align_corners=False)
        x = torch.cat([x, x5], dim=1)
        x = self.dec6(x)
        
        x = F.interpolate(x, scale_factor=2, mode='trilinear', align_corners=False)
        x = torch.cat([x, x4], dim=1)
        x = self.dec5(x)
        
        x = F.interpolate(x, scale_factor=2, mode='trilinear', align_corners=False)
        x = torch.cat([x, x3], dim=1)
        x = self.dec4(x)
        
        x = F.interpolate(x, scale_factor=2, mode='trilinear', align_corners=False)
        x = torch.cat([x, x2], dim=1)
        x = self.dec3(x)
        
        x = F.interpolate(x, scale_factor=2, mode='trilinear', align_corners=False)
        x = torch.cat([x, x1], dim=1)
        x = self.dec2(x)
        
        x = F.interpolate(x, scale_factor=2, mode='trilinear', align_corners=False)
        x = torch.cat([x, volume], dim=1)
        x = self.dec1(x)
        
        return self.sigmoid(x)
```

## Subtopics

### Implicit Transfer Function Representation
Applies the transfer function directly to the input volume to create an opacity volume, preserving volumetric shape and allowing combination with any feature tensor [S1]. It is easier for the network to learn but requires running the full network during inference [S1].

### Explicit Transfer Function Representation
Uses a TF Extractor (1D CNN) to produce a low-dimensional feature vector (descriptor) from the transfer function, which is injected into the network [S1]. It is analogous to late-fusion and can save inference time if the TF doesn't change, but requires separate extraction and may be less successful in experiments [S1].

## Operating characteristics
*   **Time Complexity:** Inference time is dominated by the 3D CNN forward pass [S1]. On the tested hardware, DVAO takes approximately 411 ms per volume [S1]. This is significantly slower than Local Ambient Occlusion (LAO) with few rays (e.g., 111 ms for LAO-32) but comparable to LAO with many rays (417 ms for LAO-190) [S1].
*   **Memory Complexity:** VRAM usage is high due to the 3D CNN weights and intermediate feature maps [S1]. DVAO uses 1322 MiB during inference, compared to 42 MiB for LAO [S1].
*   **Performance:** Achieves SSIM scores of 0.875 on the test set [S2, unofficial]. Generalizes to other modalities (MRI SSIM 0.751, EM SSIM 0.869, mouse micro-CT SSIM 0.786) [S1].
*   **Strengths:** Produces high-quality AO with fewer artifacts than LAO at similar quality levels [S1]. Supports real-time interaction when the transfer function changes, as only the AO volume needs to be re-predicted [S1].
*   **Limitations:** Limited to $128^3$ resolution due to GPU memory constraints of 3D CNNs [S1]. High VRAM usage and inference time compared to traditional numerical methods like LAO [S1].
*   **When to use:** When high-quality volumetric illumination is required and the transfer function changes infrequently, allowing for pre-computation [S1]. Suitable for interactive visualization where the cost of re-computing AO numerically is prohibitive [S1].
*   **When not to use:** When memory is constrained or real-time performance for every frame (including volume changes) is critical without pre-computation [S1]. Not suitable for very high-resolution volumes without significant architectural changes [S1].

## Background
DVAO was published by Engel and Ropinski in IEEE Transactions on Visualization and Computer Graphics in 2020 [S1]. It was funded by the Deutsche Forschungsgemeinschaft (DFG) under grant 391107954 (Inviwo) [S1]. The method addresses the limitations of traditional numerical ambient occlusion in volume rendering by leveraging deep learning to predict the AO field, building on the principles of [Neural volume rendering](1576-neural-volume-rendering.md) [S1]. It is particularly relevant in medical imaging, where transfer functions are frequently adjusted, and pre-computing AO can accelerate interaction [S1].

## Sources
[S1] Deep Volumetric Ambient Occlusion, https://arxiv.org/abs/2008.08345v2, fetched 2026-10-26
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
