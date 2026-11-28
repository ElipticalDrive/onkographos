# Explicit Neural View Synthesis (Voxel-based Neural Rendering)

Also called: Fast and Explicit Neural View Synthesis, explicit voxel-like scene representation, our model

## Overview
Explicit Neural View Synthesis (ENR) is a novel view synthesis method that explicitly encodes sparse source observations into a volumetric representation (RGBα volume) to enable amortized rendering, avoiding per-pixel ray sampling and scene-specific optimization [S1]. Unlike continuous neural radiance fields such as NeRF [26], which require independent ray integrals for each pixel and prevent amortization across views, ENR pre-computes a shared RGBα volume that can be rendered for any target view via geometric transformations [S1]. This approach offers rendering speeds over 400x faster than continuous baselines while maintaining comparable or superior reconstruction quality [S1].

## How it works
The method consists of four main stages: 2D feature encoding, inverse projection (back-projection), 3D feature decoding, and amortized rendering [S1].

### Inputs and Outputs
*   **Input:** Source image(s) $I_s \in \mathbb{R}^{3 \times h \times w}$ and camera intrinsic parameters $K$ (assumed constant) [S1]. For multi-view, multiple source images are aggregated into a single latent volume [S1].
*   **Output:** Target image $\hat{I}_t \in \mathbb{R}^{3 \times h \times w}$ rendered from a target camera pose $P_t$ [S1].

### Data Structures and Layouts
1.  **2D Feature Map ($F_s$):**
    *   Layout: $\mathbb{R}^{32 \times h \times w}$.
    *   Spatial resolution matches the input image [S1].
    *   Channels: 32.
2.  **Latent Volumetric Tensor ($Z_s$):**
    *   Layout: $\mathbb{R}^{c \times d_s \times h_s \times w_s}$.
    *   $d_s, h_s, w_s$ are depth, height, and width of the volumetric representation [S1].
    *   Spatial height and width are halved with respect to the input feature map ($h_s = h/2, w_s = w/2$) [S1].
    *   Depth $d_s$ corresponds to the depth of the voxel grid [S1].
3.  **RGBα Volume ($V_s$):**
    *   Layout: $\mathbb{R}^{4 \times d_s \times h \times w}$.
    *   Channels: 4 (R, G, B, α).
    *   Spatial resolution matches the input image ($h, w$) [S1].
    *   Produced by a final 3D convolution and sigmoid activation [S1].

### Numbered Steps

1.  **2D Feature Encoding:**
    Process the source image $I_s$ through a 2D U-Net encoder with a ResNet-18 backbone [S1].
    *   **Downsampling:** Intermediate feature channels are 64, 128, 256, 512 at each spatial resolution level.
    *   **Upsampling:** Perform deconvolution and fuse resulting features with skip encoder features at the same level through concatenation followed by two consecutive convolution blocks [S1].
    *   **Output:** A 32-channel feature map $F_s \in \mathbb{R}^{32 \times h \times w}$ preserving the spatial resolution of $I_s$ [S1].

2.  **Inverse Projection (Back-projection):**
    Back-project $F_s$ into a latent volumetric tensor $Z_s \in \mathbb{R}^{c \times d_s \times h_s \times w_s}$ [S1].
    *   **Tiling:** Tile the 2D features into a 3D cube to prepare for sampling.
    *   **Homography:** Leverage camera intrinsic parameters $K$ to compute the homography matrix mapping the cube to the viewing frustum [S1].
    *   **Grid Mapping:** Obtain the grid mapping function by multiplying the homography matrix with individual points from a fixed voxel mesh grid set [S1].
    *   **Sampling:** Apply the `grid_sample` function to perform the inverse projection. This step is critical to preserve 3D geometry and texture information, which is lost if 2D feature maps are reshaped directly into 3D volumes [S1].

3.  **3D Feature Decoding:**
    Process the latent volume $Z_s$ with a 3D U-Net model to predict the final RGBα volume $V_s$ [S1].
    *   **Architecture:** 3D U-Net with ResNet-3D blocks.
    *   **Channel Scaling:** When spatial resolution is downscaled by 2 times, double the channel numbers [S1].
    *   **Output Feature Map:** 16 channels with the same spatial resolution as the input image [S1].
    *   **Final Layer:** Apply one 3D convolution to transform the tensor map into 4 channels [S1].
    *   **Activation:** Apply sigmoid activation to produce the RGBα volume $V_s \in \mathbb{R}^{4 \times d_s \times h \times w}$ [S1].

