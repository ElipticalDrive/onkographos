# Denoising via Neural Auxiliary Features

Also called: Neural denoising for path-traced volume rendering, Volumetric neural denoising

## Overview
Denoising via Neural Auxiliary Features learns to reconstruct high-quality images from noisy, low-sample path-traced volume renderings by exploiting spatial and volumetric correlations, trading unbiased Monte Carlo convergence for biased neural reconstruction [S1, unofficial]. It is distinguished from surface-only denoisers by its requirement for specialized volumetric auxiliary features (such as optical thickness and scattering normals) because standard features like depth and albedo are ill-defined or unavailable throughout participating media [S1, unofficial].

## Shared principles
All methods in this family share the following core mechanism:

1.  **Low-Sample Input**: The input is a noisy image rendered with a very low sample count, typically 1–16 samples per pixel (spp), to enable interactive frame rates [S1, unofficial].
2.  **Volumetric Auxiliary Features**: Unlike surface rendering, this family requires auxiliary feature buffers (G-buffers) that capture volumetric properties. Common features include depths, positions, normals from the first surface-like interaction or first scattering event, gradient magnitude, and optical thickness [S1, unofficial].
3.  **Neural Reconstruction**: A neural network (e.g., autoencoder, U-Net, or kernel predictor) takes the noisy radiance and auxiliary features as input to predict a denoised radiance value, effectively learning the mapping from noisy Monte Carlo estimates to clean ground truth [S1, unofficial][S2, unofficial].
4.  **Bias Introduction**: The method introduces bias into the rendering pipeline to achieve visual quality at a fraction of the computational cost of unbiased Monte Carlo convergence [S1, unofficial].

## Subtopics

### [Joint Loss for Sparse-View 3D Gaussian Splatting Denoising](13069-joint-loss-for-sparse-view-3d-gaussian-splatting-denoising.md)
A composite objective function used in the second round of training to refine denoised images, balancing low-frequency agreement with high-frequency fidelity to enhance detail sharpness and texture fidelity while preserving structural consistency.

### [Spatiotemporal Kernel Prediction Network for VPT](13214-spatiotemporal-kernel-prediction-network-for-vpt.md)
A deep learning denoiser for Volume Path Tracing (VPT) image sequences that uses a dual-stream encoder to process G-buffers and radiance separately, and a Multi-Head Spatiotemporal Kernel Prediction Layer (MSKPL) to predict per-pixel temporal and spatial filter kernels for adaptive denoising.

### Dual autoencoder denoising
**Definition**: A dual autoencoder architecture that separates high-frequency direct lighting effects from low-frequency indirect lighting to denoise path-traced medical volumetric data [S1, unofficial].

**Steps**:
1.  Process high-frequency direct lighting via primary features derived from first scatter events, including model-space position, world-space normal, gradient magnitude, and optical thickness [S1, unofficial].
2.  Handle low-frequency indirect lighting via secondary features derived from subsequent scattering events [S1, unofficial].
3.  Combine the processed features using a relativistic discriminator [S1, unofficial].
4.  Apply a loss function designed to preserve specular detail, thereby retaining anatomical detail critical for medical interpretation [S1, unofficial].

**Parameters**:
*   **Training Data**: Approximately 3000 frames of medical CT scans with 1024 spp ground truth [S1, unofficial].
*   **Processing Time**: Approximately 131ms per frame [S1, unofficial].

**Pitfalls**:
*   **Generalization**: Limited generalization beyond medical CT anatomy; PSNR dropped from 32.98 dB to 24.75 dB on unknown scans and transfer functions [S1, unofficial].
*   **Performance**: Processing time of 131ms is unsuitable for interactive visualization [S1, unofficial].
*   **Temporal Stability**: Temporal reprojection is limited to three frames to prevent noise accumulation artifacts [S1, unofficial].

