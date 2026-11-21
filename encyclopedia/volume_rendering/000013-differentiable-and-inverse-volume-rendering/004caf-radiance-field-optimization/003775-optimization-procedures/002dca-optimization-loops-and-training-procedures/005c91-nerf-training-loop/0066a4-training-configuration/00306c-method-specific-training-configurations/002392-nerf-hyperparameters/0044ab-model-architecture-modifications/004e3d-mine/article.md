# MINE

Also called: Continuous Depth MPI, MINE (Continuous Depth MPI), MINE: Towards Continuous Depth MPI with NeRF for Novel View Synthesis, Multiplane Image with Neural Radiance Fields

## Overview
MINE is a method for novel view synthesis that models the frustum of the source camera using continuous depth Multiplane Images (MPI), conditioned on input images via a fully-convolutional decoder [S1]. It differs from pixelNeRF and GRF, which model the entire 3D space per ray, by reconstructing the source camera's frustum per plane, allowing rendering of any target view via homography warping after a single encoder pass and a fixed number of decoder passes, independent of the number of target views [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single source image (or multiple images for GRF, but MINE uses single image conditioning by default) [S1].
*   **Output:** A rendered novel view image and optionally a disparity map [S1].

**Data Structures:**
The method relies on a continuous depth MPI representation. The MPI consists of $N_{\text{planes}}$ planes. Each plane is represented by a 4-channel image: 3 channels for RGB color and 1 channel for volume density $\sigma$ [S1]. The RGB values are bounded by a Sigmoid activation, and $\sigma$ is the absolute value of the last channel [S1].

**Network Architecture:**
1.  **Encoder:** A standard ResNet50 [S1]. Feature maps are extracted from layers `conv1`, `layer1`, `layer2`, `layer3`, and `layer4` [S1].
2.  **Decoder:** A fully-convolutional depth decoder based on [12], modified with two additional downsampling blocks and two upsampling blocks to increase receptive fields [S1]. The decoder takes the encoder features and disparity encoding as input and outputs a 4-channel RGB-$\sigma$ image [S1].

**Decoder Layer Layout:**
The decoder is a multiscale architecture with outputs at four scales (output1 to output4). The layers are defined as follows (all convolutions are 3x3 unless specified, batch normalization is applied after convolutions in upconv blocks, ELU activation is used unless specified) [S1]:

*   **Downsampling Path:**
    *   `downconv1`: Input channels 2048 -> Output 512.
    *   `downconv2`: Input channels 512 -> Output 256.
