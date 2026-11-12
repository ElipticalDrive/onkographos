# NeRF Hyperparameters

Also called: NeRF Training Hyperparameters

## Overview
NeRF hyperparameters define the configuration of the optimization loop, sampling strategies, and loss functions required to train a Neural Radiance Field (NeRF) from multi-view images. They address the problem of converging an implicit volumetric representation to match observed photometric data while managing the computational cost of ray sampling and the stability of gradient descent. What sets this configuration apart from generic volume rendering is the specific interplay between coarse and fine sampling networks, the use of positional encoding frequencies, and the exponential learning rate decay schedules tailored to the high-frequency nature of radiance fields [S1][S3, S6, unofficial].

## How it works
The training process optimizes the parameters of the MLPs (and potentially auxiliary structures like hash tables or caches) by minimizing the difference between rendered pixel colors and ground truth images.

### Inputs and Outputs
*   **Inputs:** A dataset of images with known camera poses (extrinsics) and intrinsics (focal length, principal point). Pixel values are typically normalized to $[0, 1]$ by dividing by 255.0 [S7, unofficial]. Camera centers are often normalized to the range $[-1, 1]$ to ensure reliable training of positional encodings [S11].
*   **Outputs:** Optimized MLP weights (density $\sigma$ and color $\mathbf{c}$), and optionally a learned proposal distribution or cache.

### Data Structures
*   **Ray Batch:** A tensor of shape $[B, 5]$ or $[B, 6]$, where $B$ is the batch size (number of rays). Each row contains ray origin $(x, y, z)$, ray direction $(d_x, d_y, d_z)$, and optionally near/far bounds or viewing direction for view-dependent effects.
*   **Sample Points:** For each ray, $N$ query points are sampled along the ray segment $[t_{near}, t_{far}]$.
*   **MLP State:** Parameters for the position MLP (density/feature) and view-direction MLP (color).

### Numbered Steps
1.  **Data Loading:** Load images and camera transforms. Reserve one image for validation/testing [S7, unofficial]. Normalize pixel values [S7, unofficial].
2.  **Ray Sampling:** For each training iteration, sample $B$ random rays from the training images.
3.  **Point Sampling:**
    *   **Coarse Stage:** Sample $N_c$ points uniformly or stratified along the ray.
    *   **Fine Stage:** Use the density estimates from the coarse stage to sample $N_f$ additional points using importance sampling (non-uniform distribution favoring high density) [S1][S6, unofficial].
4.  **Feature Encoding:** Apply positional encoding (PE) to the 3D coordinates and viewing directions.
    *   Position PE frequencies: $L_p$ (default 10) [S5, S6, unofficial].
    *   Direction PE frequencies: $L_v$ (default 4, or 1 for mitigating artifacts) [S5, S6, unofficial].
5.  **Network Forward Pass:**
    *   Query the coarse MLP with encoded positions to get volume density $\sigma$ and feature vectors.
    *   Query the fine MLP with encoded positions and features to get refined density and color $\mathbf{c}$ (concatenated with encoded viewing directions) [S1][S6, unofficial].
6.  **Volume Rendering:** Compute the rendered color $\hat{C}$ for each ray using the standard differentiable volume rendering integral:
    $$ \hat{C} = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i $$
    where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, and $\delta_i$ is the distance between samples [S1][S6, unofficial].
7.  **Loss Computation:** Calculate the photometric loss (MSE or Charbonnier) between $\hat{C}$ and ground truth $C$. Add regularization terms if applicable (e.g., distortion, normal consistency) [S1, S8, S9].
8.  **Optimization Step:** Compute gradients and update model parameters using the Adam optimizer [S1][S3, S6, unofficial].
9.  **Learning Rate Update:** Adjust the learning rate according to the schedule (exponential decay with optional warmup/delay) [S6, S12, unofficial].
10. **Iteration:** Repeat until convergence (e.g., 200k–300k iterations) [S3, S6, unofficial].

### Formulas
*   **Volume Rendering:**
    $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i, \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right), \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
