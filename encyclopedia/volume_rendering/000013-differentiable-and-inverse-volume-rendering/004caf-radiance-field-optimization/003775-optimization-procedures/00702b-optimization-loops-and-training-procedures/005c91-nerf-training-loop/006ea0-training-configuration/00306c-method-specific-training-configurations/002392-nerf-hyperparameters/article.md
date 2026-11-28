# NeRF Hyperparameters

Also called: Hyperparameters, NeRF Training Hyperparameters, Training Hyperparameters, ViCA-NeRF Training

## Overview
NeRF hyperparameters define the specific configuration of the optimization loop, network architecture, and sampling strategies required to train a Neural Radiance Field (NeRF) from a set of posed images. While the parent family shares the goal of minimizing photometric error via differentiable volume rendering, this article specifies the concrete values for learning rates, batch sizes, sample counts, and architectural dimensions that distinguish standard NeRF training from its variants. It serves as the baseline configuration against which specialized methods (e.g., [Mega-NeRF](8766-mega-nerf-training-configuration.md) or [AR-NeRF](12407-ar-nerf-training-configuration.md)) adjust their parameters to address specific constraints like memory, speed, or artifact suppression.

## How it works
The training process minimizes the difference between rendered images and ground truth inputs by optimizing the parameters of a Multi-Layer Perceptron (MLP) that maps 3D coordinates and viewing directions to volume density ($\sigma$) and RGB color ($\mathbf{c}$).

### Inputs and Outputs
- **Inputs**: A dataset of $N$ images with known camera poses (extrinsics and intrinsics), typically normalized such that pixel values are in $[0, 1]$ [S9, unofficial].
- **Outputs**: A trained NeRF model (MLP weights) and optionally a cached density field or mesh for acceleration [S8, unofficial].

### Data Structures
The core data structure is the NeRF MLP, defined by its depth and width.
- **Position MLP**: Maps $(\mathbf{x}, \mathbf{d})$ to intermediate features and density.
- **Color MLP**: Maps intermediate features and viewing direction $\mathbf{d}$ to RGB color.
- **Sampling Points**: For each ray, a set of query points $\mathbf{x}_i$ and depths $t_i$ are generated between near and far clipping planes.

### Numbered Steps
1. **Data Loading**: Load images and camera parameters. Normalize pixel values by dividing by 255.0 [S9, unofficial]. Reserve one image for validation/testing [S9, unofficial].
2. **Ray Generation**: For each training step, sample a batch of rays. A common batch size is 1024 rays [S4][S7, S9, unofficial].
3. **Query Point Sampling**:
   - **Coarse Stage**: Sample $N_c$ points uniformly or via stratified sampling along the ray [S7, S8, unofficial]. Typical $N_c = 64$ [S7, S8, unofficial].
   - **Fine Stage**: Sample $N_f$ points using importance sampling based on the density predicted by the coarse network. Typical $N_f = 128$ for synthetic data, $64$ for LLFF [S8, unofficial].
4. **Network Forward Pass**:
   - Query the Position MLP with $(\mathbf{x}_i, \mathbf{d})$ to get density $\sigma_i$ and features.
   - Query the Color MLP with features and $\mathbf{d}$ to get color $\mathbf{c}_i$.
   - Apply tone mapping (e.g., gamma function) to $\mathbf{c}_i$ if the blur model requires scene irradiance [S4].
5. **Volume Rendering**: Compute the rendered color $C(r)$ for ray $r$ using the standard volume rendering equation:
   $$ C(r) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i $$
   where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, $\delta_i$ is the distance between samples, and $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance [G].
6. **Loss Computation**: Calculate the photometric loss (e.g., L1 or L2) between $C(r)$ and the ground truth pixel color [S3][S7, unofficial].
7. **Optimization**: Update model parameters using an optimizer (typically Adam) [S5, S8, unofficial].
8. **Iteration**: Repeat for a total number of iterations (e.g., 40,000 to 300,000) [S3][S8, S9, unofficial].

### Formulas and Parameters
- **Learning Rate Schedule**: Exponential decay is common.
  - Standard NeRF: Starts at $5 \times 10^{-4}$, decays to $5 \times 10^{-5}$ [S5, unofficial] or $8 \times 10^{-5}$ [S4].
  - FastNeRF: Starts at $5 \times 10^{-4}$ [S8, unofficial].
- **Optimizer**: Adam with $\beta_1 = 0.9$, $\beta_2 = 0.999$, $\epsilon = 10^{-7}$ [S5, S8, unofficial].
- **Network Architecture**:
  - Position MLP: 8 layers, 256 hidden units [S1][S7, unofficial].
  - Color MLP: 4 layers, 128 hidden units [S8, unofficial].
  - Positional Encoding: 10 frequencies for position, 4 for direction [S7, unofficial].
- **Batch Size**: 1024 rays per step [S4][S7, S9, unofficial].
- **Chunk Size**: To manage memory, points are processed in chunks (e.g., 4096 points) [S9, unofficial].

