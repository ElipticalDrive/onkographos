# Explicit Neural View Synthesis (Voxel-based Neural Rendering)

Also called: Fast and Explicit Neural View Synthesis, explicit voxel-like scene representation, our model

## Overview
Explicit Neural View Synthesis (ENR) is a novel view synthesis method that explicitly encodes sparse source observations into a volumetric representation (RGB$\alpha$ volume) to enable amortized rendering, avoiding per-pixel ray sampling and scene-specific optimization [S1]. Unlike continuous neural radiance fields such as NeRF [26], which require independent ray integrals for each pixel and prevent amortization across views, ENR pre-computes a shared RGB$\alpha$ volume that can be rendered for any target view via geometric transformations [S1]. This approach offers rendering speeds over 400x faster than continuous baselines while maintaining comparable or superior reconstruction quality [S1].

## How it works
The method consists of four main stages: 2D feature encoding, inverse projection (back-projection), 3D feature decoding, and amortized rendering.

### Inputs and Outputs
*   **Input:** Source image(s) $I_s \in \mathbb{R}^{3 \times h \times w}$ and camera intrinsic parameters $K$ (assumed constant). For multi-view, multiple source images are aggregated into a single latent volume.
*   **Output:** Target image $\hat{I}_t \in \mathbb{R}^{3 \times h \times w}$ rendered from a target camera pose $P_t$.

### Data Structures and Layouts
1.  **2D Feature Map ($F_s$):**
    *   Layout: $\mathbb{R}^{32 \times h \times w}$.
    *   Spatial resolution matches the input image.
    *   Channels: 32.
2.  **Latent Volumetric Tensor ($Z_s$):**
    *   Layout: $\mathbb{R}^{c \times d_s \times h_s \times w_s}$.
    *   $d_s, h_s, w_s$ are depth, height, and width of the volumetric representation.
    *   Spatial height and width are halved with respect to the input feature map ($h_s = h/2, w_s = w/2$) [S1].
    *   Depth $d_s$ corresponds to the depth of the voxel grid.
3.  **RGB$\alpha$ Volume ($V_s$):**
    *   Layout: $\mathbb{R}^{4 \times d_s \times h \times w}$.
    *   Channels: 4 (R, G, B, $\alpha$).
    *   Spatial resolution matches the input image ($h, w$) [S1].
    *   Produced by a final 3D convolution and sigmoid activation.

### Numbered Steps

1.  **2D Feature Encoding:**
    Process the source image $I_s$ through a 2D U-Net encoder with a ResNet-18 backbone [S1].
    *   **Downsampling:** Intermediate feature channels are 64, 128, 256, 512 at each spatial resolution level.
    *   **Upsampling:** Perform deconvolution and fuse resulting features with skip encoder features at the same level through concatenation followed by two consecutive convolution blocks.
    *   **Output:** A 32-channel feature map $F_s \in \mathbb{R}^{32 \times h \times w}$ preserving the spatial resolution of $I_s$ [S1].

2.  **Inverse Projection (Back-projection):**
    Back-project $F_s$ into a latent volumetric tensor $Z_s \in \mathbb{R}^{c \times d_s \times h_s \times w_s}$ [S1].
    *   **Tiling:** Tile the 2D features into a 3D cube to prepare for sampling.
    *   **Homography:** Leverage camera intrinsic parameters $K$ to compute the homography matrix mapping the cube to the viewing frustum.
    *   **Grid Mapping:** Obtain the grid mapping function by multiplying the homography matrix with individual points from a fixed voxel mesh grid set.
    *   **Sampling:** Apply the `grid_sample` function to perform the inverse projection. This step is critical to preserve 3D geometry and texture information, which is lost if 2D feature maps are reshaped directly into 3D volumes [S1].

3.  **3D Feature Decoding:**
    Process the latent volume $Z_s$ with a 3D U-Net model to predict the final RGB$\alpha$ volume $V_s$ [S1].
    *   **Architecture:** 3D U-Net with ResNet-3D blocks.
    *   **Channel Scaling:** When spatial resolution is downscaled by 2 times, double the channel numbers.
    *   **Output Feature Map:** 16 channels with the same spatial resolution as the input image.
    *   **Final Layer:** Apply one 3D convolution to transform the tensor map into 4 channels.
    *   **Activation:** Apply sigmoid activation to produce the RGB$\alpha$ volume $V_s \in \mathbb{R}^{4 \times d_s \times h \times w}$ [S1].