*   **Asymmetric Predicted Normal Loss (if used):**
    $$ \mathcal{L}_{\text{pred}} = \lambda_n \mathcal{L}_p(w, \mathbf{n}, \text{sg}(\tilde{\mathbf{n}})) + \lambda_{\tilde{n}} \mathcal{L}_p(\text{sg}(w), \text{sg}(\mathbf{n}), \tilde{\mathbf{n}}) $$
    where $\mathcal{L}_p(w, \mathbf{n}, \tilde{\mathbf{n}}) = \sum_{i=1}^N w^{(i)} \|\mathbf{n}^{(i)} - \tilde{\mathbf{n}}^{(i)}\|^2$ [S9].
*   **Simple-RF Loss:**
    $$ L = \lambda_m L_m + \lambda_a L_a + \lambda_{sd} L_{sd} + \lambda_{aug} L_{aug} + \lambda_{cfc} L_{cfc} + \lambda_{mc} L_{mc} $$
    Applied in stages: $L_m, L_a, L_{sd}$ for first 10k iterations; all terms thereafter [S8].
*   **Exponential Learning Rate Decay with Delay:**
    $$ t = \text{clip}(\text{step} / \text{max\_steps}, 0, 1) $$
    $$ \text{log\_lerp} = \exp(\log(\text{lr\_init}) \cdot (1 - t) + \log(\text{lr\_final}) \cdot t) $$
    $$ \text{delay\_rate} = \text{lr\_delay\_mult} + (1 - \text{lr\_delay\_mult}) \cdot \sin(0.5 \cdot \pi \cdot \text{clip}(\text{step} / \text{lr\_delay\_steps}, 0, 1)) $$
    $$ \text{lr} = \text{delay\_rate} \cdot \text{log\_lerp} $$
    [S12, unofficial].

### Parameters with Typical Values
*   **Optimizer:** Adam with $\beta_1=0.9, \beta_2=0.999, \epsilon=10^{-7}$ [S3, S6, unofficial].
*   **Learning Rate:** Initial $5 \times 10^{-4}$, decaying to $5 \times 10^{-5}$ or $8 \times 10^{-5}$ [S2][S3, S6, unofficial].
*   **Batch Size:** 1024 to 3072 rays per step [S2][S3, unofficial].
*   **Samples per Ray:**
    *   Coarse: 64 [S3, S6, unofficial].
    *   Fine: 64–128 [S2][S3, S6, unofficial].
    *   Proposal MLP: 8 samples (for training proposal) [S1].
    *   NeRF MLP: 32 samples (in simplified models) [S1].
*   **MLP Architecture:**
    *   Depth: 8 layers [S1][S5, unofficial].
    *   Width: 256 or 512 hidden units [S1][S5, unofficial].
    *   Positional Encoding Frequencies: 10 for position, 4 for direction [S5, S6, unofficial].
*   **Training Iterations:** 200,000 to 300,000 [S3, S6, unofficial].
*   **Clipping Planes:** Near: 2.0, Far: 6.0 (scene-dependent) [S7, unofficial].

