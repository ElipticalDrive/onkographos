# DINER

Also called: Depth-aware Image-based NEural Radiance fields

## Overview
DINER is a method for novel view synthesis from a sparse set of RGB input views (typically four) with high relative disparity, predicting depth and feature maps to guide the reconstruction of a volumetric scene representation [S1]. It solves the problem of synthesis artifacts and geometry estimation failure that occurs in standard image-based NeRFs when source cameras are far apart, by leveraging dense depth maps instead of sparse landmarks or implicit correspondence [S1]. Unlike siblings like SfMNeRF or DASNeRF which may rely on different geometric priors or optimization strategies, DINER explicitly conditions the neural radiance field on the deviation between sample locations and estimated depth, and employs depth-guided sampling to improve efficiency [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ source RGB images $\{I^{(i)}\}_{i=1}^N$ (typically $N=4$), their camera intrinsics $K^{(i)}$ and extrinsics $P^{(i)}$ [S1].
*   **Outputs:** A differentiable volumetric scene representation (NeRF) capable of rendering novel views $\hat{C}(\mathbf{r})$ [S1].

**Data Structures:**
*   **Feature Maps:** $W^{(i)}$ of size $H \times W \times C_{feat}$ (where $C_{feat}=512$) [S1].
*   **Depth Maps:** $D^{(i)}$ of size $H \times W$ [S1].
*   **Neural Networks:**
    *   Image Encoder $E$: 2D convolutional network [S1].
    *   Depth Estimator: State-of-the-art depth estimation network [S1].
    *   MLP Part 1 ($f_1$): Processes per-view features. Input dimension depends on encoded coordinates, features, and depth deviation. Output dimension is the intermediate feature vector size (512) [S1].
    *   MLP Part 2 ($f_2$): Processes aggregated features. Input dimension is the intermediate feature vector size (512). Outputs color $\mathbf{c}$ (3 channels) and density $\sigma$ (1 channel) [S1].

**Step-by-Step Method:**

1.  **Depth Estimation:** Estimate depth maps $D^{(i)}$ for each source view $i$ using a pre-trained depth estimation network [S1].
2.  **Source Feature Extrapolation:**
    *   Pad each input image $I^{(i)}$ by 64 pixels by repeating border values [S1].
    *   Compute positional encoding for normalized pixel coordinates $\mathbf{uv} \in [-1, +1]$. Use 4 exponentially increasing frequencies starting with 0.5. This results in an 18-channel map. Set this encoding to 0 for unpadded regions [S1].
    *   Concatenate the padded image and the positional encoding map.
    *   Extract per-view feature maps $W^{(i)}$ using the 2D-convolutional encoder $E$:
        $$ W^{(i)} = E(I^{(i)}_{\text{padded}}) $$
        where $I^{(i)}_{\text{padded}}$ includes the positional encoding [S1].
3.  **Ray Sampling:** For a target ray $\mathbf{r}(t) = \mathbf{o} + t \cdot \mathbf{d}$, generate candidate samples. DINER uses depth-guided sampling:
    *   Generate 1000 candidate samples per ray [S1].
    *   Shortlist 25 samples based on proximity to estimated depth [S1].
    *   Add 15 samples via Gaussian boosting around the shortlisted points [S1].
    *   Total samples contributing to final color: 40 [S1].
    *   *Note:* Normal maps for backface culling are obtained by calculating central differences on depth maps using $3 \times 3$ convolutional kernels, filtering out foreground-background edges [S1].
4.  **Per-View Feature Processing:** For each sample point $\mathbf{x}_j$ on the ray and each source view $i$:
    *   Transform $\mathbf{x}_j$ and viewing direction $\mathbf{d}$ to source view coordinates:
        $$ \mathbf{x}^{(i)} = P^{(i)} \circ \mathbf{x}_j $$
        $$ \mathbf{d}^{(i)} = R^{(i)} \circ \mathbf{d} $$
        where $P^{(i)}$ and $R^{(i)}$ are the extrinsic rotation/translation components [S1].
    *   Project $\mathbf{x}^{(i)}$ onto the feature map $W^{(i)}$ to interpolate the feature vector $\boldsymbol{\omega}^{(i)}$ using bilinear interpolation:
        $$ \boldsymbol{\omega}^{(i)} = W^{(i)}[K^{(i)} \circ \mathbf{x}^{(i)}] $$
        where $K^{(i)}$ are the intrinsic parameters [S1].
    *   Compute the deviation $\Delta z$ between the sample's z-coordinate in the source camera frame and the estimated depth $D^{(i)}$ at the projected pixel location [S1].
    *   Condition the NeRF on the positionally encoded distance $\Delta z$. Use 6 frequency channels with a base frequency of 1 meter, resulting in a 13-dimensional vector [S1].
    *   Concatenate the 13-dimensional encoded $\Delta z$ with the 512-dimensional feature vector $\boldsymbol{\omega}^{(i)}$ [S1].
    *   Process $\mathbf{x}^{(i)}$, $\mathbf{d}^{(i)}$, $\boldsymbol{\omega}^{(i)}$, and the encoded $\Delta z$ through MLP part $f_1$ to generate intermediate feature vectors $\mathbf{V}^{(i)}$:
        $$ \mathbf{V}^{(i)} = f_1(\mathbf{x}^{(i)}, \mathbf{d}^{(i)}, \boldsymbol{\omega}^{(i)}, \Delta z) $$
        [S1].
5.  **Feature Aggregation:** Average pool the intermediate feature vectors across all $N$ views:
    $$ \bar{\mathbf{V}} = \text{mean}_i (\mathbf{V}^{(i)}) $$
    [S1].
6.  **Color and Density Regression:** Process the averaged features through MLP part $f_2$ to regress final color $\mathbf{c}$ and opacity $\sigma$:
    $$ (\mathbf{c}, \sigma) = f_2(\bar{\mathbf{V}}) $$
    [S1].
7.  **Volumetric Rendering:** Render the final color $\hat{C}(\mathbf{r})$ using standard volumetric rendering integration:
    $$ \hat{C}(\mathbf{r}) = \sum_{j=1}^{M} T_j (1 - \exp(-\sigma_j \delta_j)) \mathbf{c}_j $$
    where $M=40$ is the number of samples, $\delta_j$ is the distance between adjacent samples, and $T_j = \exp\left(-\sum_{k=1}^{j-1} \sigma_k \delta_k\right)$ is the transmittance [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DINER:
    def __init__(self, N=4, num_samples=40, num_candidates=1000):
        self.N = N
        self.num_samples = num_samples
        self.num_candidates = num_candidates
        # Placeholder for actual network definitions from pixelNeRF/DINER paper
        self.image_encoder = ImageEncoder() 
        self.depth_estimator = DepthEstimator()
        self.mlp_f1 = MLP_F1()
        self.mlp_f2 = MLP_F2()
        
    def positional_encode_depth(self, delta_z, num_freq_bands=6, freq_base=1.0):
        """
        Encodes the depth deviation delta_z.
        Uses 6 frequency channels with base frequency 1 meter.
        Returns 13-dim vector (original + 6 sin/cos pairs).
        """
        # Implementation of NeRF positional encoding for 1D scalar
        scaled_z = delta_z * freq_base
        scales = torch.tensor([2**i for i in range(num_freq_bands)], device=delta_z.device)
        scaled_z = scaled_z * scales * torch.pi
        pe = torch.cat([torch.sin(scaled_z), torch.cos(scaled_z)], dim=-1)
        # Concatenate original value? The paper says "13-dimensional vector" 
        # from 6 freq channels. 6*2 + 1 = 13. So yes, include original.
        return torch.cat([delta_z, pe], dim=-1)

    def positional_encode_uv(self, uv_coords):
        """
        Encodes UV coordinates for image padding extrapolation.
        4 exponentially increasing frequencies starting with 0.5.
        Returns 18-channel map (4 * 2 * 2? No, 4 freqs * 2 (sin/cos) * 2 (u,v) = 16? 
        Paper says 18. Let's assume standard 2D PE: 4 freqs * 2 (sin/cos) * 2 (dims) = 16.
        Wait, paper says "18-channel map". 
        Let's look at source: "positional encoding map has a channel size of 18".
        Standard PE for 2D with 4 bands: 4 * 2 * 2 = 16. 
        Maybe it includes the original 2 channels? 16 + 2 = 18.
        """
        # uv_coords shape: [H, W, 2], values in [-1, 1]
        freq_bands = 4
        freqs = torch.tensor([0.5 * (2**i) for i in range(freq_bands)], device=uv_coords.device)
        # Expand for broadcasting
        freqs = freqs.view(1, 1, 1, -1)
        uv = uv_coords.unsqueeze(-1) # [H, W, 2, 1]
        scaled_uv = uv * freqs * torch.pi
        pe = torch.cat([torch.sin(scaled_uv), torch.cos(scaled_uv)], dim=-2) # [H, W, 8, 4] -> [H, W, 4, 8]
        pe = pe.permute(0, 1, 3, 2) # [H, W, 8, 4] -> [H, W, 4, 8] wait.
        # Let's stick to the result dimension: 18.
        # If we concat original UV (2) + 4 bands * 2 (sin/cos) * 2 (u,v) = 16. Total 18.
        pe_full = torch.cat([uv.squeeze(-1), pe.permute(0,1,3,2).reshape(uv.shape[0], uv.shape[1], -1)], dim=-1)
        return pe_full

    def forward(self, source_images, intrinsics, extrinsics):
        """
        source_images: [N, 3, H, W]
        intrinsics: [N, 3, 3]
        extrinsics: [N, 4, 4] (or similar format)
        """
        N = source_images.shape[0]
        
        # 1. Depth Estimation
        depth_maps = self.depth_estimator(source_images) # [N, 1, H, W]
        
        # 2. Feature Extraction with Extrapolation
        # Pad images by 64 pixels
        pad_size = 64
        padded_images = F.pad(source_images, (pad_size, pad_size, pad_size, pad_size), mode='replicate')
        
        # Compute UV coordinates for the padded image
        # Assuming standard grid generation
        H_p, W_p = padded_images.shape[2], padded_images.shape[3]
        y, x = torch.meshgrid(torch.linspace(-1, 1, H_p), torch.linspace(-1, 1, W_p), indexing='ij')
        uv_coords = torch.stack([x, y], dim=-1).unsqueeze(0).expand(N, -1, -1, -1, -1) # [N, H, W, 2]
        
        # Positional Encoding for UV
        pe_uv = self.positional_encode_uv(uv_coords) # [N, H, W, 18]
        
        # Concatenate padded image (3 channels) and PE (18 channels) -> 21 channels
        # Note: PE is 0 in unpadded regions.
        # The encoder E takes this concatenated input.
        # We need to mask the PE in unpadded regions to 0.
        # Create mask for padded region
        mask = torch.zeros((N, 1, H_p, W_p), device=source_images.device)
        mask[:, :, pad_size:-pad_size, pad_size:-pad_size] = 1.0
        
        # Apply mask to PE
        pe_uv_masked = pe_uv * mask.unsqueeze(-1)
        
        # Combine image and PE
        # Image is [N, 3, H, W]. PE is [N, H, W, 18].
        # Need to align dimensions.
        img_pe = torch.cat([padded_images, pe_uv_masked.permute(0, 3, 1, 2)], dim=1) # [N, 21, H, W]
        
        # Extract features
        feature_maps = self.image_encoder(img_pe) # [N, 512, H, W]
        
        # 3. Ray Casting and Rendering (Simplified for one ray)
        # In practice, this is done for all pixels in parallel
        pass

    def render_ray(self, ray_o, ray_d, intrinsics, extrinsics, feature_maps, depth_maps):
        """
        Renders a single ray.
        """
        N = feature_maps.shape[0]
        
        # Depth-guided sampling
        # 1. Get depth at ray intersection with near/far planes or just use depth maps to guide
        # The paper says: "generate 1000 candidate samples per ray from which we shortlist 25"
        # This implies projecting the ray onto depth maps to find surface locations.
        
        # For each source view i:
        # Project ray to 2D to get depth D^(i)
        # Sample around D^(i)
        
        # Let's assume we have a list of 40 sample depths t_j along the ray direction d
        # These are determined by the depth maps.
        
        # For each sample j:
        # 1. Compute 3D point x_j = o + t_j * d
        # 2. For each view i:
        #    a. Transform x_j to view i coords: x^(i)
        #    b. Project x^(i) to 2D: uv^(i)
        #    c. Interpolate feature w^(i) from feature_maps[i]
        #    d. Get depth D^(i) at uv^(i)
        #    e. Compute delta_z = z^(i) - D^(i)
        #    f. Encode delta_z
        #    g. Compute V^(i) = f1(x^(i), d^(i), w^(i), encoded_delta_z)
        # 3. Aggregate V^(i)
        # 4. Compute c, sigma = f2(mean(V))
        
        # Volumetric rendering
        pass
```

**Parameters:**
*   $N$: Number of source images. Typical value: 4 [S1].
*   $f_1, f_2$: MLP parts. $f_1$ input dimension is adjusted to accommodate the 13-dim encoded depth deviation and 512-dim feature vector. $f_2$ input dimension is 512 [S1].
*   Patch size for loss evaluation: $64 \times 64$ pixels (downsampled to $8 \times 8$ for the Lab dataset) [S1].
*   Loss weights: $w_l = 1.0$ (L1), $w_{vgg} = 0.1$ (Perceptual), $w_{ab} = 5.0$ (Anti-bias, or $1.0$ for DTU dataset) [S1].

**Edge Cases and Pitfalls:**
*   **Extrapolation:** Standard image-based NeRFs suffer from artifacts when rendering outside the source view frustums. DINER handles this by padding input images by 64 pixels and applying positional encoding to normalized UV coordinates, setting the encoding to 0 in unpadded regions [S1].
*   **Large Disparity:** Implicit correspondence search fails when the distance between source cameras is large. DINER uses dense depth maps instead of sparse landmarks, which are not restricted to specific object categories and provide denser guidance [S1].
*   **Depth Accuracy:** Synthesis quality improves with depth accuracy up to a standard deviation of 1mm noise. Higher accuracy yields no further improvement [S1].
*   **Backface Culling:** Normal maps are calculated via central difference on depth maps using $3 \times 3$ kernels. Foreground-background edges are filtered out to prevent artifacts [S1].

## Subtopics

### SfMNeRF
Uses structure-from-motion points for initialization. Chooses when sparse geometry is available. [SfMNeRF](8664-sfmnerf.md)

### DASNeRF
Uses depth-aware sampling but may differ in feature fusion. Chooses when depth is integrated differently. [DASNeRF](8668-dasnerf.md)

### Neural RGB-D Surface Reconstruction
Focuses on surface reconstruction from RGB-D. Chooses when explicit surface is needed. [Neural RGB-D Surface Reconstruction](11352-neural-rgb-d-surface-reconstruction.md)

## Operating characteristics
*   **Time Complexity:** Rendering a $256 \times 256$ image takes approximately two seconds [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On the DTU dataset, DINER achieves higher PSNR and SSIM compared to baselines like NeuRay and MVSNeRF [S1].
*   **Scaling:** Performance is bound by the accuracy of the depth prediction network. Replacing it with a Kinect-like sensor could improve results [S1].
*   **Strengths:** Handles input views with high relative distance; improves sampling efficiency (depth-guided sampling reduces median Chamfer distance by 33% and max Chamfer distance by a factor of 4 compared to coarse-to-fine sampling with the same number of points); no need for per-scene optimization [S1].
*   **Limitations:** Slow rendering speed (2s per image); relies on off-the-shelf depth estimator accuracy; not real-time capable [S1].
*   **When to use:** Novel view synthesis from sparse, widely spaced input views where geometry estimation is difficult for standard NeRFs.
*   **When not to use:** Real-time applications; scenes where depth estimation is unreliable.

## Background
DINER (Depth-aware Image-based NEural Radiance fields) was proposed by Malte Prinzler, Justus Thies, and Otmar Hilliges in 2023 [S1]. It builds on the pixelNeRF pipeline and addresses limitations in previous image-based NeRF methods that relied on sparse landmarks or implicit correspondence, which failed with large camera disparities [S1]. The method was evaluated on the FaceScape dataset for human heads and the DTU dataset for general objects [S1].

## Sources
[S1] DINER: Depth-aware Image-based NEural Radiance fields, https://doi.org/10.1109/cvpr52729.2023.01198, fetched 2026-10-29