4.  **Amortized Rendering:**
    Render the transformed RGB$\alpha$ volume $V_t$ into the target image $\hat{I}_t$ [S1].
    *   **Pose Transformation:** Compute the relative rigid transformation $P_{st} = P_t P_s^{-1}$ between source pose $P_s$ and target pose $P_t$. Apply this transformation to $V_s$ to obtain $V_t = T(P_{st}, V_s)$, aligning the volume with the target view. This combines rotation (transferring volume from source to target view) and perspective deformation (mapping viewing frustum to output cube) via left matrix multiplication [S1].
    *   **Perspective Deformation:** Apply inverse warping and trilinear sampling to the transformed volume. Similar to inverse projection, use mesh grid multiplication and `grid_sample` to obtain the voxel volume aligned with the target camera [S1].
    *   **Alpha Blending:** Calculate the expected color $\hat{C}_{i,j}$ for each pixel $(i,j)$ using accumulated transmittance:
        $$
        \hat{C}_{i,j} = \sum_{k=1}^{d_s} T_{i,j}^k \alpha_{i,j}^k \mathbf{c}_{i,j}^k
        $$
        where:
        *   $\mathbf{c}_{i,j}^k$ is the color value (first 3 channels) of the voxel at depth $k$.
        *   $\alpha_{i,j}^k$ is the alpha value (4th channel) of the voxel at depth $k$.
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
        """
        Args:
            intrinsic_params: Camera intrinsic matrix K (3x3).
            voxel_depth: Depth of the voxel grid d_s.
            voxel_height: Height of the voxel grid h_s (typically h/2).
            voxel_width: Width of the voxel grid w_s (typically w/2).
        """
        super().__init__()
        self.K = intrinsic_params
        self.d_s = voxel_depth
        self.h_s = voxel_height
        self.w_s = voxel_width
        
        # 2D Encoder: U-Net with ResNet-18 backbone
        self.encoder_2d = UNet2D(channels_in=3, channels_out=32)
        
        # 3D Decoder: 3D U-Net with ResNet-3D blocks
        self.decoder_3d = UNet3D(channels_in=32, channels_out=4)
        
        # Fixed voxel mesh grid for inverse projection and rendering
        # Shape: (1, d_s, h_s, w_s, 3) normalized to [-1, 1]
        self.voxel_grid = self._create_voxel_grid()

    def _create_voxel_grid(self):
        # Create a 3D grid of coordinates for the voxel space
        # This represents the fixed voxel mesh grid set
        z = torch.linspace(-1, 1, self.d_s)
        y = torch.linspace(-1, 1, self.h_s)
        x = torch.linspace(-1, 1, self.w_s)
        z, y, x = torch.meshgrid(z, y, x, indexing='ij')
        grid = torch.stack([x, y, z], dim=-1).unsqueeze(0) # (1, d_s, h_s, w_s, 3)
        return grid

    def inverse_projection(self, F_s):
        """
        Back-project 2D feature map F_s into latent volumetric tensor Z_s.
        Args:
            F_s: (B, 32, h, w)
        Returns:
            Z_s: (B, 32, d_s, h_s, w_s)
        """
        B, C, h, w = F_s.shape
        h_s, w_s = self.h_s, self.w_s
        
        # Tile 2D features into a 3D cube (B, C, d_s, h, w)
        # Note: The source mentions tiling into a 3D cube to prepare for sampling.
        # We replicate the feature map along the depth dimension.
        F_tiled = F_s.unsqueeze(2).repeat(1, 1, self.d_s, 1, 1) # (B, C, d_s, h, w)
        
        # Compute homography matrix from intrinsic parameters
        # Assuming constant intrinsic parameters and mapping cube to frustum
        # In practice, this involves projecting voxel centers to image plane
        # and using the inverse mapping.
        # For implementation, we generate a sampling grid based on camera geometry.
        
        # Generate sampling grid for inverse projection
        # Map voxel grid points to image coordinates using K
        # This step is complex; simplified here as a grid_sample operation
        # using a pre-computed homography-based grid.
        
        # Create a grid that maps voxel coordinates to image coordinates
        # This requires projecting the voxel grid points (x_v, y_v, z_v) to 2D
        # using the camera matrix K.
        # Let's assume a helper function `compute_homography_grid` exists
        # which returns a grid of shape (B, d_s, h_s, w_s, 2) in normalized [-1, 1]
        # coordinates for the source image plane.
        
        # For the sake of self-contained pseudocode, we simulate the grid generation
        # based on the description: "multiplying the homography matrix with 
        # individual points from the fixed voxel mesh grid set".
        
        # Reshape voxel grid to points
        points = self.voxel_grid.reshape(1, -1, 3) # (1, d_s*h_s*w_s, 3)
        
        # Project points to 2D image plane (homogeneous)
        # K is 3x3. Points are (x, y, z). 
        # Standard projection: x_img = K * [x/z, y/z, 1]^T ? 
        # Actually, for inverse projection, we map voxel centers to image pixels.
        # Assuming the voxel space is normalized and K is known.
        
        # Simplified: Use a pre-computed grid for grid_sample
        # In a real implementation, this grid is computed once based on K and voxel dimensions.
        # Here we assume `sampling_grid_2d` is computed externally or via a helper.
        # To keep it self-contained, we note that this step uses `grid_sample`.
        
        # Placeholder for the actual grid computation logic which depends on K
        # and the specific voxel-to-image projection model.
        # We assume `sampling_grid` is of shape (B, d_s, h_s, w_s, 2)
        sampling_grid = self._get_inverse_projection_grid(F_s)
        
        # Perform inverse projection using grid_sample
        # F_tiled is (B, C, d_s, h, w). We want to sample from F_s (B, C, h, w)
        # using the grid that maps (d_s, h_s, w_s) indices to (h, w) coordinates.
        # This is non-standard grid_sample usage. Usually grid_sample samples from 
        # a 2D image using a 2D grid. Here we are back-projecting.
        # The source says: "apply the grid_sample function to perform the inverse projection".
        # This implies we are sampling the 2D feature map F_s at locations determined 
        # by the projection of the voxel grid.
        
        # Reshape F_tiled to be sampled? No, F_s is the source.
        # We need to sample F_s at coordinates given by the projection of the voxel grid.
        # The grid should map (d_s, h_s, w_s) -> (h, w).
        # But F_s is (B, C, h, w).
        # The source says "Tile the 2D features into a 3D cube... apply grid_sample".
        # This likely means:
        # 1. Create a grid of shape (B, d_s, h_s, w_s, 2) representing the 2D coordinates
        #    in the source image corresponding to each voxel.
        # 2. Use grid_sample to sample F_s at these coordinates.
        # 3. The result is (B, C, d_s, h_s, w_s).
        
        # Let's assume `sampling_grid` is correctly computed.
        Z_s = F.grid_sample(F_s, sampling_grid, align_corners=True)
        # Note: grid_sample expects input (B, C, H, W) and grid (B, ..., 2)
        # Output is (B, C, ..., 2) -> (B, C, d_s, h_s, w_s)
        
        return Z_s

    def _get_inverse_projection_grid(self, F_s):
        # This is a placeholder for the complex geometry calculation.
        # In reality, this computes the homography and maps voxel centers to image pixels.
        # Returns grid of shape (B, d_s, h_s, w_s, 2) in normalized [-1, 1] coordinates.
        # For the purpose of this article, we assume this helper is available.
        # [G] Implementation detail: Compute projection matrix P = K [R|t]. 
        # Project voxel grid points to 2D, normalize to [-1, 1].
        pass

    def forward(self, I_s, P_t, P_s):
        """
        Args:
            I_s: Source image (B, 3, h, w)
            P_t: Target camera pose (4x4 matrix)
            P_s: Source camera pose (4x4 matrix)
        Returns:
            I_t: Rendered target image (B, 3, h, w)
        """
        # 1. Encode
        F_s = self.encoder_2d(I_s) # (B, 32, h, w)
        
        # 2. Inverse Projection
        Z_s = self.inverse_projection(F_s) # (B, 32, d_s, h_s, w_s)
        
        # 3. Decode
        V_s = self.decoder_3d(Z_s) # (B, 4, d_s, h, w)
        
        # 4. Amortized Rendering
        I_t = self.render(V_s, P_t, P_s)
        
        return I_t

    def render(self, V_s, P_t, P_s):
        """
        Render RGBalpha volume V_s to target image I_t.
        Args:
            V_s: (B, 4, d_s, h, w)
            P_t: Target pose
            P_s: Source pose
        Returns:
            I_t: (B, 3, h, w)
        """
        B, C, d_s, h, w = V_s.shape
        
        # Compute relative transformation P_st = P_t * P_s^{-1}
        P_st = P_t @ torch.linalg.inv(P_s)
        
        # Apply rigid transformation to volume
        # This involves rotating the voxel grid and resampling
        # The source says: "combine rotation transformation and perspective deformation 
        # into one by left matrix multiplication... perspective deformation maps 
        # the viewing frustum to the output cube."
        
        # Generate sampling grid for rendering
        # Map target image pixels to voxel coordinates
        # This is the inverse of the projection from voxel to image.
        # We need to sample V_s at locations determined by the target camera.
        
        # Create a grid for the target image (h, w) that maps to voxel space (d_s, h, w)
        # This grid should be computed based on P_st and intrinsic parameters.
        # Similar to inverse projection, but for the target view.
        
        # [G] Implementation detail: Compute grid that maps target image coordinates 
        # to voxel coordinates in the transformed volume.
        sampling_grid_render = self._get_rendering_grid(V_s, P_st)
        
        # Sample the volume V_s using the grid
        # V_s is (B, 4, d_s, h, w). We want to sample along the depth dimension 
        # for each pixel (i, j).
        # This is effectively a ray cast through the volume.
        # However, the source describes "amortized rendering" using "inverse warping 
        # and trilinear sampling".
        
        # The source formula:
        # C_hat_{i,j} = sum_{k=1}^{d_s} T_{i,j}^k alpha_{i,j}^k c_{i,j}^k
        # T_{i,j}^k = prod_{m=1}^{k-1} (1 - alpha_{i,j}^m)
        
        # This implies we need to access the volume along the depth axis for each pixel.
        # The "grid sampling" step likely aligns the volume with the target view 
        # so that the depth axis corresponds to the ray direction.
        
        # Let's assume `V_t` is the volume transformed and aligned such that 
        # the depth dimension is the ray direction for the target camera.
        # The source says: "After similar mesh grid multiplication and grid sampling 
        # steps as in inverse projection, we obtain the voxel volume used for alpha blending."
        
        # So, we first transform V_s to V_t.
        V_t = self.transform_volume(V_s, P_st)
        
        # Now, perform alpha blending along the depth dimension of V_t.
        # V_t is (B, 4, d_s, h, w).
        # For each pixel (i, j), we have a vector of length d_s for color and alpha.
        
        # Extract color and alpha
        # Color: (B, 3, d_s, h, w)
        # Alpha: (B, 1, d_s, h, w)
        C_t = V_t[:, :3, :, :, :]
        Alpha_t = V_t[:, 3:4, :, :, :]
        
        # Calculate transmittance T
        # T_{i,j}^k = prod_{m=1}^{k-1} (1 - alpha_{i,j}^m)
        # This is a cumulative product over the depth dimension.
        # (1 - alpha) is (B, 1, d_s, h, w)
        one_minus_alpha = 1.0 - Alpha_t
        # Cumprod along dim=2 (depth)
        # T_k = prod_{m=1}^{k-1} ...
        # So T_1 = 1, T_2 = (1-alpha_1), T_3 = (1-alpha_1)(1-alpha_2)
        # We can use cumprod and shift.
        cumprod = torch.cumprod(one_minus_alpha, dim=2) # (B, 1, d_s, h, w)
        # Shift to get T^k
        # T^1 = 1
        # T^2 = cumprod[:, :, 0, :, :]
        # T^k = cumprod[:, :, k-2, :, :] for k > 1
        # Pad with 1 at the beginning
        ones = torch.ones_like(cumprod[:, :, 0:1, :, :])
        T = torch.cat([ones, cumprod[:, :, :-1, :, :]], dim=2) # (B, 1, d_s, h, w)
        
        # Compute color
        # C_hat = sum_{k} T^k * alpha^k * c^k
        # Element-wise multiplication
        weighted_color = T * Alpha_t * C_t # (B, 3, d_s, h, w)
        # Sum over depth
        I_t = torch.sum(weighted_color, dim=2) # (B, 3, h, w)
        
        return I_t

    def transform_volume(self, V_s, P_st):
        """
        Apply rigid transformation P_st to volume V_s.
        Args:
            V_s: (B, 4, d_s, h, w)
            P_st: (4, 4) relative pose
        Returns:
            V_t: (B, 4, d_s, h, w) transformed volume
        """
        # [G] Implementation detail: Use grid_sample with a grid generated 
        # from P_st to rotate and translate the voxel grid.
        # The grid maps the target voxel coordinates to source voxel coordinates.
        # This is similar to the inverse projection grid but for the volume itself.
        pass

    def _get_rendering_grid(self, V_s, P_st):
        # Placeholder for rendering grid computation
        pass

