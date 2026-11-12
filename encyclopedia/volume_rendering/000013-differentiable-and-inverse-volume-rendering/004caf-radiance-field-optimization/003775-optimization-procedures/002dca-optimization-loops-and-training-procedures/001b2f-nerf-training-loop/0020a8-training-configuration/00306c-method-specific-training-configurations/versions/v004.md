# Method-Specific Training Configurations


## Overview
Method-Specific Training Configurations define the hyperparameters, optimization schedules, and architectural constraints required to train specific Neural Radiance Field (NeRF) variants and related implicit surface representations. This family of methods addresses the problem of stabilizing convergence and maximizing reconstruction fidelity for diverse network architectures, ranging from standard MLP-based NeRFs to compact dual-network systems and hash-encoded surface models. It differs from generic training loops by providing exact numerical values for learning rates, batch sizes, activation functions, and loss weights that are critical for the specific mathematical properties of each variant, such as handling negative density values or managing memory for high-resolution hash grids.

## Shared principles
The shared principles of method-specific training configurations involve the precise instantiation of the optimization loop parameters that govern the gradient descent process for a given neural representation. While the underlying volume rendering equation remains consistent across most methods, the specific configuration dictates how the neural network is regularized and sampled.

Common elements include:
1.  **Optimizer Selection:** Most methods utilize variants of the Adam optimizer [S1, S2, S3, S5, S6].
2.  **Learning Rate Scheduling:** Configurations often specify initial learning rates, warmup periods, and decay schedules (exponential, linear, or multi-step) to ensure stable convergence [S1, S2, S3, S5, S6].
3.  **Batch Composition:** The definition of a training batch, whether defined by the number of rays (e.g., 512, 1024, 4096) or pixels, and the sampling strategy (random, stratified, or hierarchical) [S1, S2, S3, S5, S6].
4.  **Loss Function Weighting:** The specific coefficients applied to different components of the total loss, such as the balance between photometric MSE and regularization terms like opacity or curvature [S2, S3, S4, S6].
5.  **Network Initialization and Activation:** Specific choices for activation functions (e.g., shifted softplus, widened sigmoid) and initialization biases that prevent common failure modes like negative density or saturation [S1, S3].

## Subtopics

### [NeRF Hyperparameters](3065-nerf-hyperparameters.md)
The standard NeRF configuration defines the baseline for training 8-layer position MLPs and 4-layer direction MLPs. It uses stratified sampling for coarse passes (64 samples) and hierarchical sampling for fine passes (128 samples). The optimizer is Adam with an initial learning rate of $5 \times 10^{-4}$, decaying to $5 \times 10^{-6}$ or $1 \times 10^{-4}$. It is chosen for its robustness on synthetic datasets like Blender and LLFF, serving as the reference point for other variants.

### [Mip-NeRF Optimization and Activation Details](3409-mip-nerf-optimization-and-activation-details.md)
Mip-NeRF modifies the standard NeRF training loop to handle anti-aliasing via Integrated Positional Encoding (IPE). It replaces standard ReLU with shifted softplus for density and widened sigmoid for color to prevent catastrophic failure from negative densities or saturation. It uses a specific warmup learning rate schedule and 1,000,000 iterations. It is chosen when anti-aliasing and robustness to varying focal lengths are required.

### [DONeRF](4461-donerf.md)
DONeRF uses a dual-network architecture: a Depth Oracle Network and a Shading Network. It replaces dense raymarching with a local sampling strategy guided by the oracle, reducing samples per ray to 2–16. It requires ground truth depth maps (or a pre-trained dense NeRF) and uses a Binary Cross-Entropy loss for the oracle and an Opacity Loss for the shading network. It is chosen for real-time rendering on consumer hardware where computational cost is prohibitive for standard NeRF.

### NeUDF Training Configuration
*   **Optimizer:** ADAM [S1].
*   **Learning Rate:** Warms up to $2 \times 10^{-4}$ in the first 5,000 iterations, then decreases to $1 \times 10^{-5}$ by the end [S1].
*   **Batch Size:** 512 random rays sampled from 8 randomly selected camera poses per iteration [S1].
*   **Training Duration:** 400,000 iterations total (9 hours with mask, 11 hours without mask on a single Nvidia 3090 GPU) [S1].