### Edge Cases and Pitfalls
- **Memory Overflow**: Large batch sizes or high-resolution rendering can exceed GPU memory. Use chunking [S9, unofficial] or reduce batch size.
- **Overfitting**: High-frequency positional encodings can lead to noise or floaters. Regularization or frequency tuning may be required [S18, unofficial].
- **View-Dependent Artifacts**: Direction-dependent flickering can occur. Mitigated by reducing Fourier encoding dimensionality for directions ($L=1$) or using a direction-dependent cache [S8, unofficial].

## Subtopics

### [NeRF Initialization Strategies](8674-nerf-initialization-strategies.md)
Addresses slow convergence by using pre-trained weights from 3D point clouds or 3D Morphable Models (3DMM) to provide an informed starting point, reducing training time.

### [Mega-NeRF Training Configuration](8766-mega-nerf-training-configuration.md)
Accelerates training for large scenes by sharing a single MLP between coarse and fine sampling stages, reducing model queries by 25%.

### [Alpha Invariance / Inverse Scaling](10127-alpha-invariance-inverse-scaling.md)
Ensures opacity remains constant under scene scaling by inversely scaling volume density, solving the issue where density magnitude is tied to the integration domain.

### [Batch size scaling for computational parity](10942-batch-size-scaling-for-computational-parity.md)
Adjusts batch size to maintain consistent computational load across different hardware or resolution settings [G].

### [Model Architecture Modifications](11313-model-architecture-modifications.md)
Alters the baseline MLP topology or input space to handle dynamic scenes, noise, or semantic understanding, distinct from hyperparameter tuning.

### [Regularization Techniques](11314-regularization-techniques.md)
Applies losses to suppress artifacts (floaters, noise) and enforce priors, operating on the gradient updates rather than architecture [G].

### [NeRF Pre-processing](12110-nerf-pre-processing.md)
Normalizes image features and camera geometry to stabilize MLP training by ensuring inputs fall within the effective range of positional encodings.

### [Exponential Learning Rate Decay with Delay](12969-exponential-learning-rate-decay-with-delay.md)
A learning rate schedule that applies a reverse cosine decay during an initial delay period before transitioning to log-linear decay.

### [Adaptive Time-step Training Framework](13342-adaptive-time-step-training-framework.md)
Reduces inference time steps in spike-based NeRFs by learning a scene-wise target time step, preserving parallel rendering patterns.

### [Model Architecture and Encoding](13550-model-architecture-and-encoding.md)
Defines the canonical NeRF architecture with sinusoidal positional encoding to overcome spectral bias, serving as the baseline for variants.

### [Sampling and Rendering Pipeline Configuration](15626-sampling-and-rendering-pipeline-configuration.md)
Controls ray-sampling mechanics, sample density, and noise injection to balance fidelity and cost.

### [Method-Specific Training Pipelines](15627-method-specific-training-pipelines.md)
Integrates domain-specific constraints (e.g., motion blur, GANs) into the optimization loop, deviating from standard baseline implementations [G].

### [Iterative 3D Super-Resolution](16434-iterative-3d-super-resolution.md)
A two-stage framework using diffusion-based upscaling and 3D synchronization to super-resolve NeRFs, reducing optimization duration by 4×.

### [Random Patch Sampling](16594-random-patch-sampling.md)
Extracts random image patches from ray space to prevent CNN-based SR modules from overfitting and to model defocus blur.

### [Equal-Event Event Binning](16911-equal-event-event-binning.md)
Divides event streams into bins with equal event counts to leverage temporal blur priors for HDR NeRF training.

### [MI-shaping](18674-mi-shaping.md)
Applies a contrastive loss on pixel gradient weights to align the gradient space with semantic similarity, improving label propagation.

### [NeRF Model Setup and Configuration](20253-nerf-model-setup-and-configuration.md)
Deterministically initializes the scene bounding volume, occupancy grids, and sampling constants before training [G].

### [Magic123 Coarse Stage](20303-magic123-coarse-stage.md)
Trains a NeRF from a single image using Instant-NGP and multi-modal regularization (depth/normal smoothness) from 2D diffusion priors.

### DietNeRF
Enhances few-shot NVS by introducing an auxiliary semantic consistency loss that matches high-level features across random poses [S1, S2].

### NeRF Training Configuration for View-Dependent Artifacts
Mitigates direction-dependent flickering by using a smaller direction-dependent cache ($16^3$ to $64^3$) or reducing Fourier encoding dimensionality for directions to $L=1$ [S8, unofficial].

### RoGUENeRF Training and Loss
Pre-trains on LLFF for 3000 epochs, then fine-tunes. Uses a loss combining L1 photometric error and VGG-19 perceptual loss [S10].

### Learned Exposure Scaling
Learns exposure scaling offsets for RawNeRF, parameterized as embeddings constrained to zero for exposure index 0 [S11, unofficial].

### Reliability-aware NeRF
Suppresses contradictory supervision from unreliable regions (occluded, reflective) by filtering view pairs and correspondences based on overlap and reprojection error [S12].