4.  **Amortized Rendering:**
    Render the transformed RGBα volume $V_t$ into the target image $\hat{I}_t$ [S1].
    *   **Pose Transformation:** Compute the relative rigid transformation $P_{st} = P_t P_s^{-1}$ between source pose $P_s$ and target pose $P_t$ [S1]. Apply this transformation to $V_s$ to obtain $V_t = T(P_{st}, V_s)$, aligning the volume with the target view [S1]. This combines rotation (transferring volume from source to target view) and perspective deformation (mapping viewing frustum to output cube) via left matrix multiplication [S1].
    *   **Perspective Deformation:** Apply inverse warping and trilinear sampling to the transformed volume. Similar to inverse projection, use mesh grid multiplication and `grid_sample` to obtain the voxel volume aligned with the target camera [S1].
    *   **Alpha Blending:** Calculate the expected color $\hat{C}_{i,j}$ for each pixel $(i,j)$ using accumulated transmittance:
        $$
        \hat{C}_{i,j} = \sum_{k=1}^{d_s} T_{i,j}^k \alpha_{i,j}^k \mathbf{c}_{i,j}^k
        $$
        where:
        *   $\mathbf{c}_{i,j}^k$ is the color value (first 3 channels) of the voxel at depth $k$ [S1].
        *   $\alpha_{i,j}^k$ is the alpha value (4th channel) of the voxel at depth $k$ [S1].
        *   $T_{i,j}^k$ is the accumulated transmittance, calculated using the `cumprod` method:
            $$
            T_{i,j}^k = \prod_{m=1}^{k-1} (1-\alpha_{i,j}^m)
            $$
        *   The product is over the depth dimension $k=1 \dots d_s$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ExplicitNeuralViewSynthesis(nn.Module):
    def __init__(self, intrinsic_params, voxel_depth=16, voxel_height=64, voxel_width=64):
        super().__init__()
        self.K = intrinsic_params
        self.d_s = voxel_depth
        self.h_s = voxel_height
        self.w_s = voxel_width
        self.encoder_2d = UNet2D(channels_in=3, channels_out=32)
        self.decoder_3d = UNet3D(channels_in=32, channels_out=4)
        # Fixed voxel mesh grid for inverse projection and rendering
        # Shape: (1, d_s, h_s, w_s, 3) normalized to [-1, 1]
        self.voxel_grid = self._create_voxel_grid()

    def _create_voxel_grid(self):
        z = torch.linspace(-1, 1, self.d_s)
        y = torch.linspace(-1, 1, self.h_s)
        x = torch.linspace(-1, 1, self.w_s)
        z, y, x = torch.meshgrid(z, y, x, indexing='ij')
        return torch.stack([x, y, z], dim=-1).unsqueeze(0)

    def inverse_projection(self, F_s):
        B, C, h, w = F_s.shape
        h_s, w_s = self.h_s, self.w_s
        # Tile 2D features into a 3D cube (B, C, d_s, h, w)
        F_tiled = F_s.unsqueeze(2).repeat(1, 1, self.d_s, 1, 1)
        # [G] Implementation detail: Compute homography matrix from K and map voxel centers to image pixels.
        # Returns grid of shape (B, d_s, h_s, w_s, 3) in normalized [-1, 1] coordinates.
        sampling_grid = self._get_inverse_projection_grid(F_s)
        # Perform inverse projection using grid_sample on 4D tensor
        Z_s = F.grid_sample(F_tiled, sampling_grid, align_corners=True)
        return Z_s

    def _get_inverse_projection_grid(self, F_s):
        # Not given in the sources. Computes homography and maps voxel centers to image pixels.
        pass

    def forward(self, I_s, P_t, P_s):
        F_s = self.encoder_2d(I_s)
        Z_s = self.inverse_projection(F_s)
        V_s = self.decoder_3d(Z_s)
        return self.render(V_s, P_t, P_s)

    def render(self, V_s, P_t, P_s):
        B, C, d_s, h, w = V_s.shape
        P_st = P_t @ torch.linalg.inv(P_s)
        # Apply rigid transformation to volume
        # Not given in the sources. Uses grid_sample with a grid generated from P_st to rotate/translate the voxel grid.
        V_t = self.transform_volume(V_s, P_st)
        C_t = V_t[:, :3, :, :, :]
        Alpha_t = V_t[:, 3:4, :, :, :]
        one_minus_alpha = 1.0 - Alpha_t
        cumprod = torch.cumprod(one_minus_alpha, dim=2)
        ones = torch.ones_like(cumprod[:, :, 0:1, :, :])
        T = torch.cat([ones, cumprod[:, :, :-1, :, :]], dim=2)
        weighted_color = T * Alpha_t * C_t
        return torch.sum(weighted_color, dim=2)

    def transform_volume(self, V_s, P_st):
        # Not given in the sources. Uses grid_sample with a grid generated from P_st to rotate/translate the voxel grid.
        pass

