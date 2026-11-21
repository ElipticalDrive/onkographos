# NeRF Hyperparameters

Also called: Hyperparameters, NeRF Training Hyperparameters, Training Hyperparameters, ViCA-NeRF Training

## Overview

NeRF hyperparameters define the configuration of the optimization loop, sampling strategy, and loss functions required to train a Neural Radiance Field (NeRF) model. This method specifies the numerical values and procedural steps for initializing the model, sampling rays, computing losses, and updating weights via gradient descent. It sets itself apart from sibling methods by providing the baseline training configuration for the original NeRF architecture [S1][S9, unofficial], while distinct variants (e.g., Mip-NeRF, FastNeRF) adjust these parameters to address specific artifacts or performance constraints [S1][S8, unofficial].

## How it works

**Inputs and Outputs**
*   **Inputs:** A dataset of posed images $\{I_i\}$ with corresponding camera poses $P_i$ (extrinsics and intrinsics), typically normalized such that pixel values are in $[0, 1]$ [S9, unofficial].
*   **Outputs:** A trained NeRF model (MLP weights) and optionally a cached density field or mesh for acceleration [S8, unofficial].

**Data Structures**
*   **Ray Bundle:** A tensor of shape $(B, 3)$ for origins and $(B, 3)$ for directions, where $B$ is the batch size in rays [S9, unofficial].
*   **Query Points:** A tensor of shape $(B, N, 3)$ containing 3D coordinates sampled along the rays, where $N$ is the number of samples per ray [S9, unofficial].
*   **Model Parameters:** Two Multi-Layer Perceptrons (MLPs):
    *   Position MLP: Maps 3D coordinates to density $\sigma$ and intermediate features.
    *   Direction MLP: Maps features and viewing directions to color $\mathbf{c}$ [S1][S9, unofficial].

**Numbered Steps**

1.  **Data Loading and Preprocessing:**
    *   Load the dataset using a loader (e.g., `NeRFDatasetLoader`) [S9, unofficial].
    *   Normalize image pixel values by dividing by 255.0 [S9, unofficial].
    *   Reserve one image for validation/testing (e.g., index 99 or 101) and slice the remaining images for training [S9, unofficial].
    *   Compute ray origins and directions from camera poses and intrinsics [S9, unofficial].

2.  **Ray Sampling:**
    *   For each training iteration, sample $B$ random rays [S8, unofficial].
    *   Sample $N_c$ points uniformly along each ray for the coarse network [S1][S8, unofficial].
    *   Sample $N_f$ points using importance sampling guided by the coarse network's density for the fine network [S1][S8, unofficial].
    *   Typical values: $N_c = 64$, $N_f = 128$ for NeRF 360 Synthetic; $N_c = 64$, $N_f = 64$ for LLFF [S8, unofficial].

3.  **Forward Pass (Volume Rendering):**
    *   Query the coarse MLP for density $\sigma_c$ and color $\mathbf{c}_c$ at the $N_c$ points [S8, unofficial].
    *   Compute transmittance $T_i$ and opacity $\alpha_i$ for coarse samples:
        $$ \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
        $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
    *   Render coarse color $\hat{C}_c(\mathbf{r})$:
        $$ \hat{C}_c(\mathbf{r}) = \sum_{i=1}^{N_c} T_i \alpha_i \mathbf{c}_i $$
    *   Use $\sigma_c$ to sample $N_f$ points for the fine network (importance sampling) [S1][S8, unofficial].
    *   Query the fine MLP for $\sigma_f$ and $\mathbf{c}_f$ at the $N_f$ points.
    *   Render fine color $\hat{C}_f(\mathbf{r})$ using the same volume rendering integral [S1].

4.  **Loss Computation:**
    *   Compute the photometric loss between the rendered fine color $\hat{C}_f$ and the ground truth image $C_{gt}$ [S1].
    *   Standard NeRF uses Mean Squared Error (MSE) [S1].
    *   Mip-NeRF 360 uses Charbonnier loss [S1].
    *   Total Loss $L = L_{photometric}$ [S1].

5.  **Optimization Update:**
    *   Initialize the Adam optimizer with the model parameters [S9, unofficial].
    *   Compute gradients of $L$ with respect to the MLP weights [S8, unofficial].
    *   Update weights using the Adam optimizer [S8, unofficial].
    *   Apply learning rate decay [S8, unofficial].

