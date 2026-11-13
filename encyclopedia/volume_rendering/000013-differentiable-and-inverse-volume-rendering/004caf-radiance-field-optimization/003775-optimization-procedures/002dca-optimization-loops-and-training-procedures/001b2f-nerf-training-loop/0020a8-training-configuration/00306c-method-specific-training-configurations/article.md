# Method-Specific Training Configurations


## Overview
Method-Specific Training Configurations define the hyperparameters, optimization schedules, and architectural constraints required to train specific Neural Radiance Field (NeRF) variants from multi-view images. This family addresses the problem of converging implicit volumetric representations to match observed photometric data while managing computational cost and training stability. What sets these configurations apart from generic volume rendering is the tight coupling between the sampling strategy (e.g., importance sampling, depth oracle prediction), the network activation functions (e.g., shifted softplus, widened sigmoid), and the loss landscape (e.g., GAN losses, opacity penalties) tailored to the specific representation's high-frequency nature or geometric priors [S1][S3, S6].

## Shared principles
The training process for all methods in this family optimizes the parameters of neural networks (MLPs, hash grids, or synthesis networks) by minimizing the difference between rendered pixel colors and ground truth images. The core loop involves:
1.  **Ray Sampling:** Generating rays from training images using camera intrinsics and extrinsics.
2.  **Point Sampling:** Querying points along these rays, often using adaptive strategies (uniform, stratified, or importance sampling) to concentrate samples in high-density regions.
3.  **Feature Encoding:** Transforming 3D coordinates and viewing directions into feature vectors using positional encodings (Fourier, SIREN, or hash-based).
4.  **Network Forward Pass:** Predicting volume density ($\sigma$) and color ($\mathbf{c}$) or signed distance fields (SDF) and color.
5.  **Volume Rendering:** Computing the accumulated color $\hat{C}$ using the differentiable volume rendering integral:
    $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i $$
    where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, and $\delta_i$ is the distance between samples [S1][S6].
6.  **Loss Computation:** Calculating photometric loss (MSE, Charbonnier, or GAN loss) and applying regularization terms specific to the variant.
7.  **Optimization:** Updating parameters using gradient-based optimizers (typically Adam or AdamW) with specific learning rate schedules.

## Subtopics

### [NeRF Hyperparameters](3065-nerf-hyperparameters.md)
Defines the baseline configuration for standard NeRF training. It addresses the problem of converging an implicit volumetric representation using a two-network pipeline (coarse and fine) with positional encoding. It sets itself apart by specifying the interplay between coarse/fine sampling, the use of positional encoding frequencies ($L_p=10, L_v=4$), and exponential learning rate decay. Choose this for standard supervised NeRF training.

### [Mip-NeRF Optimization and Activation Details](3409-mip-nerf-optimization-and-activation-details.md)
Specifies implementation details for Mip-NeRF and JaxNeRF, focusing on activation functions and input encoding. It solves the problem of training stability and robustness against catastrophic failure modes (e.g., negative density). It distinguishes itself by using Integrated Positional Encoding (IPE) exclusively (removing identity concatenation) and employing shifted softplus and widened sigmoid activations to ensure smoother optimization landscapes. Choose this for anti-aliased, cone-based NeRF variants.

### [DONeRF](4461-donerf.md)
A training configuration that replaces standard uniform raymarching with a two-network pipeline: a depth oracle network and a shading network. It solves the prohibitive computational cost of standard NeRF by reducing sample counts to 2–4 per ray. It sets itself apart by relying entirely on compact MLPs without explicit caching structures, enabling real-time rendering. Choose this for real-time NeRF applications where ground truth depth is available or can be estimated.

### [AR-NeRF Training Configuration](12407-ar-nerf-training-configuration.md)
An unsupervised method combining NeRF with aperture rendering to learn depth and defocus effects from natural images. It solves the problem of estimating per-pixel depth and camera aperture parameters from monocular images without ground-truth metadata. It distinguishes itself by explicitly modeling the camera aperture to render defocus blur, enabling continuous control over focus distance. Choose this for generative NeRF tasks involving defocus and monocular inputs.

### Blender Dataset Simplified Configuration
For the Blender dataset, a simplified configuration is used to speed up training on small synthetic objects. It uses one round of sampling, 128 samples for the proposal MLP, and 32 samples for the NeRF MLP. The proposal MLP has 4 layers with 256 hidden units, while the NeRF MLP has 8 layers with 256 (or 512) hidden units. It employs axis-aligned IPE, MSE loss, and disables the distortion regularizer [S1].

### LLFF Training Configuration
For the LLFF dataset, the distortion regularizer is not disabled, and Charbonnier loss is used instead of MSE [S1]. Performance metrics on LLFF with 256 hidden units show mip-NeRF achieving PSNR 26.93, SSIM 0.830, LPIPS 0.177, Time 2.48 hrs, Params 0.61M, while the proposed model achieves PSNR 26.68, SSIM 0.847, LPIPS 0.150, Time 2.39 hrs, Params 0.84M. With 512 hidden units, mip-NeRF achieves PSNR 27.01, SSIM 0.845, LPIPS 0.148, Time 6.15 hrs, Params 2.27M, while the proposed model achieves PSNR 26.86, SSIM 0.858, LPIPS 0.128, Time 3.84 hrs, Params 3.23M. The proposed model does not outperform mip-NeRF in PSNR but yields improved SSIM and LPIPS metrics and a significant speedup when the NeRF MLP is large [S1].

