# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric error between rendered images and ground-truth training views by updating the parameters of a Neural Radiance Field (NeRF) [S5, S6, S14, unofficial]. It solves the inverse rendering problem of recovering continuous 3D geometry and appearance from discrete 2D observations by treating the scene as a differentiable volumetric function [S1, S8]. It sets itself apart from sibling methods like [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md) by typically employing a unified or coarse-to-fine single-network (or dual-network) optimization with stochastic ray sampling, whereas [Photometric Optimization](5790-photometric-optimization.md) focuses specifically on the loss formulation rather than the loop mechanics, and [Staggered Optimization Scheme](5832-staggered-optimization-scheme.md) decouples parameter updates across time or space [S11, S14, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics $C2W$ and intrinsics $K$) and dimensions $H \times W$ [S5, S11, unofficial]. A neural network $f_\theta$ (typically an MLP) parameterized by weights $\theta$ [S5, S11, unofficial].
*   **Outputs:** Optimized network weights $\theta$ that minimize the reconstruction loss [S5, S14, unofficial].

**Data Structures:**
*   **Rays:** A ray is defined by origin $\mathbf{o}$ and direction $\mathbf{d}$. Training rays are pre-computed or generated on-the-fly. In batching implementations, rays are stored in a flat array `rays_rgb` of shape $[(N-1) \cdot H \cdot W, 3, 3]$ (or $[N_{rays}, 3, 3]$) where axis 1 contains `[origin, direction, target_rgb]` and axis 2 contains the XYZ/RGB components [S5, S14, unofficial]. Data type is typically `float32` [S5, S14, unofficial].
*   **Checkpoint:** A dictionary containing `'model'` (or `'coarse'`, `'fine'`) state dictionaries, `'optimizer'` state dictionary, and `'step'` or `'epoch'` integer [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the NeRF model(s). For coarse-to-fine, initialize both coarse and fine MLPs [S11, unofficial]. Initialize background model if applicable [S15, unofficial].
    *   Initialize the optimizer (typically Adam or AdamW) with a learning rate $lr$ (default $5 \times 10^{-4}$ or $0.01$) [S5, S11, S17, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   If resuming, load checkpoint and set scheduler last epoch to `start_epoch - 1` [S9, unofficial].

2.  **Ray Sampling:**
    *   For each iteration, sample a batch of $N_{rand}$ rays (default 1024 or 2048) [S5, S11, S14, unofficial].
    *   If using pre-computed batching, slice the `rays_rgb` array and shuffle when exhausted [S5, S14, unofficial].
    *   If generating on-the-fly, randomly select pixel coordinates $(x, y)$ from training images, compute ray origins/directions using camera matrices, and gather target RGB colors [S4, S14, unofficial].
    *   Optionally apply a "precrop" grid (centered crop) for the first $I_{precrop}$ iterations to focus on high-frequency details [S2, S14, unofficial].
    *   Sample $K$ points along each ray between near $t_{near}$ and far $t_{far}$ bounds. Bounds are dataset-dependent (e.g., Blender: $t_{near}=2.0, t_{far}=6.0$; LLFF: computed from bounding depths) [S2, S13, unofficial].
    *   Apply stratified sampling with jitter $\epsilon \sim U(0,1)$ to sample points $t_i$ within intervals $[t_{i-1}, t_i]$ to reduce aliasing [S5, S18, unofficial].

3.  **Forward Pass (Rendering):**
    *   Query the neural network $f_\theta(\mathbf{x}, \mathbf{d})$ at each sample point $\mathbf{x}$ with view direction $\mathbf{d}$ to obtain density $\sigma$ and color $\mathbf{c}$ [S5, S6, unofficial].
    *   Compute transmittance $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ and weights $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ [S5, S18, unofficial].
    *   Render color $\hat{\mathbf{C}} = \sum w_i \mathbf{c}_i$ [S5, unofficial].
    *   If using coarse-to-fine training, render coarse output $\hat{\mathbf{C}}_{coarse}$, sample new points based on coarse weights (importance sampling), and render fine output $\hat{\mathbf{C}}_{fine}$ [S5, S11, unofficial].

4.  **Loss Computation:**
    *   Compute Mean Squared Error (MSE) between rendered color $\hat{\mathbf{C}}$ and target color $\mathbf{C}_{gt}$:
        $$ \mathcal{L}_{rgb} = \frac{1}{N_{rays}} \sum_{i=1}^{N_{rays}} \| \hat{\mathbf{C}}_i - \mathbf{C}_{gt,i} \|^2 $$
        [S14, S17, unofficial].
    *   If coarse output is available, add coarse loss: $\mathcal{L}_{total} = \mathcal{L}_{rgb} + \mathcal{L}_{coarse}$ [S6, S11, S18, unofficial].
    *   Optionally add regularization terms (e.g., Eikonal, TV, Sparsity) weighted by $\lambda$ [S7, unofficial][S8][S16, unofficial].

5.  **Backpropagation and Update:**
    *   Zero gradients: `optimizer.zero_grad()` [S18, S17, unofficial].
    *   Compute gradients: `loss.backward()` [S18, S17, unofficial].
    *   **Pitfall:** Higher-order derivatives (e.g., for logging gradients or Laplacians) must be computed *before* `loss.backward()` as the latter frees the autograd graph [S3, unofficial].
    *   Update weights: `optimizer.step()` [S18, S17, unofficial].
    *   Update learning rate according to scheduler [S9, S18, unofficial].

6.  **Logging and Checkpointing:**
    *   Log loss and PSNR ($PSNR = -10 \log_{10}(MSE)$) every $I_{print}$ steps [S14, S2, unofficial].
    *   Render validation images every $I_{render}$ steps [S9, S11, unofficial].
    *   Save checkpoint every $I_{save}$ steps [S9, S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def train_nerf_loop(model, train_dataset, val_dataset, config):
    """
    Main training loop for NeRF.
    
    Args:
        model: Neural Radiance Field model (MLP).
        train_dataset: Dataset yielding (images, poses, intrinsics).
        val_dataset: Validation dataset.
        config: Dictionary containing hyperparameters.
    """
    # 1. Initialization
    optimizer = optim.Adam(model.parameters(), lr=config['lr'])
    scheduler = optim.lr_scheduler.ExponentialLR(optimizer, gamma=config['decay_rate'])
    
    # Pre-compute rays if batching is enabled
    if config['use_batching']:
        rays_rgb = precompute_rays(train_dataset) # Shape: [N_total, 3, 3]
        rays_rgb = rays_rgb.shuffle() # Shuffle initial order
        i_batch = 0
    
    n_iters = config['n_iters'] # e.g., 100,000
    batch_size = config['batch_size'] # e.g., 1024
    
    for step in range(n_iters):
        # 2. Ray Sampling
        if config['use_batching']:
            # Slice pre-computed batch
            batch = rays_rgb[i_batch : i_batch + batch_size]
            i_batch += batch_size
            if i_batch >= len(rays_rgb):
                rays_rgb.shuffle()
                i_batch = 0
            
            # Split into rays (origin, dir) and targets (rgb)
            # batch shape: [B, 3, 3] -> [B, 3] origins, [B, 3] dirs, [B, 3] targets
            rays_o = batch[:, 0, :]
            rays_d = batch[:, 1, :]
            target_rgb = batch[:, 2, :]
        else:
            # On-the-fly sampling
            img_idx = torch.randint(0, len(train_dataset), (1,)).item()
            rays_o, rays_d, target_rgb = sample_rays_from_image(train_dataset, img_idx, batch_size)
            
        # Sample depths along rays
        z_vals = sample_depths(rays_o, rays_d, config['near'], config['far'], config['num_samples'])
        
        # 3. Forward Pass
        # Flatten rays for network input
        pts = rays_o[:, None, :] + rays_d[:, None, :] * z_vals[:, :, None] # [B, K, 3]
        viewdirs = rays_d[:, None, :].expand_as(pts) # [B, K, 3]
        
        # Query NeRF
        raw = model(pts, viewdirs) # [B, K, 4] (RGB + sigma)
        
        # Volume Rendering
        rgb_map = volume_render(raw, z_vals, rays_d) # [B, 3]
        
        # 4. Loss Computation
        loss_fn = nn.MSELoss()
        loss = loss_fn(rgb_map, target_rgb)
        
        # Add coarse loss if applicable (pseudo-code for coarse-to-fine)
        if hasattr(model, 'coarse_model') and step < config['fine_start_step']:
            raw_coarse = model.coarse_model(pts, viewdirs)
            rgb_coarse = volume_render(raw_coarse, z_vals, rays_d)
            loss += loss_fn(rgb_coarse, target_rgb)
            
        # 5. Backpropagation and Update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        scheduler.step()
        
        # 6. Logging
        if step % config['log_interval'] == 0:
            psnr = -10.0 * torch.log10(loss.item())
            print(f"Step {step}: Loss {loss.item():.6f}, PSNR {psnr:.2f}")
            
        if step % config['save_interval'] == 0:
            save_checkpoint(model, optimizer, step, config['checkpoint_path'])
```

**Parameters with Typical Values:**
*   `lr`: $5 \times 10^{-4}$ [S11, unofficial] or $0.01$ [S17, unofficial].
*   `batch_size` ($N_{rand}$): 1024 [S11, unofficial] or 2048 [S5, unofficial].
*   `n_iters`: 100,000 [S11, unofficial] or 1,000,000 [S14, unofficial].
*   `decay_rate`: 0.1 [S6, S18, unofficial].
*   `decay_steps`: `args.lrate_decay * 1000` [S5, S18, unofficial].
*   `num_samples`: 128 [S17, unofficial] or 64/128 for coarse/fine [S5, unofficial].
*   `near`/`far`: Dataset dependent (Blender: 2.0/6.0; LLFF: computed) [S2, S13, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Large batches or high-resolution rays can overflow GPU memory. Mitigate by chunking ray batches during the forward pass (`chunk=args.chunk`) [S5, S14, unofficial].
*   **Instability in Dynamic Growth:** When growing sub-networks (e.g., in Recursive-NeRF), initialize alpha linear weights of new sub-networks to match the parent to avoid density collapse [S1].
*   **Checkpoint Resumption:** When resuming, ensure the learning rate scheduler's last epoch is set to `start_epoch - 1` to maintain correct decay trajectory [S9, unofficial].
*   **Empty Checkpoints:** If a checkpoint is loaded but the step counter is 0, treat as invalid/error [S15, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images from a Neural Radiance Field (NeRF) during the optimization loop. It solves the problem of mapping discrete, noisy neural network outputs (volume density and color) into a coherent, continuous image by sampling points along camera rays, querying the implicit function, and integrating them via alpha compositing. It sets itself apart from its siblings by combining ray generation, adaptive marching with occupancy grid pruning, neural network evaluation, and alpha compositing into a single differentiable forward pass that supports gradient computation for backpropagation.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the static and dynamic hyperparameters, optimization routines, and architectural constraints required to train differentiable volume rendering models. It solves the problem of translating theoretical radiance field formulations into stable, convergent training procedures by specifying exact network structures, sampling strategies, and loss functions. It differs from [Training Loop Setup](1522-training-loop-setup.md) by defining *what* parameters are used (e.g., learning rates, batch sizes, loss weights) rather than *how* the loop executes (tensor layouts, optimizer state interactions). It serves as the foundational configuration for higher-level procedures like [Training Loop](4290-training-loop.md) and [Training Loop with Adaptive Regularization & LR Decay](2408-training-loop-with-adaptive-regularization-lr-decay.md).

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [MVSNeRF](6197-mvsnerf.md)
MVSNeRF is a method for fast, generalizable radiance field reconstruction from multi-view stereo (MVS) images. Unlike standard NeRF which requires per-scene optimization, MVSNeRF uses a feedforward neural network to predict the radiance field directly from input images, significantly reducing training time and iteration counts. It sets itself apart by learning a data prior that accelerates convergence, enabling rapid inference without iterative scene-specific optimization.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
This family of methods addresses the problem of optimizing Neural Radiance Field (NeRF) scene representations and camera parameters (poses and intrinsics) simultaneously, either when initial poses are inaccurate or entirely unknown. Unlike standard NeRF training which assumes ground-truth camera poses, these methods treat camera parameters as trainable variables within the computational graph. They are distinguished from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by their explicit optimization of the camera state $\pi$ alongside the neural weights $\Theta$, often requiring specialized strategies (e.g., frequency scheduling, multi-stage loss switching, or geometric priors) to avoid local minima caused by the non-convexity of the joint optimization landscape.

### [Specific Model Architectures](8585-specific-model-architectures.md)
Specific Model Architectures refers to the family of neural network structures and representations used to encode the radiance field $f_\theta(\mathbf{x}, \mathbf{d})$ within the NeRF training loop. While the standard NeRF uses a simple Multi-Layer Perceptron (MLP) to map 3D coordinates and view directions to density and color, this family encompasses modifications to the network topology, input encoding, and output heads to address specific limitations such as rendering speed, geometric ambiguity, multi-view consistency, and generalization. These architectures differ from their siblings in the training loop (such as [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) or [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)) by fundamentally altering the function $f_\theta$ itself, rather than just the sampling strategy or loss weights.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
This family encompasses the loss functions and regularization terms applied during the optimization of Neural Radiance Fields (NeRFs) to improve view synthesis quality, geometric accuracy, and training stability. While the standard photometric loss minimizes pixel-wise differences between rendered and ground truth images, it is often insufficient for sparse views or misaligned data, leading to artifacts such as "floaters," over-smoothing, or geometric degeneracy. These methods set themselves apart by introducing priors—such as diffusion models, normalizing flows, geometric consistency constraints, or adaptive weighting—that constrain the density and color fields beyond simple pixel reconstruction.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Progressive Inpainting-and-Erasing (PERF) is a training strategy for single-view NeRF that avoids geometry conflicts between visible and invisible regions by computing a mask of conflicted regions and eliminating them from NeRF training. It sets itself apart from standard single-view NeRF methods by integrating a collaborative RGBD inpainting step with a progressive erasing mechanism to handle the ambiguity of unseen geometry, building on the [NeRF Training Loop](970-nerf-training-loop.md) by dynamically expanding the reference view set.

### [KNN-NeRF](8672-knn-nerf.md)
KNN-NeRF is a baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Hard Sample Mining](8673-hard-sample-mining.md)
Hard Sample Mining (HSM) is an optimization procedure for neural radiance fields that improves training efficiency and reconstruction quality by selectively updating the network based on pixel-level error. It solves the problem of uniform computational waste in standard NeRF training, where samples contributing little to the gradient are treated equally to those with high error. Unlike sibling methods such as [Probability-based Ray Sampling with Quadtree Subdivision](7481-probability-based-ray-sampling-with-quadtree-subdivision.md) or [Nerfacc](https://github.com/NVlabs/nerfacc) which prune low-weight samples based on density or transmittance, HSM ranks samples using the propagated pixel loss to subsample a "hard" minibatch that retains most of the update information.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Progressive and Dynamic Scene Growth addresses the challenge of reconstructing large-scale or complex 3D scenes from limited input data by incrementally expanding the neural representation's capacity or the training dataset. Unlike static NeRF architectures that require pre-defining the entire hierarchy or fixed network size, these methods adaptively grow the model structure or add synthetic reference views during training to resolve occlusions and capture high-frequency details. This family distinguishes itself from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) (which focuses on ray selection) and [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md) (which relies on external priors) by actively modifying the model's topology or the set of supervision signals to improve reconstruction quality in complex regions.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Adaptive ray sampling methods optimize the distribution of sample points along viewing rays to concentrate computational resources on geometrically complex or opaque regions, thereby improving reconstruction quality and training efficiency compared to uniform sampling. This family is distinguished from its siblings by its focus on *where* to sample along the ray depth or *which* rays to cast based on learned or heuristic importance, whereas [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) focuses on the integration mechanics and [Training Configuration](2332-training-configuration.md) focuses on hyperparameters.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
This family addresses the degradation of NeRF quality—such as blurred geometry, slow convergence, and floaters—when trained on sparse input views (typically 15–20 images or fewer than 50–100 views). It distinguishes itself from standard NeRF training by decoupling the initialization and regularization processes from the default random or empty-space assumptions, instead leveraging external geometric scaffolds, semantic masks, or visibility priors to anchor the optimization.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Scene Representation and Partitioning addresses the scalability limits of global NeRF models, which suffer from high memory consumption, slow training convergence, and rendering artifacts when applied to large-scale environments or dynamic scenes. By decomposing the 3D spatial domain into disjoint regions (blocks, tiles, or cells), each managed by a dedicated local neural network or data structure, these methods enable distributed training, faster inference, and the handling of scenes spanning thousands of square meters. Unlike global approaches, partitioning allows for independent optimization of local regions, though it introduces challenges regarding boundary continuity and temporal consistency in dynamic settings.

### [Zip-NeRF Training Procedure](9538-zip-nerf-training-procedure.md)
The Zip-NeRF Training Procedure is a two-stage optimization method for inverse rendering that jointly trains a physically-based scene model, environment map, and radiance caches (Zip-NeRF and Fast Cache) to estimate incoming radiance. It solves the bias introduced by approximating incoming radiance using neural caches by employing a control variate strategy and a specific gradient trick for the photometric loss. It sets itself apart from standard NeRF training by decoupling the cache initialization from the joint optimization of the physical model and by using an occlusion-aware importance sampler to reduce variance in the radiance estimation.

### [DPPE (Dense Pose Estimation in a Plenoxels Environment)](9640-dppe-dense-pose-estimation-in-a-plenoxels-environment.md)
DPPE is a 6-DoF monocular RGB-only pose estimation procedure for Plenoxels environments that recovers the ground truth camera pose after a perturbation by minimizing photometric errors in re-rendering. It distinguishes itself from analytical derivative-based methods (e.g., Teigen et al. [30]) by using numerical gradient approximation, making it generalizable to any scene representation where re-renderings are not prohibitively expensive, and from NeRF-based methods (e.g., iNeRF) by leveraging Plenoxels' rendering speed to re-render the full image at each optimization step rather than subsampling rays.

### [NeRV Training](10253-nerv-training.md)
NeRV training optimizes a continuous volumetric representation that decouples scene geometry (shape) from material appearance (reflectance) and lighting interactions (visibility) to enable efficient relighting and view synthesis. It differs from standard NeRF training by introducing a separate "visibility MLP" that predicts ray visibility and termination depth, supervised by consistency terms with the primary shape MLP rather than ground-truth labels, allowing the simulation of direct and one-bounce indirect illumination within the training loop.

### [NeRV Rendering](10264-nerv-rendering.md)
NeRV rendering computes the pixel color for a camera ray by integrating reflected radiance along the ray, where the radiance at each sample point is derived from a physically-based shading model that accounts for direct and indirect illumination. It differs from standard NeRF volume rendering by explicitly modeling surface normals and microfacet BRDF parameters via separate MLPs, and by estimating indirect lighting through secondary visibility queries rather than simple volumetric absorption.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
This family comprises architectural modifications and training extensions applied to the NeRF training loop to address specific scene characteristics, such as unbounded environments, partial defocus blur, or large-scale drone-captured imagery. It differs from the general [NeRF Training Loop](970-nerf-training-loop.md) by integrating specialized sampling strategies, preprocessing pipelines, and spatial partitioning schemes rather than altering the core optimization algorithm itself.

### [Aug-NeRF](10294-aug-nerf.md)
Aug-NeRF is a training framework for Neural Radiance Fields (NeRF) that injects worst-case adversarial perturbations into three distinct levels of the pipeline—input coordinates, intermediate features, and pre-rendering output—to implicitly regularize the model with physical grounds. It addresses the generalization gap in NeRF where interpolation between seen views yields inconsistent and non-smooth geometries. Unlike explicit regularizers like Laplacian or Total Variation which constrain MLP flexibility aggressively, Aug-NeRF uses data-adaptive smoothness priors to enhance generalization to unseen views without sacrificing representation flexibility.

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
Cross-Guided Optimization of Radiance Fields (CROP) is a framework that performs multi-view image super-resolution (MVSR) on training-view images during the radiance field optimization process to improve high-resolution novel view synthesis (HRNVS). It fuses single-image super-resolution (SISR) features with voxel-based uncertainty fields to ensure multi-view consistency and high-frequency details. Unlike methods that rely solely on the spectral characteristics of coordinate-based networks for upscaling, CROP explicitly bridges 2D SISR and 5D radiance fields by iteratively refining input images based on rendering errors.

### [Truncated Depth-Based Sampling](10613-truncated-depth-based-sampling.md)
Truncated Depth-Based Sampling (TDBS) is a ray sampling strategy for neural radiance fields that utilizes coarse monocular depth priors to guide point sampling via a truncated normal distribution, improving convergence speed and pose estimation accuracy in joint optimization scenarios. It sets itself apart from uniform sampling and methods like NoPe-NeRF by explicitly modeling the probability of the true surface being near the estimated depth, rather than treating depth merely as a distortion constraint or ignoring the prior's distributional nature.

### [Diffusion-based Regularization for NeRF](10661-diffusion-based-regularization-for-nerf.md)
Diffusion-based regularization incorporates a denoising diffusion model as a prior for the density ($\rho$) and color ($c$) fields in Neural Radiance Fields (NeRF) to mitigate interference factors and improve reconstruction stability in limited-view scenarios. Unlike methods that rely solely on pixel-wise reconstruction losses (e.g., the 'No DDM' variant), this approach uses the diffusion prior to prevent blurry reconstruction results and the disappearance of detailed information. It addresses the challenge of low-quality geometry and appearance when training data is sparse or contains degrading interference.

#### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. The full-resolution supersampling baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

#### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

#### Multi-view SDS
A plug-and-play method that creates multi-view supervision during SDS optimization to address the Janus problem caused by one-by-one view sampling [S23]. Rendered images from 4 sampled views are stitched into a large 2x2 image, and gradients are calculated under the condition of the multi-view normal map [S23]. The method converts a 3D asset into differentiable representations including NeRF, 3DGS, and DMTet through rendering supervision, which are then optimized to improve the quality of the 3D asset [S23].

#### Guided NeRF Optimization for MVS
A multi-view depth estimation method that utilizes conventional reconstruction (sparse SfM+MVS) and learning-based priors over Neural Radiance Fields (NeRF) [S24]. It performs test-time optimization directly over implicit volumes to eliminate the need for accurate pixel matching across views, addressing the shape-radiance ambiguity of NeRF in indoor scenes [S24]. Step 1 runs COLMAP on the test scene to acquire per-view sparse depth maps by projecting fused 3D point clouds after multi-view stereo [S24]. Step 2 adapts a monocular depth network to the target scene by finetuning it on the sparse depth maps from COLMAP using a scale-invariant loss [S24].

#### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S25]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S25]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S25].

#### Dynamically-informed iNeRF Estimator
A state estimation method that uses an iNeRF initialized with a dynamics prior, optimized only via photometric loss, acting as a recursive Bayesian filter with infinite state covariance [S26]. The process loss is set to 0 [S26]. An iNeRF estimator initialized without a dynamics prior will very quickly diverge [S26].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rays} \cdot N_{samples} \cdot D)$, where $N_{iters}$ is the number of training steps, $N_{rays}$ is the batch size, $N_{samples}$ is the number of samples per ray, and $D$ is the network depth/width [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rays} \cdot N_{samples} \cdot D)$ for storing intermediate activations during the forward pass, plus optimizer state $O(D_{params})$ [S5, S14, unofficial]. Chunking reduces peak memory usage [S5, unofficial].
*   **Measured Performance:** Training typically requires 100,000 to 1,000,000 iterations [S11, S14, unofficial]. For a 199-image dataset, ~8M rays are processed per epoch with a batch size of 1024, resulting in ~7,800 iterations per epoch [S17, unofficial].
*   **Scaling:** Performance scales with batch size, but larger batches hinder data loading performance and prevent full GPU occupancy, while smaller batches cap iterations per second [S15, unofficial]. A batch size of $2^{10}$ (1024) is often chosen as a tradeoff [S15, unofficial].
*   **Strengths:** Simple to implement, differentiable end-to-end, produces high-quality novel views [S5, S14, unofficial].
*   **Limitations:** Slow convergence due to high iteration counts, prone to overfitting and "floaters" without regularization, memory-intensive for large scenes [S1][S17, unofficial].
*   **When to use:** Standard view synthesis with dense input views and ground-truth poses [S5, S14, unofficial].
*   **When not to use:** Real-time rendering, sparse view scenarios without specialized initialization, or dynamic scenes without temporal extensions [S1][S17, unofficial].

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, S18, unofficial]. It builds on the concept of coordinate-based neural networks for implicit representations [S3, S18, unofficial]. The method uses stochastic gradient descent with Adam optimizer to minimize the MSE between rendered and observed colors [S5, S14, unofficial]. It is widely used in computer vision and graphics for novel view synthesis, 3D reconstruction, and AR/VR applications [S1, S20].

## Sources
[S1] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
[S2] GreatDrake/reparameterized-volume-sampling, https://github.com/GreatDrake/reparameterized-volume-sampling, fetched 2026-10-26 (unofficial)
[S3] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S4] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S5] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S6] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S7] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S8] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S9] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S10] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S11] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S12] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S13] suvadityamuk/NeRF-Experiments, https://github.com/suvadityamuk/NeRF-Experiments, fetched 2026-10-27 (unofficial)
[S14] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S15] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S16] DESY-FS-PETRA/HoloINR, https://github.com/DESY-FS-PETRA/HoloINR, fetched 2026-10-27 (unofficial)
[S17] 7.6. Neural Radiance Fields for Drones, https://www.roboticsbook.org/S76_drone_learning.html, fetched 2026-10-29 (unofficial)
[S18] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S19] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
[S20] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S21] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S22] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S23] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S24] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S25] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S26] Vision-Only Robot Navigation in a Neural Radiance World, https://doi.org/10.1109/lra.2022.3150497, fetched 2026-10-26