class UNet2D(nn.Module):
    def __init__(self, channels_in, channels_out):
        super().__init__()
        # Not given in the sources. 2D U-Net with ResNet-18 backbone.
        pass

class UNet3D(nn.Module):
    def __init__(self, channels_in, channels_out):
        super().__init__()
        # Not given in the sources. 3D U-Net with ResNet-3D blocks.
        pass
```

### Parameters
*   **Learning Rate:** 0.0016 (fixed throughout training) [S1].
*   **SSIM Loss Weight:** 0.05 [S1].
*   **Training Epochs:** 150 [S1].
*   **Nonlinearity:** ReLU used throughout the model for each nonlinear transformation [S1].
*   **Normalization:** GroupNorm [54] used throughout the model for each nonlinear transformation [S1].
*   **Data Normalization:** ImageNet statistics applied only at the preprocessing stage [S1].
*   **Intrinsic Camera Parameters:** Used for inverse projection and perspective deformation; assumed constant [S1].

### Edge Cases and Pitfalls
*   **Geometry Preservation:** Reshaping 2D feature maps directly into 3D volumes loses 3D geometry and texture information. The inverse projection step is essential to preserve this [S1].
*   **Resolution Limits:** The resolution of rendered images is capped by the explicit voxel-like scene representation. Continuous representations are resolution-free but slower [S1]. Future work plans to use octrees, mixture of volumetric primitives, or scene graphs to increase capacity for complex regions while minimizing resources for empty regions, and to enhance with super-resolution methods [S1].
*   **View-Dependent Lighting:** The current model cannot produce view-dependent lighting effects. It assumes a Lambertian RGBA model. Future work plans to incorporate physically based rendering with view direction, light sources, material properties (roughness, metalness), and surface information (normals, displacement maps), potentially using spherical harmonics or learned MLPs for specular effects [S1].

## Operating characteristics
*   **Rendering Speed:** Over 400x faster than continuous baselines like NeRF [S1].
*   **Complexity:** Rendering is amortized across views because the RGBα volume is shared. The cost is dominated by the initial encoding and decoding, which are fixed per scene [S1].
*   **Memory:** Memory usage is determined by the size of the 3D voxel grid ($d_s \times h \times w$). The source does not specify exact memory consumption in MB, but notes that resolution is capped by the explicit representation [S1].
*   **Performance:** On unseen-category view synthesis (Table 8 in [S1]), the two-view model achieves mean PSNR 24.80, SSIM 0.893, LPIPS 0.107, outperforming the ENR baseline (PSNR 23.16, SSIM 0.807, LPIPS 0.190) [S1]. The single-view model achieves the same mean metrics (PSNR 24.80, SSIM 0.893, LPIPS 0.107) [S1].
*   **Strengths:** Fast rendering, no per-pixel ray sampling, no scene-specific optimization, category-agnostic training [S1].
*   **Limitations:** Resolution capped by voxel grid, no view-dependent lighting effects [S1].

## Background
Explicit Neural View Synthesis was proposed by Pengsheng Guo et al. from Apple [S1]. It challenges the trend of continuous and implicit radiance field representations by proposing a simple, explicit volumetric approach that achieves comparable or better quality with significantly faster rendering speeds [S1].

## Sources
[S1] Fast and Explicit Neural View Synthesis, https://doi.org/10.1109/wacv51458.2022.00009, fetched 2026-10-30