### NeUDF Training Configuration
Trains Neural Unsigned Distance Fields using the ADAM optimizer. The learning rate warms up to $2 \times 10^{-4}$ in the first 5k iterations and decreases to $1 \times 10^{-5}$ by the end. The batch size is 512 random rays sampled from 8 randomly selected camera poses per iteration. Training lasts for 400k iterations, taking 9 hours with mask and 11 hours without mask on a single Nvidia 3090 GPU [S2].

### NeRF-SDP Training Configuration
Trains NeRF with Scene Depth Perception using the Adam optimizer. The learning rate is $1 \times 10^{-3}$ for the Res-UNet and $5 \times 10^{-4}$ for other parts. The batch size is 500. Training runs for 250,000 steps on an Nvidia GeForce RTX 3090. The first 170,000 steps optimize auxiliary and rendering losses, while the last 80,000 steps optimize only the rendering loss. Feature dimensions are 24 for 2D features and 8 for 3D features. Datasets include LLFF, IBRNet, and Google Scanned [S3].

### Neuralangelo Training Configuration
Trains high-fidelity neural surface reconstruction using hash encoding. It assumes the ROI is inside a unit sphere and runs for 500k total iterations. It samples pixels randomly without tracking statistics to reduce stochastics. When a hash resolution is inactive, feature vectors are set to zero. The learning rate is $1 \times 10^{-3}$ with a linear warmup of 5k iterations, decaying by a factor of 10 at 300k and 400k iterations. It uses AdamW with weight decay $10^{-2}$ and $w_{iek}=0.1$. The SDF MLP has one layer, and the color MLP has four layers. Batch size is 1 for DTU and 16 for Tanks and Temples [S4].

### GO-NeRF Training Configuration
Optimizes a generation model on a single Nvidia 3090 GPU for 20,000 iterations, with 30% of iterations trained with background augmentation. The scheduler values for $\lambda_S$ and $\lambda_O$ range from 30 to 300, and $\lambda_R$ is set to 500 [S5].

### Tri-MipRF Training Configuration
Trains Tri-MipRF using the AdamW optimizer with weight decay $1 \times 10^{-5}$ and learning rate $2 \times 10^{-3}$, scheduled by MultiStepLR. The base mipmap shape $M_{l0}$ is $H=512, W=512, C=16$. The learning rate for Tri-Mip encoding is scaled up 10x relative to the MLP. It uses an area loss $L_{area}$, scaling pixel loss by the area of its footprint on the image plane. Implementation uses PyTorch with tiny-cuda-nn and nvdiffrast [S6].

### NeRF† Implementation Configuration
A variant of nerfacto using a standard MLP density field instead of HashMLPDensityField. For the Blender dataset, $t_n=2.0, t_f=6.0$, camera optimizer, appearance embedding, distortion loss, and scene contraction are disabled, background is white, batch size is 2048, initial sampler is uniform, and training is 16.5k epochs. For LLFF, camera optimizer, appearance embedding, and distortion loss are disabled, scene contraction uses $\ell_\infty$ norm, batch size is 1024 (training) / 2048 (eval), training is 30k epochs, and leaves scene uses uniform initial proposal sampler. On Mip-NeRF 360, nerfacto† achieves PSNR 24.20, SSIM 0.726, LPIPS 0.247. It achieves comparable metrics to nerfacto but is significantly slower [S7].

### DVGO Reference Architecture
Uses Re:NeRF compression method, runs on NVIDIA A40 GPU, and uses PyTorch 1.12 [S8].

### Nerfacto Training Configuration
Uses Adam optimizer with a learning rate schedule of $10^{-2} \to 10^{-4}$, trained for 30k iterations by default [S9, unofficial].

## Background
The concept of method-specific training configurations emerged with the introduction of Neural Radiance Fields (NeRF) by Mildenhall et al. in 2020 [S1, S3, S6]. The original NeRF paper established the baseline hyperparameters, including the two-stage training loop (coarse and fine networks), positional encoding, and exponential learning rate decay. Subsequent works like Mip-NeRF [S1] introduced activation function modifications (shifted softplus, widened sigmoid) and Integrated Positional Encoding to address aliasing and training instability. DONeRF [S3] proposed a depth oracle approach to reduce sample counts for real-time rendering. Other variants like AR-NeRF [S1], NeUDF [S2], NeRF-SDP [S3], Neuralangelo [S4], GO-NeRF [S5], Tri-MipRF [S6], and Nerfacto [S9, unofficial] have further specialized these configurations to address specific challenges such as defocus blur, unsigned distance fields, scene depth perception, high-fidelity surface reconstruction, object generation, anti-aliasing, and production-ready efficiency. These configurations are critical for reproducing results and adapting NeRF to diverse datasets and hardware constraints.

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
[S3] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
[S4] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S5] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
[S6] Tri-MipRF: Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01811, fetched 2026-10-26
[S7] Analyzing the Internals of Neural Radiance Fields, https://doi.org/10.1109/cvprw63382.2024.00288, fetched 2026-10-26
[S8] Two is Better than One: Achieving High-Quality 3D Scene Modeling with a NeRF Ensemble, https://doi.org/10.1007/978-3-031-43153-1_27, fetched 2026-10-26
[S9] NeRF — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-NeRF_SOTA_Updated.html, fetched 2026-10-29 (unofficial)