**Performance**:
*   On known scans with unknown transfer functions at 1 spp: 30.50 dB PSNR, 0.915 SSIM [S1, unofficial].
*   Outperformed OpenImageDenoise by approximately 6 dB PSNR [S1, unofficial].
*   Achieved a 1000× sample reduction and approximately 45× speedup for 300-frame video generation [S1, unofficial].

### Neural bilateral grid denoising
**Definition**: A denoising method employing neural bilateral grids with volumetric auxiliary features for path-traced volume rendering [S1, unofficial].

**Parameters**:
*   **Auxiliary Features**: Depths, positions, and normals from the first surface-like interaction and the first scattering event [S1, unofficial].

**Performance**:
*   At 1 spp, it matches the quality of 16 spp rendering [S1, unofficial].
*   Denoising time: 17ms for 1024×1024 resolution [S1, unofficial].
*   Quality metrics: PSNR 30.52, SSIM 0.966 [S1, unofficial].

### Dual-input feature fusion denoising
**Definition**: A denoising method introducing dual-input feature fusion with radiance decomposition weighted by transmittance to separate low-noise Signed Distance Field (SDF) components from high-noise quotient components [S1, unofficial].

**Steps**:
1.  Perform radiance decomposition weighted by transmittance [S1, unofficial].
2.  Separate low-noise SDF components from high-noise quotient components [S1, unofficial].
3.  Use a learned weight map blending mechanism to achieve superior temporal stability [S1, unofficial].

**Performance**:
*   Real-time 4 spp denoising in approximately 20ms at 2560×1440 resolution [S1, unofficial].

### U-Net Based Denoising
**Definition**: A 2D U-Net architecture used for image enhancement or denoising, consisting of an encoder and a decoder, applied to enhance reconstructed CT volumes by removing noise and artifacts [S2, unofficial].

**Steps**:
1.  Load the pre-trained U-Net model from `denoiser.pt` [S2, unofficial].
2.  Normalize the input volume to the [0, 1] range [S2, unofficial].
3.  Process the volume slice-by-slice (axial direction) in inference mode [S2, unofficial].
4.  For each slice:
    *   Convert to tensor [S2, unofficial].
    *   Add batch and channel dimensions [S2, unofficial].
    *   Pass through the model [S2, unofficial].
5.  Store the output slice in the enhanced volume [S2, unofficial].
6.  Normalize the enhanced volume and save [S2, unofficial].
7.  The forward pass applies the encoder to input $x$, then passes the result through the decoder [S2, unofficial].

**Parameters**:
*   **MODEL_PATH**: Path to the trained model weights, default `'denoiser.pt'` [S2, unofficial].

**Layout**:
*   **Encoder `enc`**: Sequential block with `Conv2d(1,32,3,1,1) + ReLU`, then `Conv2d(32,64,3,2,1) + ReLU` [S2, unofficial].
*   **Decoder `dec`**: Sequential block with `ConvTranspose2d(64,32,2,2) + ReLU`, then `Conv2d(32,1,1)` [S2, unofficial].

## Background
Monte Carlo path tracing produces physically accurate volume renderings but requires high sample counts (hundreds to thousands of samples per pixel) to converge to noise-free images [S1, unofficial]. At low sample counts enabling interactive frame rates (1–16 samples per pixel), the resulting images contain high-variance noise that obscures volumetric features [S1, unofficial].

Hofmann et al. [66] pioneered neural denoising for path-traced medical volumetric data [S1, unofficial]. The key challenge identified was that existing surface-only denoisers fail on participating media because auxiliary features such as depth, normals, and albedo are ill-defined or unavailable throughout scattering volumes [S1, unofficial].

Subsequent work by Jabbireddy et al. [70] employed neural bilateral grids, while Xu et al. [169] introduced dual-input feature fusion [S1, unofficial]. Engel and Ropinski [31] explored predicting per-voxel ambient occlusion volumes using 3D convolutional encoder-decoders [S1, unofficial]. Computer graphics denoising methods for surface rendering [15, 179] provide architectural foundations but require significant adaptation for volumetric media [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S2] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