### BARF (Blending Artificially Realized Frequency)
Stabilizes pose optimization using a coarse-to-fine positional encoding schedule gated by a cosine-annealed scalar $\alpha(t)$ [S13].

### LU-NeRF-1
A lightweight model for joint pose and scene estimation in mini-scenes, using a 4-layer MLP without positional encoding and strong depth total variation regularization [S14].

### Dimension Refinement
Resolves jagged edges by linearly doubling the NeRF rendering resolution from 64 to 128 during a late-stage refinement phase [S15, unofficial].

### Elastic Weight Consolidation (EWC)
Regularizes parameters to stay close to previous weights, weighted by Fisher information importance [S16].

### FreeNeRF Frequency Regularization
Implements frequency regularization with a 90% schedule while disabling occlusion regularization to prevent floaters [S17].

### Frequency Domain Consistency Regularization
Tunes positional encoding frequency by enforcing consistency in the frequency domain of rendered images from slightly different viewpoints [S18, unofficial].

### Nerflets Training Configuration
Uses a higher semantic loss weight early in training, with specific ray sampling (64 coarse, 128 fine) and dynamic loss weights [S19].

### OmniSDF Training and Loss
Trains an SDF network and color network with specific sample counts and positional encoding frequencies, including eikonal and mask losses [S20].

## Operating characteristics
- **Time Complexity**: Training time scales with the number of iterations, batch size, and network depth. Standard NeRF training on synthetic data (e.g., Blender) takes approximately 12–15 hours on a single GPU for 200k iterations [S7, unofficial].
- **Memory Complexity**: Memory usage is dominated by the MLP weights and the storage of intermediate gradients. Chunking allows processing large batches by splitting them into smaller subsets (e.g., 4096 points) [S9, unofficial].
- **Performance**: On the Blender dataset with 256 hidden units, mip-NeRF achieves PSNR 33.09, SSIM 0.961, and LPIPS 0.043 in 2.89 hours [S1].
- **Strengths**: High-quality novel view synthesis with continuous representation.
- **Limitations**: Slow training times, high memory consumption for large scenes, and susceptibility to artifacts like floaters or view-dependent flickering without specific hyperparameter tuning [S8, unofficial].

## Background
NeRF (Neural Radiance Fields) was introduced by Mildenhall et al. in 2020 [S1][S7, unofficial]. It revolutionized novel view synthesis by using a continuous volumetric scene representation parameterized by an MLP. The standard training configuration described here is based on the original NeRF implementation and subsequent refinements like mip-NeRF [S1] and FastNeRF [S8, unofficial]. The hyperparameters detailed in this article reflect the consensus from these foundational works and their direct descendants.

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] StegaNeRF: Embedding Invisible Information within Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.00047, fetched 2026-10-26
[S3] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S4] DP-NeRF: Deblurred Neural Radiance Field with Physical Scene Priors, https://arxiv.org/abs/2211.12046v4, fetched 2026-10-26
[S5] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
[S6] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
[S7] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S8] FastNeRF by Garbin et al., https://arxiv.org/abs/2103.10380, fetched 2026-10-26 (unofficial)
[S9] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S10] RoGUENeRF: A Robust Geometry-Consistent Universal Enhancer for NeRF, https://doi.org/10.1007/978-3-031-73254-6_4, fetched 2026-10-29
[S11] google-research/multinerf, https://github.com/google-research/multinerf, fetched 2026-10-26 (unofficial)
[S12] High fidelity three dimensional reconstruction and interactive visualization of public art sculptures using neural radiance fields, https://link.springer.com/article/10.1007/s44163-026-02441-3, fetched 2026-10-26
[S13] MZEN: Multi-zoom Enhanced NeRF for 3-D Reconstruction with Unknown Camera Poses, https://link.springer.com/article/10.1007/s10994-026-07114-x, fetched 2026-10-26
[S14] LU-NeRF: Scene and Pose Estimation by Synchronizing Local Unposed NeRFs, https://doi.org/10.1109/iccv51070.2023.01679, fetched 2026-10-26
[S15] DITTO-NeRF: Diffusion-based Iterative Text To Omni-directional 3D Model, https://arxiv.org/abs/2304.02827v1, fetched 2026-10-26 (unofficial)
[S16] CLNeRF: Continual Learning Meets NeRF, https://doi.org/10.1109/iccv51070.2023.02119, fetched 2026-10-26
[S17] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
[S18] Harnessing Low-Frequency Neural Fields for Few-Shot View Synthesis, https://doi.org/10.48550/arxiv.2303.08370, fetched 2026-10-29 (unofficial)
[S19] Nerflets: Local Radiance Fields for Efficient Structure-Aware 3D Scene Representation from 2D Supervision, https://doi.org/10.1109/cvpr52729.2023.00800, fetched 2026-10-26
[S20] OmniSDF: Scene Reconstruction Using Omnidirectional Signed Distance Functions and Adaptive Binoctrees, https://doi.org/10.1109/cvpr52733.2024.01912, fetched 2026-10-26