**Formulas and Parameters**

*   **Adam Optimizer:**
    *   $\beta_1 = 0.9, \beta_2 = 0.999, \epsilon = 10^{-7}$ [S5, unofficial][S6][S8, unofficial].
*   **Learning Rate:**
    *   Initial learning rate $\eta_0 = 5 \times 10^{-4}$ [S8, S9, unofficial].
    *   Decays exponentially to $5 \times 10^{-5}$ (or $8 \times 10^{-5}$ in some implementations) [S4][S5, S8, unofficial].
*   **Batch Size:**
    *   $B = 1024$ rays [S4][S7, unofficial].
    *   Alternative: $B = 3072$ rays to fully utilize GPU memory [S5, unofficial].
    *   Alternative: $B = 1000$ rays [S8, unofficial].
*   **Training Iterations:**
    *   Standard NeRF: 300,000 iterations [S8, unofficial].
    *   Mip-NeRF 360: 200,000 iterations [S4][S7, unofficial].
    *   ViCA-NeRF: 40,000 iterations (30k pre-train + 10k edit) [S3].
*   **Chunking:**
    *   Process points in chunks of size 4096 to avoid out-of-memory errors [S9, unofficial].
*   **Clipping Planes:**
    *   Near threshold: 2.0 [S9, unofficial].
    *   Far threshold: 6.0 [S9, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def train_nerf(dataset, model, num_iterations=300000, batch_size=1024, 
               lr=5e-4, n_samples_coarse=64, n_samples_fine=128):
    """
    Train a NeRF model.
    
    Args:
        dataset: Iterable yielding (images, poses, intrinsics) batches.
        model: NeRF model (Coarse and Fine MLPs).
        num_iterations: Total training steps.
        batch_size: Number of rays per batch.
        lr: Initial learning rate.
        n_samples_coarse: Samples per ray for coarse network.
        n_samples_fine: Samples per ray for fine network.
    """
    optimizer = optim.Adam(model.parameters(), lr=lr, betas=(0.9, 0.999), eps=1e-7)
    
    # Learning rate scheduler (exponential decay)
    # Note: Exact decay formula varies; here we assume a simple exponential decay
    # or step decay as per common implementations [S4][S8, unofficial].
    scheduler = optim.lr_scheduler.ExponentialLR(optimizer, gamma=0.999) 
    
    model.train()
    
    for i in range(num_iterations):
        # 1. Get batch of rays
        # Assume dataset yields normalized images [0,1] and poses
        images, poses, intrinsics = next(dataset)
        
        # Flatten images to [H*W, 3]
        H, W, _ = images.shape
        images_flat = images.reshape(-1, 3)
        
        # Sample random rays
        ray_indices = torch.randint(0, H * W, (batch_size,))
        rays = get_rays_from_indices(ray_indices, poses, intrinsics) # Helper to generate origins/directions
        
        origins, directions = rays
        
        # 2. Sample points
        # Uniform sampling for coarse
        depths_coarse = torch.linspace(0, 1, n_samples_coarse).to(origins.device)
        query_points_coarse = origins.unsqueeze(1) + directions.unsqueeze(1) * depths_coarse.unsqueeze(0).unsqueeze(-1)
        
        # Query coarse network
        # Chunking to avoid OOM
        sigma_coarse, rgb_coarse = query_model_chunked(model.coarse_mlp, query_points_coarse, chunksize=4096)
        
        # Compute transmittance and opacity for coarse
        # delta is distance between samples (assumed uniform for simplicity here)
        delta = 1.0 / n_samples_coarse 
        alpha_coarse = 1.0 - torch.exp(-sigma_coarse * delta)
        T_coarse = torch.cumprod(torch.cat([torch.ones_like(alpha_coarse[:, :1]), 1.0 - alpha_coarse], dim=1)[:, :-1], dim=1)
        
        # Render coarse color
        weights_coarse = T_coarse * alpha_coarse
        rgb_rendered_coarse = torch.sum(weights_coarse.unsqueeze(-1) * rgb_coarse, dim=1)
        
        # Importance sampling for fine network based on coarse density
        # (Simplified: using coarse alpha as importance weights)
        weights_for_sampling = weights_coarse.detach()
        # Sample indices for fine points based on weights
        # In practice, this involves stratified sampling or inverse CDF sampling
        # Here we assume a helper function that returns fine query points
        query_points_fine = importance_sample(origins, directions, weights_for_sampling, n_samples_fine)
        
        # Query fine network
        sigma_fine, rgb_fine = query_model_chunked(model.fine_mlp, query_points_fine, chunksize=4096)
        
        # Render fine color
        alpha_fine = 1.0 - torch.exp(-sigma_fine * delta)
        T_fine = torch.cumprod(torch.cat([torch.ones_like(alpha_fine[:, :1]), 1.0 - alpha_fine], dim=1)[:, :-1], dim=1)
        weights_fine = T_fine * alpha_fine
        rgb_rendered_fine = torch.sum(weights_fine.unsqueeze(-1) * rgb_fine, dim=1)
        
        # 3. Compute Loss
        # Get corresponding ground truth colors
        gt_colors = images_flat[ray_indices]
        
        # MSE Loss
        loss = torch.mean((rgb_rendered_fine - gt_colors) ** 2)
        
        # 4. Backpropagate
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # 5. Update LR
        scheduler.step()
        
        if i % 100 == 0:
            print(f"Iter {i}, Loss: {loss.item():.4f}")

def query_model_chunked(mlp, points, chunksize=4096):
    """
    Query MLP in chunks to manage memory.
    """
    N = points.shape[0]
    all_sigma = []
    all_rgb = []
    
    for start in range(0, N, chunksize):
        end = min(start + chunksize, N)
        chunk_points = points[start:end]
        
        # Forward pass
        sigma, rgb = mlp(chunk_points)
        
        all_sigma.append(sigma)
        all_rgb.append(rgb)
        
    return torch.cat(all_sigma, dim=0), torch.cat(all_rgb, dim=0)

def get_rays_from_indices(indices, poses, intrinsics):
    """
    Helper to generate ray origins and directions from indices.
    """
    # This is a placeholder for the actual ray generation logic
    # which depends on the specific camera model (e.g., pinhole)
    # and pose format (e.g., world-to-camera or camera-to-world).
    # [G] Standard pinhole camera model:
    # u, v = indices % W, indices // W
    # x = (u - cx) / fx
    # y = (v - cy) / fy
    # z = 1
    # ray_dir_cam = [x, y, z]
    # ray_dir_world = pose[:3, :3] @ ray_dir_cam
    # ray_orig = pose[:3, 3]
    # return ray_orig, ray_dir
    pass

def importance_sample(origins, directions, weights, n_samples):
    """
    Sample points based on importance weights.
    [G] This function implements stratified sampling or inverse CDF sampling
    to select points along the ray where density is high.
    """
    pass
```

## Subtopics

### [NeRF Initialization Strategies](8674-nerf-initialization-strategies.md)
NeRF Initialization Strategies address the slow convergence and instability inherent in training classical Neural Radiance Fields (NeRFs) from random weight initialization. By leveraging pre-computed weights from related models or geometric priors, these methods significantly reduce the number of training epochs required to achieve high-quality rendering. Unlike standard random initialization, which often leads to local minima or requires extensive hyperparameter tuning, initialization strategies provide a geometrically or structurally informed starting point, trading off initial setup time for faster convergence.

### [Mega-NeRF Training Configuration](8766-mega-nerf-training-configuration.md)
Mega-NeRF is a training configuration for Neural Radiance Fields (NeRF) designed to accelerate training and reduce model query costs for large-scale scenes. It differs from standard NeRF by using a single shared Multi-Layer Perceptron (MLP) for both coarse and fine sampling stages, which reduces model size and saves 25% of model queries per ray by reusing coarse outputs during the fine rendering stage.

### [Alpha Invariance / Inverse Scaling](10127-alpha-invariance-inverse-scaling.md)
Alpha invariance is a property in neural volume rendering where the opacity (alpha value) of a discretized segment remains constant despite changes in the scene's physical scale. It solves the problem that the magnitude of the optimized volume density $\sigma(\mathbf{x})$ is intrinsically tied to the domain of integration (distance along the ray), meaning that scaling the scene requires inversely scaling the density to preserve the visual appearance of local geometry. Unlike Multi-Plane Images (MPIs) which fix discretization in advance, NeRF's continuous density parameterization allows flexible resampling but requires this specific handling to maintain robust view synthesis across arbitrary scene scales.

### [Batch size scaling for computational parity](10942-batch-size-scaling-for-computational-parity.md)
Not given in the sources.

### [Model Architecture Modifications](11313-model-architecture-modifications.md)
Model Architecture Modifications alter the standard Neural Radiance Field (NeRF) baseline [Parent](3065-nerf-hyperparameters.md) to address specific limitations such as inference speed, scene complexity, semantic understanding, or dynamic content. While the standard architecture uses two Multi-Layer Perceptrons (MLPs) with positional encoding, these modifications generally fall into three categories: input expansion (adding conditioning vectors), output expansion (adding auxiliary heads for semantic/object codes), and representation substitution (replacing MLPs with discrete structures or SDFs) [14428-sparse-view-and-few-shot-optimization.md]. These architectures aim to improve rendering fidelity, enable novel tasks like object manipulation, or reduce computational costs on constrained hardware.

### [Regularization Techniques](11314-regularization-techniques.md)
Regularization techniques in NeRF training are strategies applied to the optimization objective or the model parameters to mitigate artifacts such as floaters, blur, and overfitting, particularly in sparse-view or ill-posed reconstruction scenarios. They set themselves apart from general hyperparameters (like learning rate or batch size) by directly modifying the geometry, density distribution, or view-dependency of the radiance field to enforce physical plausibility or structural consistency.

### [NeRF Pre-processing](12110-nerf-pre-processing.md)
NeRF pre-processing stabilizes training by constraining the dynamic range of image features and normalizing camera geometry before optimization. It addresses feature fluctuations caused by varying exposure, illumination, and occlusions, which otherwise lead to unstable MLP training. Unlike sibling methods that modify architecture or loss functions, this approach treats normalization as a conventional data preparation step to ensure inputs fall within the effective range of positional encodings.

### [Exponential Learning Rate Decay with Delay](12969-exponential-learning-rate-decay-with-delay.md)
Exponential Learning Rate Decay with Delay is a learning rate schedule function used in NeRF optimization that interpolates between an initial and final learning rate using logarithmic linear interpolation, while optionally applying a warm-up phase via reverse cosine decay. It distinguishes itself from standard exponential decay by providing a controlled initial delay period where the learning rate is modulated by a separate cosine schedule, allowing for stable initialization before the main decay trajectory begins.

### [Adaptive Time-step Training Framework](13342-adaptive-time-step-training-framework.md)
Adaptive Time-step Training (PATA) is a training strategy for spike-based Neural Radiance Fields (NeRF) that reduces inference time steps by learning a scene-wise target time step $t^*$ while maintaining rendering quality through distillation and loss functions. Unlike sample-level or layer-level dynamic inference strategies which cause irregular execution, PATA uses a scene-wise temporal budget to preserve the parallel rendering pattern required by dense ray sampling. It converts time-step selection from a manually tuned hyperparameter into a learned scene-wise budget, reducing temporal computation while preserving rendering behavior.

### [Model Architecture and Encoding](13550-model-architecture-and-encoding.md)
This family defines the neural network structures and coordinate encoding schemes used to map 3D spatial coordinates and viewing directions to radiance field attributes (density, color, semantics). It addresses the challenge of representing high-frequency scene details and complex view-dependent effects within differentiable volume rendering pipelines. It differs from sibling methods like [NeRF Initialization Strategies](8674-nerf-initialization-strategies.md) by focusing on the architectural topology and feature mapping rather than weight initialization, and from [Model Architecture Modifications](11313-model-architecture-modifications.md) by encompassing both the base encoding functions and the MLP/grid structures that consume them.

### [Sampling and Rendering Pipeline Configuration](15626-sampling-and-rendering-pipeline-configuration.md)
This configuration defines the parameters and logic for generating rays and querying the neural radiance field during training and inference. It solves the problem of balancing rendering fidelity with computational cost by controlling sample density, noise injection, and multi-resolution strategies. It differs from sibling configurations by focusing specifically on the ray-sampling mechanics and the structure of the rendering kwargs dictionary, rather than initialization weights, architecture, or regularization terms.

### [Method-Specific Training Pipelines](15627-method-specific-training-pipelines.md)
Not given in the sources.

### [Iterative 3D Super-Resolution](16434-iterative-3d-super-resolution.md)
Iterative 3D Super-Resolution (I3DS) is a two-stage alternating training framework designed to super-resolve Neural Radiance Fields (NeRFs) by disentangling 2D image upscaling from 3D NeRF synchronization. Unlike direct score distillation methods that produce blurred details due to local ray supervision, I3DS uses a diffusion-based upscaler to generate high-resolution, low-resolution-consistent images in Stage 1, then synchronizes these images into the 3D representation in Stage 2 to enforce multi-view consistency. It is distinguished from NeRF editing approaches by optimizing both stages concurrently and replacing all training images in a single batch, achieving a 4× reduction in optimization duration.

### [Random Patch Sampling](16594-random-patch-sampling.md)
Random patch sampling is a training strategy for Neural Radiance Field (NeRF) models, specifically those employing Super-Resolution (SR) modules or handling defocus blur, where image patches are extracted from random positions in the ray space rather than a fixed grid. It solves the problem of rigid boundary artifacts in grid-based sampling, which prevents convolutional kernels from seeing variations between grid lines, and reduces training time by allowing neighboring pixels to share ray computations in blur modeling. Unlike grid-based patch sampling, it avoids cutting ray space strictly, ensuring the model sees diverse patterns without bias.

## Operating characteristics

**Time and Memory Complexity**
*   **Time:** Training typically takes 17–25 hours on a single NVIDIA RTX 3090 GPU for 200–300k iterations [S5, unofficial].
*   **Scaling:** Training time quadruples when increasing sampling resolution from 32x32 to 64x64 samples [S6].
*   **Memory:** Memory usage is dominated by the batch size and the number of samples per ray. Chunking (e.g., 4096 points) is required to avoid out-of-memory errors during the forward pass [S9, unofficial].

**Measured Performance**
*   **Blender Dataset (256 hidden units):**
    *   Mip-NeRF: PSNR 33.09, SSIM 0.961, LPIPS 0.043, Time 2.89 hrs [S1].
    *   Our Model (Proposal-based): PSNR 32.96, SSIM 0.960, LPIPS 0.043, Time 1.86 hrs [S1].
*   **Blender Dataset (512 hidden units):**
    *   Mip-NeRF: PSNR 33.03, SSIM 0.964, LPIPS 0.037, Time 7.03 hrs [S1].
    *   Our Model (Proposal-based): PSNR 33.25, SSIM 0.962, LPIPS 0.039, Time 3.42 hrs [S1].
*   **Lego Scene (Blender Synthetic):**
    *   After 200k iterations: PSNR ~31-32 dB, SSIM ~0.96, Time ~12-15 hours [S7, unofficial].

**Strengths and Limitations**
*   **Strengths:** High-quality novel view synthesis; robust to moderate occlusions; well-understood optimization landscape with Adam [S1][S8, unofficial].
*   **Limitations:** Slow convergence (requires hundreds of thousands of iterations); sensitive to hyperparameters (learning rate, batch size); prone to view-dependent artifacts (flickering) if not regularized [S8, unofficial]; high computational cost for large scenes [S1].

**When to Use**
*   Use for standard novel view synthesis tasks with dense input views.
*   Use when high fidelity is required and computational resources are available.

**When Not to Use**
*   Do not use for real-time rendering without acceleration structures (e.g., caching, distillation) [S8, unofficial].
*   Do not use for sparse-view scenarios without regularization or specialized architectures [S1, S2].

## Background

**Origin**
NeRF (Neural Radiance Fields) was introduced by Mildenhall et al. in 2020 [S1][S8, unofficial]. The method uses a fully connected MLP to represent a continuous 5D volumetric scene function, mapping 3D coordinates and viewing directions to density and color.

**History and Development**
*   **2020:** Original NeRF published [S1][S8, unofficial].
*   **2021:** Mip-NeRF introduced anti-aliasing and improved sampling [S1].
*   **2021:** FastNeRF introduced caching and direction-dependent caches to accelerate training and rendering [S8, unofficial].
*   **2022:** Mip-NeRF 360 extended the method to unbounded scenes [S1].
*   **2023:** StegaNeRF introduced steganographic capabilities [S2].
*   **2024:** ViCA-NeRF introduced view-consistency-aware editing [S3].

**Usage**
NeRF is widely used in computer vision for novel view synthesis, 3D reconstruction, and scene editing [S1, S3][S8, unofficial].

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
[S13] StyleNeRF: A Style-based 3D-Aware Generator for High-resolution Image Synthesis, https://doi.org/10.48550/arxiv.2110.08985, fetched 2026-10-26 (unofficial)