class UNet2D(nn.Module):
    def __init__(self, channels_in, channels_out):
        super().__init__()
        # Simplified U-Net with ResNet-18 backbone
        # Channels: 64, 128, 256, 512 during downsampling
        # Output: 32 channels
        pass

class UNet3D(nn.Module):
    def __init__(self, channels_in, channels_out):
        super().__init__()
        # Simplified 3D U-Net with ResNet-3D blocks
        # Output: 4 channels (RGBa)
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
*   **Resolution Limits:** The resolution of rendered images is capped by the explicit voxel-like scene representation. Continuous representations are resolution-free but slower. Future work plans to use octrees, mixture of volumetric primitives, or scene graphs to increase capacity for complex regions while minimizing resources for empty regions, and to enhance with super-resolution methods [S1].
*   **View-Dependent Lighting:** The current model cannot produce view-dependent lighting effects. It assumes a Lambertian RGBA model. Future work plans to incorporate physically based rendering with view direction, light sources, material properties (roughness, metalness), and surface information (normals, displacement maps), potentially using spherical harmonics or learned MLPs for specular effects [S1].

## Operating characteristics
*   **Rendering Speed:** Over 400x faster than continuous baselines like NeRF [S1].
*   **Complexity:** Rendering is amortized across views because the RGB$\alpha$ volume is shared. The cost is dominated by the initial encoding and decoding, which are fixed per scene.
*   **Memory:** Memory usage is determined by the size of the 3D voxel grid ($d_s \times h \times w$). The source does not specify exact memory consumption in MB, but notes that resolution is capped by the explicit representation [S1].
*   **Performance:** On unseen-category view synthesis (Table 8 in [S1]), the two-view model achieves mean PSNR 24.80, SSIM 0.893, LPIPS 0.107, outperforming the ENR baseline (PSNR 23.16, SSIM 0.807, LPIPS 0.190) [S1]. The single-view model achieves the same mean metrics (PSNR 24.80, SSIM 0.893, LPIPS 0.107) [S1].
*   **Strengths:** Fast rendering, no per-pixel ray sampling, no scene-specific optimization, category-agnostic training [S1].
*   **Limitations:** Resolution capped by voxel grid, no view-dependent lighting effects [S1].

## Background
Explicit Neural View Synthesis was proposed by Pengsheng Guo et al. from Apple [S1]. The paper was presented at the IEEE Winter Conference on Applications of Computer Vision (WACV) in 2022 [S1]. It challenges the trend of continuous and implicit radiance field representations by proposing a simple, explicit volumetric approach that achieves comparable or better quality with significantly faster rendering speeds [S1].

## Sources
[S1] Fast and Explicit Neural View Synthesis, https://doi.org/10.1109/wacv51458.2022.00009, fetched 2026-10-30
