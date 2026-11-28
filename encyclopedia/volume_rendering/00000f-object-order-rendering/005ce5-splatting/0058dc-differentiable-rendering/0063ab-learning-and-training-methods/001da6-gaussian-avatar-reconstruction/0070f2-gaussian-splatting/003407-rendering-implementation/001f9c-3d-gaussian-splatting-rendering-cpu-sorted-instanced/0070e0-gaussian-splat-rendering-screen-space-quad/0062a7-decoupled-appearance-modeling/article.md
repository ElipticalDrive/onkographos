# Decoupled Appearance Modeling

Also called: Appearance Adjustment CNN, Decoupled AM, Transformation Map Generator

## Overview
Decoupled Appearance Modeling is a post-processing module that handles appearance variations, such as uneven illumination, by generating a per-pixel transformation map to adjust rendered images without modifying the underlying 3D Gaussian representation [S1]. It prevents the 3D Gaussian Splatting (3DGS) optimizer from creating "floaters" (spurious geometry) to compensate for photometric changes between training views [S1]. Unlike sibling methods that rely on complex neural radiance fields or MLPs for appearance, this method uses a lightweight CNN to learn a residual appearance adjustment, allowing the 3D Gaussians to converge to a consistent average appearance and correct geometry [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A rendered image $I_{ir}$ of resolution $H \times W \times 3$ from the parent [Gaussian Splat Rendering (Screen-Space Quad)](2075-gaussian-splat-rendering-screen-space-quad.md) method, and an optimizable appearance embedding vector $\ell_i$ of length $m$ [S1].
*   **Output:** An appearance-variant image $I_{ia}$ of resolution $H \times W \times 3$, and a transformation map $M_i$ of resolution $H \times W \times C$ (where $C$ is the number of channels in the transformation, typically 3) [S1].

**Data Structures:**
*   **Appearance Embedding ($\ell_i$):** A learnable vector of length $m$. Typical value: $m=64$ [S1].
*   **Feature Map ($D_i$):** A 2D tensor of shape $\frac{H}{32} \times \frac{W}{32} \times (3 + m)$ [S1].
*   **Transformation Map ($M_i$):** A 2D tensor of shape $H \times W \times C$. For simple pixel-wise multiplication, $C=3$ [S1].

**Numbered Steps:**
1.  **Downsample:** Downsample the rendered image $I_{ir}$ by a factor of 32 to obtain an image of resolution $\frac{H}{32} \times \frac{W}{32} \times 3$ [S1]. This prevents the network from learning high-frequency details and reduces computation/memory [S1].
2.  **Concatenate Embedding:** Concatenate the appearance embedding $\ell_i$ (length $m$) to every pixel of the downsampled image. This creates the input feature map $D_i$ with shape $\frac{H}{32} \times \frac{W}{32} \times (3 + m)$ [S1].
3.  **Initial Convolution:** Pass $D_i$ through a $3 \times 3$ convolutional layer to increase the channel depth to 256. The output shape is $\frac{H}{32} \times \frac{W}{32} \times 256$ [S1].
4.  **Upsampling Blocks:** Process the feature map through 4 upsampling blocks. Each block consists of:
    *   A $4 \times$ pixel shuffle layer (increasing resolution by $2 \times$ in each dimension, halving channels).
    *   A $3 \times 3$ convolutional layer.
    *   A ReLU activation [S1].
    *   After 4 blocks, the resolution becomes $H \times W$ and the channel depth becomes 16 [S1].
5.  **Bilinear Interpolation:** Upsample the feature map to $H \times W$ via bilinear interpolation if not already at full resolution (the pixel shuffle blocks double resolution 4 times, $32 \to 16 \to 8 \to 4 \to 2 \to 1$ relative to the downsampled input, effectively restoring full resolution $H \times W$) [S1]. *Note: The source states "After that, a bilinear interpolation layer upsamples the feature map to H x W", implying the pixel shuffle blocks may not fully restore resolution or an explicit interpolation step is added for smoothness. The source explicitly lists "Bilinear Interpolation" as a distinct step after the 4 blocks yielding $H \times W \times 16$* [S1].
6.  **Final Convolution:** Apply two $3 \times 3$ convolutional layers with a ReLU activation between them to produce the transformation map $M_i$ of shape $H \times W \times 3$ [S1].
7.  **Transformation:** Apply the pixel-wise transformation $T$ using $M_i$ to the original rendered image $I_{ir}$ to get $I_{ia}$ [S1].

**Formulas:**
The appearance-variant image is computed as:
$$ I_{ia} = T(I_{ir}; M_i) $$
where $T$ is a pixel-wise transformation [S1]. For the default configuration, $T$ is simple pixel-wise multiplication:
$$ I_{ia}(x,y) = I_{ir}(x,y) \odot M_i(x,y) $$
where $\odot$ denotes element-wise multiplication [S1].

The loss function combines a structural loss on the raw rendering and an appearance loss on the adjusted rendering:
$$ \mathcal{L} = (1 - \lambda)\mathcal{L}_1(I_{ia}, I_i) + \lambda \mathcal{L}_{D-SSIM}(I_{ir}, I_i) $$
where $I_i$ is the ground truth image, $\mathcal{L}_1$ is the L1 loss, and $\mathcal{L}_{D-SSIM}$ is the D-SSIM loss [S1]. The parameter $\lambda$ balances the two terms [S1].

**Parameters:**
*   **Downsampling factor:** 32 [S1].
*   **Appearance embedding length ($m$):** 64 [S1].
*   **Initial conv output channels:** 256 [S1].
*   **Number of upsampling blocks:** 4 [S1].
*   **Transformation channels:** 3 (default, for multiplication) [S1]. Alternative configurations use 6 channels (multiplication + addition) or 4 channels (multiplication + Gamma correction) [S1].
*   **Learning rate:** 0.001 for both appearance embeddings and CNN [S1].

**Edge Cases and Pitfalls:**
*   **Visualization:** Transformation maps are 3-channel tensors. For visualization purposes, they must be converted to grayscale, as noted in the supplementary material [S1].
*   **Complexity:** Extending to affine transformations (multiplication + addition) or Gamma correction yields only marginal improvements in quality metrics (SSIM, PSNR, LPIPS) on the tested datasets, suggesting simple multiplication is sufficient [S1].
*   **Training vs. Inference:** The CNN and embeddings are optimized during training but must be discarded after training to maintain real-time rendering speed [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DecoupledAppearanceModeling(nn.Module):
    def __init__(self, m=64):
        super().__init__()
        self.m = m
        # Step 3: Initial Conv
        self.conv_init = nn.Conv2d(3 + m, 256, kernel_size=3, padding=1)
        
        # Step 4: 4 Upsampling Blocks
        # Each block: PixelShuffle(2) -> Conv2d(256->128) -> ReLU
        # Block 1: 256 -> 128
        # Block 2: 128 -> 64
        # Block 3: 64 -> 32
        # Block 4: 32 -> 16
        self.upsample_blocks = nn.ModuleList([
            nn.Sequential(
                nn.PixelShuffle(2),
                nn.Conv2d(256, 128, kernel_size=3, padding=1),
                nn.ReLU()
            ),
            nn.Sequential(
                nn.PixelShuffle(2),
                nn.Conv2d(128, 64, kernel_size=3, padding=1),
                nn.ReLU()
            ),
            nn.Sequential(
                nn.PixelShuffle(2),
                nn.Conv2d(64, 32, kernel_size=3, padding=1),
                nn.ReLU()
            ),
            nn.Sequential(
                nn.PixelShuffle(2),
                nn.Conv2d(32, 16, kernel_size=3, padding=1),
                nn.ReLU()
            )
        ])
        
        # Step 5: Bilinear Interpolation is handled in forward pass
        # Step 6: Final Convs
        self.conv_final_1 = nn.Conv2d(16, 16, kernel_size=3, padding=1)
        self.conv_final_2 = nn.Conv2d(16, 3, kernel_size=3, padding=1) # Output 3 channels for multiplication
        
    def forward(self, rendered_image, appearance_embedding):
        """
        rendered_image: Tensor of shape (H, W, 3)
        appearance_embedding: Tensor of shape (m,)
        Returns:
            I_ia: Tensor of shape (H, W, 3)
            M_i: Tensor of shape (H, W, 3)
        """
        H, W, _ = rendered_image.shape
        
        # Step 1: Downsample by 32
        # Using average pooling for downsampling
        downsampled = F.avg_pool2d(rendered_image.permute(2, 0, 1).unsqueeze(0), kernel_size=32, stride=32)
        downsampled = downsampled.squeeze(0).permute(1, 2, 0) # Shape: H/32, W/32, 3
        
        # Step 2: Concatenate appearance embedding
        # Repeat embedding to match spatial dimensions
        emb_repeated = appearance_embedding.unsqueeze(0).unsqueeze(0).repeat(1, 1, downsampled.shape[0], downsampled.shape[1])
        # downsampled is (H/32, W/32, 3), emb is (1, m, H/32, W/32)
        # Need to permute downsampled to (N, C, H, W) for concat if using standard conv, 
        # but here we concatenate along channel dim.
        # Let's keep it in (H, W, C) format for clarity, then permute for conv.
        
        # Concatenate along channel dim: (H/32, W/32, 3 + m)
        # Note: appearance_embedding is (m,). We need to broadcast it.
        # Create a tensor of shape (H/32, W/32, m) filled with embedding values
        emb_grid = torch.zeros(downsampled.shape[0], downsampled.shape[1], self.m, device=rendered_image.device)
        for i in range(self.m):
            emb_grid[:, :, i] = appearance_embedding[i]
            
        D_i = torch.cat([downsampled, emb_grid], dim=-1) # Shape: H/32, W/32, 3+m
        
        # Permute to (N, C, H, W) for Conv2d
        D_i = D_i.permute(2, 0, 1).unsqueeze(0) # Shape: 1, 3+m, H/32, W/32
        
        # Step 3: Initial Conv
        x = self.conv_init(D_i) # Shape: 1, 256, H/32, W/32
        
        # Step 4: Upsampling Blocks
        for block in self.upsample_blocks:
            x = block(x)
        # After 4 blocks, x is Shape: 1, 16, H, W
            
        # Step 5: Bilinear Interpolation (already at H, W from pixel shuffle, but source specifies it)
        # If pixel shuffle didn't reach full H,W, we interpolate. 
        # 32 -> 16 -> 8 -> 4 -> 2 -> 1 (relative to downsampled). 
        # 4 blocks of 2x upscale = 16x. Downsampled is 1/32. 1/32 * 16 = 1/2. 
        # So we are at H/2, W/2. We need to interpolate to H, W.
        x = F.interpolate(x, size=(H, W), mode='bilinear', align_corners=False)
        
        # Step 6: Final Convs
        x = F.relu(self.conv_final_1(x))
        M_i = self.conv_final_2(x) # Shape: 1, 3, H, W
        
        # Permute back to (H, W, 3)
        M_i = M_i.squeeze(0).permute(1, 2, 0)
        
        # Step 7: Transformation
        # Simple pixel-wise multiplication
        I_ia = rendered_image * M_i
        
        return I_ia, M_i
```

## Subtopics

### WebGL Gaussian Splatting
A rendering implementation targeting web browsers, potentially using different optimization strategies for appearance due to hardware constraints. [Link](2084-webgl-gaussian-splatting.md)

### Debug Splat Point Rendering
A visualization method for debugging individual splat points, lacking the appearance adjustment logic. [Link](2101-debug-splat-point-rendering.md)

### Gaussian Splat Fragment Shader
The low-level shader implementation for rendering splats, which may be modified to incorporate the transformation map if not done in post-processing. [Link](3258-gaussian-splat-fragment-shader.md)

### Gaussian Splat Shader Property Configuration
Configuration of shader properties, which might include passing the transformation map to the shader if integrated differently. [Link](3596-gaussian-splat-shader-property-configuration.md)

### Animated 3D Gaussian Splatting (Splatv rendering)
A method for animating Gaussians, which may require separate appearance modeling for temporal consistency. [Link](7758-animated-3d-gaussian-splatting-splatv-rendering.md)

### Gaussian Splat URP Render Pass
An implementation within Unity's Universal Render Pipeline, which might handle appearance adjustments differently. [Link](17074-gaussian-splat-urp-render-pass.md)

## Operating characteristics
*   **Time Complexity:** The CNN processing is lightweight. On the Sci-Art scene, adding decoupled AM increases training time from 1h25m to 1h33m, a less than 10% increase [S1].
*   **Memory Complexity:** VRAM usage increases from 10.23G to 11.18G, a less than 10% increase [S1].
*   **Performance:** Improves quality metrics significantly: SSIM from 0.858 to 0.885, PSNR from 25.08 to 26.81, and LPIPS from 0.148 to 0.121 on the Sci-Art scene [S1].
*   **Strengths:** Prevents floaters caused by photometric inconsistencies; allows 3D Gaussians to learn consistent geometry; minimal impact on training time and memory; discards overhead at inference time for real-time rendering [S1].
*   **Limitations:** Requires additional training time and memory (though small); the CNN must be discarded after training; simple multiplication is sufficient, limiting the complexity of appearance adjustments that can be learned [S1].
*   **When to use:** When training views have significant appearance variations (e.g., lighting changes) that cause floaters in standard 3DGS.
*   **When not to use:** When real-time inference speed is critical and the overhead of the CNN cannot be avoided (though it is discarded, the training overhead remains); when appearance variations are minimal.

## Background
Decoupled Appearance Modeling was introduced in the paper "VastGaussian: Vast 3D Gaussians for Large Scene Reconstruction" [S1]. It addresses the issue where 3D Gaussian Splatting tries to compensate for photometric changes between training views by creating spurious geometry (floaters) [S1]. By decoupling appearance adjustment from the 3D Gaussian representation, the method allows the Gaussians to converge to a consistent average appearance and correct geometry [S1]. The method was evaluated on large-scale scenes from the Mill-19 and UrbanScene3D datasets [S1].

## Sources
[S1] VastGaussian: Vast 3D Gaussians for Large Scene Reconstruction, https://doi.org/10.1109/cvpr52733.2024.00494, fetched 2026-10-26