### Edge Cases and Pitfalls
*   **Overfitting View-Dependent Artifacts:** High-frequency Fourier encoding for directions ($L=4$) can cause flickering in motion. Reducing $L$ to 1 or using a smaller direction-dependent cache ($16^3$ to $32^3$) mitigates this [S6, unofficial].
*   **Normal Convergence:** Standard symmetric predicted normal loss oversmooths geometry. Asymmetric loss with stop-gradient operators is required for accurate normals [S9].
*   **Loss Imbalance:** In sparse input scenarios, regularization terms must be weighted carefully. $L_{cfc}$ and $L_{mc}$ are only applied to specific model variants (Main NeRF vs. Augmented TensoRF) [S8].
*   **Test Rendering:** Perturbation and raw noise must be disabled during evaluation to ensure deterministic results [S13, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
import torch.nn as nn
import torch.optim as optim

def positional_encoding(x, num_frequencies):
    """Standard NeRF positional encoding."""
    x = x.unsqueeze(-1)
    scales = torch.tensor([2**i for i in range(num_frequencies)], dtype=x.dtype, device=x.device)
    shaped_scales = (1, 1, 1, 2 * num_frequencies)
    output = torch.cat([x, x * scales], dim=-1)
    return output.reshape(*x.shape[:-1], -1)

def volume_rendering(ray_origins, ray_directions, near, far, network_fn, network_query_fn, N_samples, perturb=1.0):
    """Render a batch of rays using uniform sampling."""
    t_vals = torch.linspace(0., 1., steps=N_samples, device=ray_origins.device)
    z_vals = near * (1. - t_vals) + far * t_vals
    
    if perturb > 0.:
        mids = .5 * (z_vals[..., 1:] + z_vals[..., :-1])
        upper = torch.cat([mids, z_vals[..., -1:]], -1)
        lower = torch.cat([z_vals[..., :1], mids], -1)
        t_rand = torch.rand(z_vals.shape, device=ray_origins.device)
        z_vals = lower + (upper - lower) * t_rand

    pts = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * z_vals.unsqueeze(2).unsqueeze(3)
    
    # Query network
    raw = network_query_fn(pts, ray_directions, network_fn)
    sigma = raw[..., 0]
    rgb = raw[..., 1:]
    
    # Compute weights
    deltas = z_vals[..., 1:] - z_vals[..., :-1]
    delta_inf = 1e10
    deltas = torch.cat([deltas, delta_inf.unsqueeze(-1)], -1)
    
    alpha = 1. - torch.exp(-sigma * deltas)
    T = torch.cumprod(1. - alpha + 1e-10, dim=-2)
    weights = alpha * T[..., :-1]
    
    rgb_final = torch.sum(weights.unsqueeze(-1) * rgb, dim=-2)
    return rgb_final, weights, z_vals

def train_step(model, optimizer, ray_batch, render_kwargs_train, lr_scheduler):
    """Single training step."""
    ray_origins = ray_batch['origins']
    ray_directions = ray_batch['directions']
    targets = ray_batch['targets']
    near = ray_batch['near']
    far = ray_batch['far']
    
    optimizer.zero_grad()
    
    # Coarse rendering
    rgb_coarse, weights_coarse, z_vals_coarse = volume_rendering(
        ray_origins, ray_directions, near, far, 
        model.network_fn, model.network_query_fn, 
        render_kwargs_train['N_samples'], 
        perturb=render_kwargs_train['perturb']
    )
    
    # Fine rendering (if N_importance > 0)
    if render_kwargs_train['N_importance'] > 0:
        # Importance sampling based on coarse weights
        # ... (implementation of importance sampling omitted for brevity, see [S1][S6, unofficial])
        rgb_fine, _, _ = volume_rendering(
            ray_origins, ray_directions, near, far,
            model.network_fine, model.network_query_fn,
            render_kwargs_train['N_importance'],
            perturb=0.0
        )
        rgb = rgb_fine
    else:
        rgb = rgb_coarse
        
    # Loss calculation
    loss = torch.mean((rgb - targets) ** 2)
    
    # Backward pass
    loss.backward()
    optimizer.step()
    
    # Learning rate update
    lr_scheduler.step()
    
    return loss.item()

def get_learning_rate(step, lr_init, lr_final, max_steps, lr_delay_steps=0, lr_delay_mult=1.0):
    """Exponential decay with delay."""
    t = np.clip(step / max_steps, 0, 1)
    log_lerp = np.exp(np.log(lr_init) * (1 - t) + np.log(lr_final) * t)
    
    if lr_delay_steps > 0:
        delay_rate = lr_delay_mult + (1 - lr_delay_mult) * np.sin(0.5 * np.pi * np.clip(step / lr_delay_steps, 0, 1))
    else:
        delay_rate = 1.0
        
    return delay_rate * log_lerp
```

## Subtopics

### [HashNeRF Parameters](3068-hashnerf-parameters.md)
HashNeRF parameters configure the training loop and model architecture for the HashNeRF method, which builds on [NeRF Hyperparameters](3065-nerf-hyperparameters.md). Unlike standard NeRF which relies on positional encoding or fixed-resolution grids, HashNeRF uses a multi-resolution hashed embedding table indexed by voxel coordinates, requiring specific settings for the hash map size, loss regularization, and dataset-specific loading behaviors.

### [NeRF Initialization Strategies](8674-nerf-initialization-strategies.md)
NeRF Initialization Strategies address the slow convergence and instability inherent in training classical Neural Radiance Fields (NeRFs) from random weight initialization. By leveraging pre-computed weights from related models or geometric priors, these methods significantly reduce the number of training epochs required to achieve high-quality rendering. Unlike standard random initialization, which often leads to local minima or requires extensive hyperparameter tuning, initialization strategies provide a geometrically or structurally informed starting point, trading off initial setup time for faster convergence.

### [Mega-NeRF Training Configuration](8766-mega-nerf-training-configuration.md)
Mega-NeRF is a training configuration for Neural Radiance Fields (NeRF) designed to accelerate training and reduce model query costs for large-scale scenes. It differs from standard NeRF by using a single shared Multi-Layer Perceptron (MLP) for both coarse and fine sampling stages, which reduces model size and saves 25% of model queries per ray by reusing coarse outputs during the fine rendering stage.

### [Alpha Invariance / Inverse Scaling](10127-alpha-invariance-inverse-scaling.md)
Alpha invariance is a property in neural volume rendering where the opacity (alpha value) of a discretized segment remains constant despite changes in the scene's physical scale. It solves the problem that the magnitude of the optimized volume density $\sigma(\mathbf{x})$ is intrinsically tied to the domain of integration (distance along the ray), meaning that scaling the scene requires inversely scaling the density to preserve the visual appearance of local geometry. Unlike Multi-Plane Images (MPIs) which fix discretization in advance, NeRF's continuous density parameterization allows flexible resampling but requires this specific handling to maintain robust view synthesis across arbitrary scene scales.

### [Batch size scaling for computational parity](10942-batch-size-scaling-for-computational-parity.md)

### [Model Architecture Modifications](11313-model-architecture-modifications.md)
Model Architecture Modifications refer to structural changes in the neural network components of Neural Radiance Fields (NeRF) to address specific limitations such as overfitting in sparse views, lack of semantic understanding, dynamic scene representation, or computational efficiency. Unlike standard NeRF which maps 3D coordinates and view directions to density and color via a single Multi-Layer Perceptron (MLP), these modifications introduce auxiliary branches, conditional latent codes, or alternative geometric representations (e.g., SDFs, Voxels) to decouple or enhance specific scene properties. They are chosen when the standard volume rendering pipeline fails to capture complex phenomena like depth-of-field blur, semantic segmentation, or topological changes.

### [Regularization Techniques](11314-regularization-techniques.md)
Regularization techniques in NeRF training stabilize optimization, suppress artifacts (such as floaters and blur), and improve generalization in few-shot or sparse input scenarios. Unlike [HashNeRF Parameters](3068-hashnerf-parameters.md) which focus on architectural capacity, or [NeRF Initialization Strategies](8674-nerf-initialization-strategies.md) which set initial states, regularization modifies the loss function or gradient updates during training to enforce physical priors or consistency.

### [Depth-Guided NeRF Training Configuration](11916-depth-guided-nerf-training-configuration.md)
A training configuration for Neural Radiance Fields (NeRF) that incorporates depth priors to guide optimization, addressing the ambiguity of monocular depth estimation and improving geometry reconstruction. It distinguishes itself from standard NeRF training by integrating a depth loss term with specific uncertainty weighting and architectural regularization (dropout) to prevent overfitting on noisy depth inputs.

### [CLIPNeRF Training Configuration](12073-clipnerf-training-configuration.md)
CLIPNeRF training configuration defines the optimization procedure for text-guided NeRF stylization, specifically addressing the challenge of enforcing semantic style transfer while maintaining geometric consistency. It distinguishes itself from standard NeRF training by freezing density-related parameters and minimizing the negative cosine similarity between rendered images and text prompts in the CLIP embedding space. Unlike global contrastive losses that may cause uneven stylization, this configuration often employs directional losses to guide the trajectory of stylization.

### [Adaptive Time-step Training Framework](13342-adaptive-time-step-training-framework.md)
Adaptive Time-step Training (PATA) is a training strategy for spike-based Neural Radiance Fields (NeRF) that reduces inference time steps by learning a scene-wise target time step $t^*$ while maintaining rendering quality through distillation and loss functions. Unlike sample-level or layer-level dynamic inference strategies which cause irregular execution, PATA uses a scene-wise temporal budget to preserve the parallel rendering pattern required by dense ray sampling. It converts time-step selection from a manually tuned hyperparameter into a learned scene-wise budget, reducing temporal computation while preserving rendering behavior.

### [BAD-NeRF](13523-bad-nerf.md)
BAD-NeRF is a method for 3D reconstruction and novel view synthesis from motion-blurred images that jointly optimizes the Neural Radiance Field (NeRF) parameters and the camera motion trajectories during the shutter exposure interval. Unlike standard NeRF which assumes sharp images with infinitesimal exposure times, BAD-NeRF models the physical image formation process by rendering a sequence of sharp intermediate views along a continuous SE(3) trajectory and averaging them to synthesize the observed blur. It distinguishes itself from methods like DeblurNeRF, which rely on fixed camera poses (often inaccurate due to blur), by estimating the start and end poses of each image's exposure jointly with the scene representation, making it robust to severe motion blur and inaccurate initial pose estimates.

## Operating characteristics
*   **Time Complexity:** Training time scales linearly with the number of iterations and the number of samples per ray. Quadrupling the sampling resolution (e.g., from 32x32 to 64x64) quadruples the training time [S4].
*   **Memory Complexity:** Memory usage is dominated by the MLP weights and the batch of rays being processed. Chunking (e.g., 4096 points per minibatch) is used to avoid out-of-memory errors during the forward pass [S7, unofficial].
*   **Measured Performance:** On the Blender dataset with 256 hidden units, training takes ~1.86–2.89 hours [S1]. On an RTX 3090, 200k–300k iterations take 17–25 hours [S3, unofficial].
*   **Strengths:** The hyperparameter configuration allows for high-fidelity view synthesis with relatively simple MLP architectures.
*   **Limitations:** Training is slow compared to explicit representations. Performance is sensitive to the choice of positional encoding frequencies and learning rate decay [S6, unofficial].
*   **When to Use:** Suitable for high-quality novel view synthesis from dense input views.
*   **When Not to Use:** Unsuitable for real-time applications or extremely large-scale scenes without modifications (e.g., Mega-NeRF) [S11].

## Background
The NeRF hyperparameter configuration originates from the seminal "Neural Radiance Fields" paper by Mildenhall et al. (2020) [S1][S6, unofficial]. The standard setup involves an 8-layer MLP with 256 hidden units, positional encoding, and Adam optimization with exponential decay. Subsequent works like mip-NeRF [S1] and FastNeRF [S6, unofficial] refined these parameters to address anti-aliasing and speed, respectively. The configuration has become a baseline for numerous extensions, including regularization techniques [S8, S9] and large-scale training [S11].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] DP-NeRF: Deblurred Neural Radiance Field with Physical Scene Priors, https://arxiv.org/abs/2211.12046v4, fetched 2026-10-26
[S3] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
[S4] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
[S5] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S6] FastNeRF by Garbin et al., https://arxiv.org/abs/2103.10380, fetched 2026-10-26 (unofficial)
[S7] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S8] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
[S9] NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections, https://arxiv.org/abs/2405.14871v1, fetched 2026-10-26
[S10] RoGUENeRF: A Robust Geometry-Consistent Universal Enhancer for NeRF, https://doi.org/10.1007/978-3-031-73254-6_4, fetched 2026-10-29
[S11] Leveraging Neural Radiance Fields for Large-Scale 3D Reconstruction from Aerial Imagery, https://doi.org/10.3390/rs16244655, fetched 2026-10-26
[S12] minghanqin/LangSplat, https://github.com/minghanqin/LangSplat, fetched 2026-10-27 (unofficial)
[S13] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