### NeRF-SDP Training Configuration
*   **Optimizer:** Adam [S2].
*   **Learning Rate:** $1 \times 10^{-3}$ for Res-UNet, $5 \times 10^{-4}$ for other parts [S2].
*   **Batch Size:** 500 [S2].
*   **Hardware:** Nvidia GeForce RTX 3090 [S2].
*   **Total Steps:** 250,000. First 170,000 steps optimize auxiliary and rendering losses; last 80,000 steps optimize only rendering loss [S2].
*   **Feature Dimensions:** 2D features 24, 3D features 8 [S2].
*   **Datasets:** LLFF, IBRNet, Google Scanned [S2].

### Neuralangelo Training Configuration
*   **Sampling:** Samples pixels from images randomly without tracking statistics/errors; uses long training iterations to reduce stochastics [S3].
*   **ROI:** Assumed to be inside a unit sphere [S3].
*   **Total Training Iterations:** 500,000 [S3].
*   **Inactive Hash Resolution Handling:** Feature vectors are set to zero when a given hash resolution is not active [S3].
*   **Learning Rate:** $1 \times 10^{-3}$ with a linear warmup of 5,000 iterations. Decays by a factor of 10 at 300,000 and 400,000 iterations [S3].
*   **Optimizer:** AdamW with weight decay of $10^{-2}$. $w_{iek}$ is set to 0.1 [S3].
*   **MLP Architecture:** SDF MLP has one layer; color MLP has four layers [S3].
*   **Batch Size:** 1 for DTU benchmark, 16 for Tanks and Temples dataset [S3].

### GO-NeRF Training Configuration
*   **Optimization:** Optimize the generation model on a single Nvidia 3090 GPU for a total of 20,000 iterations [S4].
*   **Augmentation:** Train 30% of the iterations with background augmentation [S4].
*   **Scheduler Values:** Range from 30 to 300 [S4].
*   **Lambda R:** Set to 500 [S4].

### Tri-MipRF Training Configuration
*   **Base Mipmap Shape:** $M_{l0}$: H=512, W=512, C=16 [S5].
*   **Optimizer:** AdamW with weight decay $1 \times 10^{-5}$ and learning rate $2 \times 10^{-3}$ [S5].
*   **Learning Rate Schedule:** MultiStepLR [S5].
*   **Tri-Mip Encoding LR:** Scaled up 10x relative to the MLP [S5].
*   **Loss Function:** Area loss $L_{area}$, scaling pixel loss by the area of its footprint on the image plane [S5].
*   **Implementation Framework:** PyTorch with tiny-cuda-nn extension and nvdiffrast library for mipmap query [S5].

### NeRF† Implementation Configuration
*   **Definition:** A variant of nerfacto that uses a standard MLP density field instead of HashMLPDensityField, with specific hyperparameter settings per dataset [S6].
*   **Blender Dataset:** $t_n = 2.0$, $t_f = 6.0$. Camera optimizer, appearance embedding, distortion loss, and scene contraction are disabled. Background color is manually set to white. Batch size = 2048 rays. Initial sampler = uniform. Training epochs = $16.5 \times 10^3$ [S6].
*   **LLFF Dataset:** Camera optimizer, appearance embedding, and distortion loss are disabled. Scene contraction uses $\ell_\infty$ norm. Batch size = 1024 rays during training, 2048 during evaluation. Training epochs = $30 \times 10^3$. Leaves scene uses uniform initial proposal sampler [S6].
*   **Performance:** On Mip-NeRF 360, nerfacto† achieves PSNR 24.20, SSIM 0.726, LPIPS 0.247 [S6].
*   **Comparison:** Compared to nerfacto (HashMLPDensityField), nerfacto† achieves comparable PSNR/SSIM/LPIPS on Mip-NeRF 360 but is significantly slower [S6].

## Background
The concept of method-specific training configurations emerged with the introduction of Neural Radiance Fields (NeRF) by Mildenhall et al. in 2020 [S3, S4]. The original NeRF paper established the baseline hyperparameters (8-layer MLP, Adam optimizer, specific learning rates) that became the standard reference. Subsequent works identified that these generic settings were insufficient for new architectures or specific challenges like anti-aliasing (Mip-NeRF) or computational efficiency (DONeRF). As the field expanded to include surface reconstruction (Neuralangelo) and generalizable models (NeRF-SDP), each variant required distinct training schedules, loss functions, and architectural constraints to converge properly. The "Method-Specific Training Configurations" family aggregates these empirical findings, which are often scattered across supplementary materials and implementation details in various research papers [S1, S2, S3, S4, S5, S6].

## Sources
[S1] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
[S2] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
[S3] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S4] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
[S5] Tri-MipRF: Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01811, fetched 2026-10-26
[S6] Analyzing the Internals of Neural Radiance Fields, https://doi.org/10.1109/cvprw63382.2024.00288, fetched 2026-10-26
