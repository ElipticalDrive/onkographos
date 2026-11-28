# Gradient-Aware Super Sampling (GASS)

Also called: GASS module, Gradient-Aware Interpolation

## Overview
Gradient-Aware Super Sampling (GASS) is a post-rendering module that leverages the continuous differentiability of 3D Gaussian Splatting (3DGS) to extract image gradients, which guide a refinement network to enable high-fidelity super sampling from low-resolution inputs [S1]. It addresses aliasing artifacts and detail loss inherent in standard 3DGS rendering by combining low-resolution frame intensities with their spatial gradients and historical features [S1]. Unlike UpscaleGS, which uses image-space gradients to fit a bicubic spline function, GASS employs a network to leverage geometric and historical priors for more robust interpolation [S1].

## How it works
GASS consists of two main components: Gradient-aware interpolation and Geometry-temporal recurrent refinement [S1].

### Inputs and Outputs
**Inputs:**
*   Low-resolution rendered image $I$ [S1].
*   Image-space gradients $G$ (computed from $I$) [S1].
*   Depth map $D$ [S1].
*   Normal-view inner product map $N_v$ [S1].
*   Warped historical feature $\tilde{h}$ from the previous frame [S1].

**Outputs:**
*   High-resolution image $\hat{I}$ [S1].
*   Updated historical feature $h$ [S1].

### Data Structures
*   **Low-Resolution Image $I$**: 2D array of pixel intensities.
*   **Gradients $G$**: 2D arrays for $\frac{\partial I}{\partial x}$ and $\frac{\partial I}{\partial y}$.
*   **Depth Map $D$**: 2D array of depth values.
*   **Normal-View Map $N_v$**: 2D array of dot products between surface normals and view directions.
*   **Feature Map $F$**: 3D tensor with height/width matching $I$ and channels $C_F$. The number of channels $C_F$ must exceed $\min\{S^2, 27\}$ to ensure the coefficient matrix $M$ has full column or row rank [S1].
*   **Historical Feature $\tilde{h}$**: 3D tensor aligned to the current frame via warping [S1].

### Gradient Computation
The rendered image $I(x, y)$ is differentiable with respect to 2D spatial coordinates $(x, y)$ [S1]. The gradients are computed by blending weighted per-Gaussian gradients [S1]. For $N$ Gaussians, the x-gradient is:

$$
\frac {\partial I(x, y)}{\partial x}=\sum _{i=1}^N c_i\left(\frac {\partial T_i}{\partial x} \alpha _i+T_i \frac {\partial \alpha _i}{\partial x}\right)
$$

where $c_i$ is the view-dependent color, $\alpha_i$ is the opacity, and $T_i$ is the transmittance [S1]. The differential of transmittance $T_i$ with respect to $x$ is recursively expanded as:

$$
\frac {\partial T_i}{\partial x} = - \frac {\partial \alpha _{i-1}}{\partial x}T_{i-1} + (1 - \alpha _{i-1}) \frac {\partial T_{i-1}}{\partial x}
$$

The y-gradient $\frac {\partial I(x, y)}{\partial y}$ follows an analogous form [S1].

### Gradient-Aware Interpolation
This step maps a low-resolution block to a high-resolution block using a bivariate polynomial constrained by function values and derivatives [S1].

1.  **Block Definition**: For each pixel $(x_0, y_0)$ in $I$, consider it and its eight adjacent pixels to form a $3 \times 3$ block of interpolation nodes [S1].
2.  **Polynomial Constraints**: A bivariate polynomial $f(x, y)$ is defined such that for $i, j \in \{-1, 0, 1\}$:
    $$
    \begin{aligned}
    f(x_0+i, y_0+j) &= I(x_0+i, y_0+j) \\
    \frac {\partial }{\partial x}f(x_0+i, y_0+j) &= \frac {\partial }{\partial x}I(x_0+i, y_0+j) \\
    \frac {\partial }{\partial y}f(x_0+i, y_0+j) &= \frac {\partial }{\partial y}I(x_0+i, y_0+j)
    \end{aligned}
    $$
    [S1]
3.  **Mapping**: Under an $S \times S$ super-sampling setting, the $3 \times 3$ block in $I$ centered at $(x_0, y_0)$ maps to an $S \times S$ block in $\hat{I}$ centered at $(\hat{x}_0, \hat{y}_0)$ [S1].
4.  **Linear Combination**: Each pixel in the high-resolution block is a linear combination of the low-resolution intensities and their spatial gradients, weighted by a coefficient matrix $M$:
    $$
    \mathrm {Vec}(\hat {I}_{S\times S}(\hat {x}_0, \hat {y}_0)) = \mathbf {M} \begin {bmatrix} \mathrm {Vec}(I_{3\times 3}(x_0, y_0)) \\ \mathrm {Vec}(\frac {\partial }{\partial x}I_{3\times 3}(x_0, y_0))\\ \mathrm {Vec}(\frac {\partial }{\partial y}I_{3\times 3}(x_0, y_0)) \end {bmatrix}
    $$
    where $\mathrm {Vec}(\cdot)$ denotes column-wise vectorization [S1].
