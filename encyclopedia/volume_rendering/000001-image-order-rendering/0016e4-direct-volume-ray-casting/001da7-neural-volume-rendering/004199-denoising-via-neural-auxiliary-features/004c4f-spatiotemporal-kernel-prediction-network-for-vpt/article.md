# Spatiotemporal Kernel Prediction Network for VPT

Also called: Dual-stream CNN denoiser, MSKPL denoiser

## Overview
A deep learning denoiser for Volume Path Tracing (VPT) image sequences that uses a dual-stream encoder to process G-buffers and radiance separately, and a Multi-Head Spatiotemporal Kernel Prediction Layer (MSKPL) to predict per-pixel temporal and spatial filter kernels for adaptive denoising [S1, unofficial]. It builds on [Denoising via Neural Auxiliary Features](10536-denoising-via-neural-auxiliary-features.md) by explicitly decoupling the encoding of low-noise auxiliary features from high-noise radiance to reduce color bias and improve learning efficiency [S1, unofficial]. Unlike unified encoders, this separation leverages the color-independence of G-buffers to guide the denoising of radiance without introducing chromatic artifacts [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A tuple $X = (\text{G-buffers}, \text{Radiance})$ for the current frame $n$, and the denoised output from the previous frame $n-1$ reprojected to the current frame.
    *   **G-buffers**: 4-channel buffer containing Texture, illumination, alpha, and world position [S1, unofficial].
    *   **Radiance**: The noisy volumetric radiance image from the current frame [S1, unofficial].
*   **Output**: The denoised radiance image $x_n''$ for frame $n$ [S1, unofficial].

**Data Structures**
*   **Feature Maps**: Intermediate representations with dimensions $(W, H)$, where $W$ and $H$ are the image width and height [S1, unofficial].
*   **Kernels**: Per-pixel temporal kernel $K_h$ and spatial kernel $K_c$, predicted by the network [S1, unofficial].
*   **Network Parameters**: $\theta$ represents the trainable parameters of the network [S1, unofficial].

**Numbered Steps**

1.  **Reprojection**: Compute the reprojection of the historical denoised frame $n-1$ onto the current frame $n$ using the motion vector $v$. The reprojected value $x_{np}$ at spatial location $(i,j)$ is calculated as:
    $$x_{np}(i,j) = x_{n-1}'(i+v, j+v)$$
    *Note: The source formula contains a typo `j+v,j`; the standard interpretation for 2D spatial reprojection is $(i+v, j+v)$ [S1, unofficial].*
    The reprojection utilizes the nearest sample point along each viewing ray that satisfies an opacity threshold condition $\alpha > 0.8$, determined through ray-marching sampling. If no sample meets this criterion, the sample with the maximum opacity along the ray is selected [S1, unofficial].

2.  **Dual-Stream Encoding**: The network consists of two parallel encoders, $E_a$ (for radiance) and $E_b$ (for G-buffers), and a decoder $D$. Each encoder block comprises two $3 \times 3$ dilated convolution layers, LeakyReLU layers, and max-pooling layers [S1, unofficial]. Let $L$ be the number of blocks in the encoder and decoder (set to $L=3$) [S1, unofficial].
    *   **G-buffer Stream ($E_b$)**: Compute the output $h_i$ of the $i$-th layer:
        $$h_i = \text{LeakyReLU}(E_b(h_{i-1}))$$
    *   **Radiance Stream ($E_a$)**: Compute the output $x_i$ of the $i$-th layer using the previous radiance output and the max-pooled output from the G-buffer stream:
        $$x_i = \text{LeakyReLU}(x_{i-1}, \text{maxpool}(h_i))$$
    The low-noise G-buffers preserve edge and detail information, guiding the high-noise radiance encoder [S1, unofficial].

3.  **Cascade Fusion**: Fuse the feature maps from the dual-stream encoder using the Cascade Fusion Module (CFM). The CFM consists of convolutional layers, batch normalization layers, and pixel-wise convolutional layers to standardize the distribution of the dual-channel feature maps and enhance information integration [S1, unofficial].

4.  **Decoding**: Compute the output $d_i$ of the $i$-th decoder layer using the upsampled previous decoder output and the corresponding encoder output from the reverse path:
    $$d_i = D(\text{upsample}(d_{i-1}) \oplus x_{L-i})$$
    Here, $\oplus$ denotes concatenation or addition (source uses $\oplus$) [S1, unofficial]. This residual connection improves generalization and reduces information loss [S1, unofficial].

5.  **Kernel Prediction**: Input the reprojected historical features, current frame radiance, and G-buffers into the network $S$ (the MSKPL) to generate the temporal kernel $K_h$ and spatial kernel $K_c$:
    $$K_h, K_c = S(x_{n-1}', x_n, \text{gbuffers})$$
    The output layer uses a softmax activation function [S1, unofficial].

6.  **Spatial-Temporal Filtering**: Apply the predicted kernels to the reprojected historical frame and the current noisy frame to obtain the denoised volume data $x_n'$:
    $$x_n'(i,j) = K_h(i,j) * x_{np}(i,j) + K_c(i,j) * x_n(i,j)$$
    where $*$ denotes convolution [S1, unofficial].

7.  **Background Blending**: Blend the denoised volume data $x_n'$ with the background $B$ using the alpha map $A$:
    $$x_n''(i,j) = A(i,j) \times x_n'(i,j) + (1 - A(i,j)) \times B(i,j)$$
    This step is performed last because volume data contain numerous semi-transparent structures [S1, unofficial].

**Parameters**
*   $L$: Number of blocks in encoder and decoder. Typical value: $L=3$ [S1, unofficial].
*   Opacity threshold for reprojection: $0.8$ [S1, unofficial].

**Pitfalls and Edge Cases**
*   **Kernel Polarization**: The softmax activation can cause kernel values to approach 0 or 1 during training, preventing effective weight learning. Solution: Add dropout regularization after the softmax layer [S1, unofficial].
*   **Rapid Motion/Illumination Changes**: If the camera moves rapidly, illumination changes abruptly, or the transfer function varies quickly, historical frames become unusable. The method may fail to utilize historical information effectively in these scenarios [S1, unofficial].
*   **Transparent Regions**: In volume rendering, if a ray passes entirely through transparent regions, the historical frame failure rate increases, potentially degrading temporal stability [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DualStreamEncoder(nn.Module):
    def __init__(self, L=3):
        super().__init__()
        self.L = L
        # Encoder blocks for G-buffer (Eb) and Radiance (Ea)
        # Each block: 2x 3x3 dilated conv, LeakyReLU, MaxPool
        self.Eb_blocks = nn.ModuleList([
            nn.Sequential(
                nn.Conv2d(in_channels=4, out_channels=64, kernel_size=3, dilation=1),
                nn.LeakyReLU(),
                nn.Conv2d(in_channels=64, out_channels=64, kernel_size=3, dilation=2),
                nn.LeakyReLU(),
                nn.MaxPool2d(kernel_size=2)
            ) for _ in range(L)
        ])
        self.Ea_blocks = nn.ModuleList([
            nn.Sequential(
                nn.Conv2d(in_channels=3, out_channels=64, kernel_size=3, dilation=1),
                nn.LeakyReLU(),
                nn.Conv2d(in_channels=64, out_channels=64, kernel_size=3, dilation=2),
                nn.LeakyReLU(),
                nn.MaxPool2d(kernel_size=2)
            ) for _ in range(L)
        ])

    def forward(self, g_buffers, radiance):
        # g_buffers: (B, 4, H, W)
        # radiance: (B, 3, H, W)
        
        h = g_buffers
        x = radiance
        encoder_outputs = []
        
        for i in range(self.L):
            # G-buffer stream
            h = self.Eb_blocks[i](h)
            # Radiance stream: uses previous x and maxpool(h)
            # Note: Source formula xi = LeakyRelu(xi-1, maxpool(hi)) implies concatenation or addition
            # Assuming concatenation based on standard dual-stream practices, 
            # but source notation is ambiguous. We assume concatenation along channel dim.
            h_pooled = F.max_pool2d(h, kernel_size=2)
            x_input = torch.cat([x, h_pooled], dim=1)
            x = self.Ea_blocks[i](x_input)
            encoder_outputs.append(x)
            
        return encoder_outputs

class CascadeFusionModule(nn.Module):
    def __init__(self):
        super().__init__()
        # CFM: Conv, BatchNorm, Pixel-wise Conv
        self.conv = nn.Sequential(
            nn.Conv2d(in_channels=128, out_channels=64, kernel_size=1), # Pixel-wise conv
            nn.BatchNorm2d(64),
            nn.Conv2d(in_channels=64, out_channels=64, kernel_size=3, padding=1),
            nn.BatchNorm2d(64),
            nn.LeakyReLU()
        )

    def forward(self, features):
        # features: list of encoder outputs from dual stream
        # Assuming fusion of corresponding levels
        fused = torch.cat(features, dim=1)
        return self.conv(fused)

class KernelPredictor(nn.Module):
    def __init__(self):
        super().__init__()
        # Network S to predict Kh and Kc
        self.predictor = nn.Sequential(
            nn.Conv2d(in_channels=128, out_channels=64, kernel_size=3, padding=1),
            nn.LeakyReLU(),
            nn.Conv2d(in_channels=64, out_channels=2, kernel_size=1) # 2 channels: Kh, Kc
        )
        self.dropout = nn.Dropout(p=0.1) # Mitigate kernel polarization

    def forward(self, fused_features):
        kh_kc = self.predictor(fused_features)
        # Apply softmax to ensure weights sum to 1 (or appropriate normalization)
        kh_kc = F.softmax(kh_kc, dim=1)
        kh_kc = self.dropout(kh_kc)
        return kh_kc

class Denoiser(nn.Module):
    def __init__(self, L=3):
        super().__init__()
        self.encoder = DualStreamEncoder(L=L)
        self.cfm = CascadeFusionModule()
        self.decoder = nn.Sequential(
            nn.ConvTranspose2d(in_channels=64, out_channels=3, kernel_size=2, stride=2),
            nn.Conv2d(in_channels=3, out_channels=3, kernel_size=1)
        )
        self.kernel_pred = KernelPredictor()

    def forward(self, g_buffers, radiance, historical_denoised, motion_vectors, alpha_map, background):
        """
        g_buffers: (B, 4, H, W)
        radiance: (B, 3, H, W)
        historical_denoised: (B, 3, H, W) - already reprojected
        motion_vectors: (B, 2, H, W) - v_x, v_y
        alpha_map: (B, 1, H, W)
        background: (B, 3, H, W)
        """
        # Step 1: Dual-Stream Encoding
        encoder_outs = self.encoder(g_buffers, radiance)
        
        # Step 2: Cascade Fusion
        fused_features = self.cfm(encoder_outs)
        
        # Step 3: Kernel Prediction
        kh_kc = self.kernel_pred(fused_features)
        Kh = kh_kc[:, 0:1, :, :] # Temporal kernel
        Kc = kh_kc[:, 1:2, :, :] # Spatial kernel
        
        # Step 4: Spatial-Temporal Filtering
        # Apply kernels to historical and current radiance
        # Note: Kh and Kc are per-pixel weights. 
        # The source formula implies a weighted sum, not necessarily convolution 
        # if Kh/Kc are scalars per pixel. However, source says "* denotes convolution".
        # If Kh/Kc are single values per pixel, it's element-wise multiplication.
        # If they are kernels, we convolve. Given "per-pixel ... kernels", 
        # and the formula structure, it likely means element-wise weighting 
        # or small local convolution. We assume element-wise for per-pixel weights.
        
        # Reprojection is done externally or via grid_sample if motion vectors are available.
        # Assuming historical_denoised is already reprojected x_np.
        
        denoised_volume = Kh * historical_denoised + Kc * radiance
        
        # Step 5: Background Blending
        denoised_final = alpha_map * denoised_volume + (1 - alpha_map) * background
        
        return denoised_final
```

## Subtopics

### [Weighted Recursive Least Squares (wRLS)](13229-weighted-recursive-least-squares-wrls.md)
Weighted Recursive Least Squares (wRLS) is a spatiotemporal denoising method for Volumetric Path Tracing (VPT) that adapts the Recursive Least Squares (RLS) algorithm to handle heterogeneous noise in low-sample rendering. Unlike standard RLS which relies on noise-free G-buffers as predictors, wRLS uses a temporally stable feature derived from an exponentially weighted history buffer and assigns weights to samples based on their variance, thereby reducing spike noise while maintaining temporal stability. It builds on [Spatiotemporal Kernel Prediction Network for VPT](13214-spatiotemporal-kernel-prediction-network-for-vpt.md) by providing a non-learning, linear-algebraic alternative for temporal coherence in VPT sequences.

## Operating characteristics
*   **Time Complexity**: Real-time capable due to the feed-forward nature of the CNN and the efficiency of the dual-stream encoder-decoder structure [S1, unofficial].
*   **Memory Complexity**: Requires storage for G-buffers (4 channels), radiance, alpha map, background, and the reprojected historical frame [S1, unofficial].
*   **Performance**: Achieves real-time cinematic rendering quality in AR/VR applications with high temporal stability [S1, unofficial].
*   **Strengths**: Separates G-buffer and radiance encoding to reduce color bias and improve learning efficiency; handles semi-transparent structures via background blending [S1, unofficial].
*   **Limitations**: Struggles with rapid camera motion, abrupt illumination changes, or fast transfer function variations where historical frames become unusable; prone to kernel polarization without dropout regularization [S1, unofficial].
*   **When to use**: Low-sample VPT rendering where real-time performance and temporal stability are critical, and scene dynamics are moderate [S1, unofficial].
*   **When not to use**: Scenes with extreme dynamic changes or fully transparent rays where reprojection fails frequently [S1, unofficial].

## Background
The method was proposed by Xu et al. to enable real-time volume-rendering denoising for VPT-based DVR (Direct Volume Ray Casting) in AR/VR applications [S1, unofficial]. It addresses the need for high-quality, temporally stable rendering with low sample counts by decoupling the denoising process into spatial and temporal components guided by auxiliary features [S1, unofficial].

## Sources
[S1] Real-Time Volume-Rendering Image Denoising Based on ... - NIH, https://pmc.ncbi.nlm.nih.gov/articles/PMC12028196/, fetched 2026-10-26 (unofficial)