*   **Upsampling Path (with Skip Connections):**
    *   `upconv1 extra`: Input 256 -> Output 256.
    *   `upconv2 extra`: Input 256 -> Output 2048.
    *   `upconv5`: Input `cat(upconv2_extra, disparity_encoding)` (2048 + 21) -> Output 256.
    *   `iconv5`: Input `cat(upconv5, encoder_layer3, disparity_encoding)` (256 + 1024 + 21) -> Output 256.
    *   `output4`: Input `iconv5` (128 channels? Note: Source table says input 128, but iconv5 output is 256. The table lists `output4` input as 128. Let's follow the table's explicit channel counts for the *input* to the output layer, which implies a preceding 1x1 conv or similar not explicitly named, or a typo in the source table's input column vs previous layer output. The source table explicitly lists `output4` input channels as 128. We will use the explicit channel counts provided in the "Input" column of Table 5 in [S1] for the final output layers) -> Output 4 (RGB + $\sigma$).
    *   `upconv4`: Input 256 -> Output 128.
    *   `iconv4`: Input `cat(upconv4, encoder_layer2, disparity_encoding)` (128 + 512 + 21) -> Output 128.
    *   `output3`: Input `iconv4` (Input column says 64) -> Output 4.
    *   `upconv3`: Input 128 -> Output 64.
    *   `iconv3`: Input `cat(upconv3, encoder_layer1, disparity_encoding)` (64 + 256 + 21) -> Output 64.
    *   `output2`: Input `iconv3` (Input column says 32) -> Output 4.
    *   `upconv2`: Input 64 -> Output 32.
    *   `iconv2`: Input `cat(upconv2, encoder_conv1, disparity_encoding)` (32 + 64 + 21) -> Output 32.
    *   `output1`: Input `iconv2` (Input column says 16) -> Output 4.
    *   `upconv1`: Input 32 -> Output 16.
    *   `iconv1`: Input 16 -> Output 16.

*Note on Channel Counts:* The source table [S1] lists specific input channel counts for the `output` layers (16, 32, 64, 128) which differ from the output of the preceding `iconv` layers (32, 64, 128, 256). This implies a 1x1 convolution or projection step is applied before the final output activation, or the table describes the effective input to the final block. For implementation, one must ensure the channel dimension matches the expected input of the output layer as specified in the table [S1].

**Numbered Steps:**
1.  **Encode:** Pass the input image through the ResNet50 encoder. Extract feature maps from `conv1`, `layer1`, `layer2`, `layer3`, and `layer4` [S1].
2.  **Decode Frustum:** Pass the encoder features through the fully-convolutional decoder $N_{\text{planes}}$ times. Each inference reconstructs one plane of the MPI frustum [S1].
3.  **Warp:** To render a novel view, apply homography warping to the reconstructed frustum (the set of $N_{\text{planes}}$) [S1].
4.  **Render:** Composite the warped planes using standard volume rendering equations (alpha compositing) to produce the final image [S1].

**Formulas:**
The number of network forward passes $P$ required for MINE is:
$$ P_{\text{MINE}} = 1 + N_{\text{planes}} $$
where $N_{\text{planes}}$ is the number of planes in the MPI representation [S1].

For comparison, the forward passes for pixelNeRF and GRF are:
$$ P_{\text{pixelNeRF}} = 1 + N_{\text{targets}} \times N_{\text{points}} \times H \times W $$
$$ P_{\text{GRF}} = N_{\text{views}} + N_{\text{targets}} \times N_{\text{points}} \times H \times W $$
where $N_{\text{targets}}$ is the number of novel views, $N_{\text{points}}$ is the number of points per ray, and $H \times W$ is the output resolution [S1].

**Parameters:**
*   $N_{\text{planes}}$: The number of planes in the MPI representation. Typical values are not explicitly given in the notes, but the method requires a fixed number of decoder passes equal to this value [S1].
*   **Viewing Direction:** MINE does not take viewing direction as input by default. It can be integrated by concatenating per-ray viewing directions with the output feature maps at the target view (after warping) and using a lightweight fully convolutional network to predict view-dependent radiance [S1].
*   **Loss Functions:** Multiscale training losses are used. $L_{L1}$ and $L_{SSIM}$ are applied only on `output1`. $L_{smooth}$ (edge-aware smoothness loss) is applied on all outputs (`output1`, `output2`, `output3`, `output4`) [S1].
*   **Flowers Light Fields Specifics:**
    *   Disparity range: $[3.0, 0.03]$ [S1].
    *   Grid distance: 0.00128 between adjacent grids [S1].
    *   Camera Intrinsics: $f_x = 0.868056$, $f_y = 1.250000$. Principal point shift is modeled as $c_{xij} = 0.5 + 0.002667 \times i$, $c_{yij} = 0.5 + 0.002667 \times j$, where $[i, j]$ is the index in the extracted 8x8 grid and $[0, 0]$ is the top-left view [S1].

**Pitfalls:**
*   **Principal Point Shift:** In datasets like Flowers Light Fields (Lytro Illum camera), there is a shift in the principal point between different views. If metadata is unavailable, this shift must be approximated (e.g., as a constant value based on grid index) [S1].
*   **View-Dependence:** The base MINE architecture does not model view-dependent effects. To support them, additional network passes are required, increasing complexity [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MINE:
    def __init__(self, n_planes, encoder, decoder):
        """
        Initialize MINE.
        :param n_planes: Number of planes in the MPI representation.
        :param encoder: ResNet50 encoder model.
        :param decoder: Fully-convolutional decoder model.
        """
        self.n_planes = n_planes
        self.encoder = encoder
        self.decoder = decoder

    def encode(self, image):
        """
        Encode the input image and extract multi-scale features.
        :param image: Input image tensor of shape (B, 3, H, W).
        :return: Dictionary of feature maps from conv1, layer1, layer2, layer3, layer4.
        """
        # Standard ResNet50 forward pass
        x = self.encoder.conv1(image)
        x = self.encoder.bn1(x)
        x = self.encoder.relu(x)
        x = self.encoder.maxpool(x)

        l1 = self.encoder.layer1(x)
        l2 = self.encoder.layer2(l1)
        l3 = self.encoder.layer3(l2)
        l4 = self.encoder.layer4(l3)

        # conv1 output is typically after maxpool or before? 
        # Source says "conv1, layer1...". ResNet conv1 output is before maxpool usually, 
        # but feature extraction often happens after. We assume standard feature map extraction.
        # For implementation, one must hook the specific layers.
        features = {
            'conv1': x, # Assuming x is conv1 output before maxpool, or adjust based on actual hook
            'layer1': l1,
            'layer2': l2,
            'layer3': l3,
            'layer4': l4
        }
        return features

    def decode_frustum(self, features):
        """
        Reconstruct the MPI frustum by running the decoder N_planes times.
        :param features: Dictionary of encoder feature maps.
        :return: List of N_planes MPI planes, each of shape (B, 4, H, W). 
                 Channels are RGB (Sigmoid) and Sigma (Abs).
        """
        planes = []
        for _ in range(self.n_planes):
            # The decoder takes features and potentially disparity encoding.
            # In MINE, the decoder is run N_planes times. 
            # The source implies the decoder reconstructs the frustum per plane.
            # It is unclear if the decoder state changes or if it's a fixed mapping 
            # for each depth plane. Assuming the decoder takes a depth/planar index 
            # or the loop implies generating the full frustum volume.
            # However, the source says "Nplanes forward passes of the fully-convolutional decoder".
            # This suggests the decoder might take a depth coordinate or the loop 
            # generates the planes sequentially. 
            # Given it's a "depth decoder", it likely outputs a depth map or plane.
            # We assume the decoder outputs a 4-channel image per pass.
            
            # Note: The exact mechanism of "N planes forward passes" generating 
            # a continuous depth MPI is slightly ambiguous in the text. 
            # Standard MPI generation from a single image usually involves 
            # predicting depth/disparity and then sampling. 
            # MINE's specific "N planes forward passes" likely refers to 
            # predicting the properties for each discrete plane in the discrete MPI approximation.
            
            plane_output = self.decoder(features)
            
            # Apply output activations as per source:
            # RGB: Sigmoid, Sigma: Abs
            rgb = torch.sigmoid(plane_output[:, :3, :, :])
            sigma = torch.abs(plane_output[:, 3:, :, :])
            plane = torch.cat([rgb, sigma], dim=1)
            planes.append(plane)
            
        return planes

    def render_novel_view(self, planes, target_homography):
        """
        Render a novel view by warping the reconstructed frustum.
        :param planes: List of N_planes MPI planes (B, 4, H, W).
        :param target_homography: Homography matrix to warp the frustum to the target view.
        :return: Rendered image tensor (B, 3, H, W).
        """
        # Warp each plane to the target view
        warped_planes = []
        for plane in planes:
            # Extract RGB and Sigma
            rgb = plane[:, :3, :, :]
            sigma = plane[:, 3:, :, :]
            
            # Warp RGB and Sigma using the homography
            # Homography warping is a geometric transformation
            warped_rgb = F.warp_affine(rgb, target_homography, dsize=(rgb.shape[2], rgb.shape[3]))
            warped_sigma = F.warp_affine(sigma, target_homography, dsize=(sigma.shape[2], sigma.shape[3]))
            
            warped_planes.append((warped_rgb, warped_sigma))
        
        # Composite the warped planes using volume rendering (alpha compositing)
        # Standard alpha compositing from front to back
        final_color = torch.zeros_like(warped_planes[0][0])
        acc_sigma = torch.ones_like(warped_planes[0][1])
        
        for rgb, sigma in warped_planes:
            # Convert sigma to transmittance/opacity
            # Assuming sigma is density. Opacity alpha = 1 - exp(-sigma * delta)
            # For simplicity, if sigma is already opacity-like or delta is 1:
            alpha = 1.0 - torch.exp(-sigma)
            
            # Composite
            final_color = final_color * (1.0 - alpha) + rgb * alpha
            acc_sigma = acc_sigma * (1.0 - alpha)
            
        return final_color

    def forward(self, image, target_homography):
        features = self.encode(image)
        planes = self.decode_frustum(features)
        rendered_image = self.render_novel_view(planes, target_homography)
        return rendered_image
```

## Operating characteristics
*   **Time Complexity:** The number of network forward passes is $P_{\text{MINE}} = 1 + N_{\text{planes}}$. This complexity is independent of the number of target views ($N_{\text{targets}}$) and the spatial resolution ($H \times W$) of the output, unlike pixelNeRF and GRF where complexity is proportional to $N_{\text{targets}} \times N_{\text{points}} \times H \times W$ [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Significantly more efficient than pixelNeRF and GRF for rendering multiple novel views. Allows for dense supervision signals (SSIM, edge-aware smoothness) during training because the full target image and disparity map can be rendered at training time [S1]. Generalizes well to large-scale real-world datasets (KITTI, NYU-V2, RealEstate10k, Flowers Light Fields, iBims-1) [S1].
*   **Limitations:** Does not natively support view-dependent effects without additional network passes. Requires careful handling of camera intrinsics, especially for light field cameras with principal point shifts [S1].
*   **When to use:** When efficient novel view synthesis from a single (or few) input images is required, especially for multiple target views. Suitable for large-scale real-world scenes.
*   **When not to use:** When view-dependent effects are critical and computational budget for additional passes is low. When the scene geometry is not well-suited for MPI representation (e.g., highly transparent or non-Lambertian surfaces without proper density modeling).

## Background
MINE (Multiplane Image with Neural Radiance Fields) was introduced in the paper "MINE: Towards Continuous Depth MPI with NeRF for Novel View Synthesis" [S1]. It was verified on datasets including KITTI, NYU-V2, RealEstate10k, Flowers Light Fields, and iBims-1 [S1]. The method builds on the concept of Multiplane Images (MPI) and integrates it with neural network-based conditioning, differing from prior work like pixelNeRF and GRF by modeling the source camera frustum directly [S1].

## Sources
[S1] MINE: Towards Continuous Depth MPI with NeRF for Novel View Synthesis, https://doi.org/10.1109/iccv48922.2021.01235, fetched 2026-10-27