5.  **Implementation**: This operation is implemented by cascading a convolution layer with a transposed-convolution layer [S1]:
    $$
    F = \operatorname {Conv}(I)
    $$
    $$
    \hat {I} = \operatorname {TConv}(F)
    $$
    where $\operatorname {Conv}(\cdot)$ is a $3 \times 3$ convolution layer and $\operatorname {TConv}(\cdot)$ is an $S \times S$ transposed-convolution layer [S1]. $F$ acts as the coefficient vector in the polynomial space [S1].

### Geometry-Temporal Recurrent Refinement
To mitigate aliasing artifacts caused by gradient sampling frequency limits, a refinement network updates the feature map using geometric and temporal priors [S1].

1.  **Refinement Network Input**: The network takes $I$, $G$, $D$, $N_v$, and the warped historical feature $\tilde{h}$ [S1].
2.  **Feature Update**: The historical feature is updated via a modified GRU architecture:
    $$
    h = \operatorname {RefineNet}(G, D, N_v, \tilde{h})
    $$
    [S1]
3.  **Residual Generation**: A residual feature $\Delta F$ is generated from the updated historical feature:
    $$
    \Delta F = \operatorname {Conv}(h)
    $$
    [S1]
4.  **Feature Map Update**: The feature map $F$ is updated as:
    $$
    F \leftarrow F + \Delta F
    $$
    [S1]
    This updated $F$ is then used in the transposed convolution step to produce the final high-resolution image [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GASSModule(nn.Module):
    def __init__(self, super_sample_factor=2):
        super(GASSModule, self).__init__()
        self.S = super_sample_factor
        # Convolution layers for interpolation
        # Input channels: 1 (for I) -> Output channels: C_F
        # C_F must be > min(S^2, 27)
        self.conv_interpolation = nn.Conv2d(1, 32, kernel_size=3, padding=1)
        # Transposed convolution for upsampling
        self.tconv_interpolation = nn.ConvTranspose2d(32, 1, kernel_size=S, stride=S)
        
        # Refinement Network components
        # Simplified GRU-like structure for feature update
        self.refine_conv = nn.Conv2d(32, 32, kernel_size=3, padding=1)
        # Inputs to RefineNet: G (2 channels), D (1 channel), Nv (1 channel), h_tilde (32 channels)
        self.refine_net_input_conv = nn.Conv2d(2 + 1 + 1 + 32, 32, kernel_size=3, padding=1)
        
        # Parameters
        self.lambda1 = 0.1

    def compute_gradients(self, I):
        """
        Computes image-space gradients G from low-res image I.
        Assumes I is differentiable with respect to x and y.
        In practice, this might use finite differences or the analytical 3DGS gradient formula.
        Here we use Sobel filters as a proxy for the analytical gradient extraction described.
        """
        # I shape: (B, 1, H, W)
        # Gradient kernels
        sobel_x = torch.tensor([[-1, 0, 1], [-2, 0, 2], [-1, 0, 1]], dtype=I.dtype, device=I.device).view(1, 1, 3, 3)
        sobel_y = torch.tensor([[-1, -2, -1], [0, 0, 0], [1, 2, 1]], dtype=I.dtype, device=I.device).view(1, 1, 3, 3)
        
        # Pad I for convolution
        I_padded = F.pad(I, (1, 1, 1, 1), mode='reflect')
        
        # Compute gradients
        grad_x = F.conv2d(I_padded, sobel_x, padding=0)
        grad_y = F.conv2d(I_padded, sobel_y, padding=0)
        
        return torch.cat([grad_x, grad_y], dim=1) # Shape: (B, 2, H, W)

    def forward(self, I, D, Nv, h_tilde):
        """
        I: Low-res image (B, 1, H, W)
        D: Depth map (B, 1, H, W)
        Nv: Normal-view inner product map (B, 1, H, W)
        h_tilde: Warped historical feature (B, 32, H, W)
        """
        # 1. Compute Gradients
        G = self.compute_gradients(I)
        
        # 2. Gradient-Aware Interpolation
        # F = Conv(I)
        F = self.conv_interpolation(I)
        
        # 3. Geometry-Temporal Recurrent Refinement
        # Concatenate inputs for RefineNet: G, D, Nv, h_tilde
        # G: (B, 2, H, W), D: (B, 1, H, W), Nv: (B, 1, H, W), h_tilde: (B, 32, H, W)
        refine_input = torch.cat([G, D, Nv, h_tilde], dim=1)
        
        # h = RefineNet(G, D, Nv, h_tilde)
        # Simplified: Convolution on concatenated inputs to produce updated feature h
        h = self.refine_net_input_conv(refine_input)
        
        # Delta F = Conv(h)
        delta_F = self.refine_conv(h)
        
        # F <- F + Delta F
        F = F + delta_F
        
        # 4. Upsample
        # I_hat = TConv(F)
        I_hat = self.tconv_interpolation(F)
        
        return I_hat

    def compute_loss(self, I_hat, I_gt):
        """
        Computes GASS loss: L = L_C + lambda1 * L_lap
        L_C: Charbonnier loss
        L_lap: Laplacian loss ||grad^2 X - grad^2 Y||_1
        """
        # Charbonnier loss
        epsilon = 1e-3
        L_C = torch.mean(torch.sqrt((I_hat - I_gt)**2 + epsilon**2))
        
        # Laplacian loss
        # Approximate Laplacian using 2nd order derivatives
        # Simple 2D Laplacian kernel
        laplacian_kernel = torch.tensor([[0, 1, 0], [1, -4, 1], [0, 1, 0]], dtype=I_hat.dtype, device=I_hat.device).view(1, 1, 3, 3)
        
        I_hat_lap = F.conv2d(F.pad(I_hat, (1, 1, 1, 1), mode='reflect'), laplacian_kernel, padding=0)
        I_gt_lap = F.conv2d(F.pad(I_gt, (1, 1, 1, 1), mode='reflect'), laplacian_kernel, padding=0)
        
        L_lap = torch.mean(torch.abs(I_hat_lap - I_gt_lap))
        
        L = L_C + self.lambda1 * L_lap
        return L
```

### Parameters
*   **Super-sampling factor $S$**: Determines the output resolution relative to input. Typical values depend on the desired output resolution (e.g., $S=2$ for 2x upscaling) [S1].
*   **Number of channels in $F$ ($C_F$)**: Must exceed $\min\{S^2, 27\}$ [S1].
*   **$\lambda_1$**: Weighting factor for Laplacian loss. Value is $0.1$ [S1].

### Edge Cases and Pitfalls
*   **Gradient Sampling Frequency**: Gradients are derived from partial derivatives of the original continuous image function, so they remain constrained by the sampling frequency, making it difficult to avoid aliasing artifacts without refinement [S1].
*   **Rank of Matrix M**: The number of channels in $F$ must be sufficient to ensure the coefficient matrix $M$ has full column or row rank; otherwise, the interpolation may not capture higher-order relationships [S1].
*   **Historical Feature Alignment**: The warped historical feature $\tilde{h}$ must be accurately aligned to the current frame; misalignment can introduce temporal inconsistencies [S1].

## Operating characteristics
*   **Time Complexity**: GASS is designed to be lightweight and computationally efficient, reducing average rendering time per frame by enabling lower-resolution rendering followed by super-sampling [S1]. The use of convolutional layers and a compact GRU-like backbone contributes to this efficiency [S1].
*   **Memory Complexity**: Requires storage for low-resolution images, gradients, depth maps, normal maps, and historical features. The feature map $F$ size depends on the chosen number of channels [S1].
*   **Strengths**: Effectively leverages geometric and historical priors for superior super-sampling performance compared to gradient-free methods; reduces aliasing artifacts; improves rendering efficiency [S1].
*   **Limitations**: Relies on the availability of accurate depth and normal maps; performance may degrade if historical features are poorly aligned; gradients are still limited by the initial sampling frequency [S1].
*   **When to Use**: When real-time rendering of large-scale 3DGS scenes is required, and high-resolution output is needed without the computational cost of rendering at full resolution directly [S1].
*   **When Not to Use**: When geometric information (depth, normals) is unavailable or unreliable; when temporal coherence is not a factor (e.g., single static image) [S1].

## Background
GASS was introduced in the paper "3DGS$^3$: Joint Super Sampling and Frame Interpolation for Real-Time Large-Scale 3DGS Rendering" by Y. Zhao et al. [S1]. It is part of a unified framework that jointly performs super-sampling and frame interpolation to accelerate 3DGS rendering [S1]. The method builds on the differentiable nature of 3DGS-rendered images and incorporates insights from Hermite interpolation [S1]. It is designed to work collaboratively with a Lightweight Temporal Frame Interpolation (LTFI) module to reduce computational cost [S1].

## Sources
[S1] 3DGS$$^3$$: Joint Super Sampling and Frame Interpolation for Real-Time Large-Scale 3DGS Rendering, https://doi.org/10.1007/978-3-032-37314-4_30, fetched 2026-10-29
