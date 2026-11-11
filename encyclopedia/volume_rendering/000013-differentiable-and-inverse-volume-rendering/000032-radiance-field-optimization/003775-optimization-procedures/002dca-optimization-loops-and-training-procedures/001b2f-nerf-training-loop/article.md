# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview

The NeRF Training Loop is the iterative optimization procedure that learns a continuous volumetric radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ by minimizing the difference between rendered images and ground-truth observations [S5, S6, S14, unofficial]. It solves the problem of reconstructing 3D geometry and appearance from 2D images by differentiating the volume rendering equation through a neural network [44-radiance-field-optimization.md]. What sets it apart from its siblings is its reliance on stochastic ray sampling and standard gradient descent (typically Adam) to update MLP weights, whereas [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md) decouples geometry and appearance, and [Photometric Optimization](5790-photometric-optimization.md) focuses on specific loss formulations rather than the core optimization mechanics.

## How it works

**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics $C$ and intrinsics $K$), and a neural network architecture (typically an MLP) parameterized by $\Theta$.
*   **Outputs:** Optimized network weights $\Theta$ that define the radiance field.

**Data Structures:**
*   **Ray Bundle:** A tensor of shape $[B, 3, 3]$ or $[B, 6]$ (depending on implementation) containing $B$ rays. Each ray consists of origin $\mathbf{o}$, direction $\mathbf{d}$, and target color $\mathbf{c}_{gt}$ [S5, S14, unofficial]. In pre-computed batching, rays are stored in a flat array `rays_rgb` of shape $[(N-1) \cdot H \cdot W, 3, 3]$ where axis 1 is $[\mathbf{o}, \mathbf{d}, \mathbf{c}_{gt}]$ [S5, S14, unofficial].
*   **Optimizer State:** Maintained by the optimizer (e.g., Adam), storing exponential moving averages of gradients and squared gradients.

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S9, S11, unofficial].
    *   Initialize the neural network weights $\Theta$. For standard NeRF, weights are initialized from a normal distribution or specific SIREN initialization [S3, S15, unofficial].
    *   Initialize the optimizer (typically Adam) with a learning rate $\eta$ (e.g., $5 \times 10^{-4}$) [S5, S11, S17, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Pre-compute ray bundles if using batching: generate all possible rays from training images, stack origins, directions, and target RGBs, and shuffle the array [S5, S14, unofficial].

2.  **Iteration Loop:** For iteration $i$ from 1 to $N_{iters}$ (typically 100,000–1,000,000) [S11, S14, unofficial]:
    a.  **Ray Sampling:**
        *   Select a batch of $N_{rand}$ rays (e.g., 1024 or 2048) from the pre-computed pool or randomly from current training images [S5, S14, unofficial].
        *   If using pre-computed batching, slice `rays_rgb[i_batch : i_batch + N_{rand}]` and increment `i_batch`. If `i_batch` exceeds the total number of rays, shuffle the pool and reset [S5, S14, unofficial].
        *   If not using pre-computed batching, randomly select pixel coordinates from training images, convert to ray origins/directions using camera parameters, and gather target colors [S14, unofficial].
        *   *Optional:* Apply perturbations to sampling positions if `perturb > 0` to avoid aliasing [S5, unofficial].
    b.  **Forward Pass (Rendering):**
        *   Sample $K$ points along each ray within the near/far bounds [S5, S13, unofficial].
        *   Query the neural network $F_\Theta$ at each point to obtain density $\sigma$ and color $\mathbf{c}$ [S5, S6, unofficial].
        *   Compute the rendered color $\hat{\mathbf{C}}$ using the differentiable volume rendering equation (DVR) [44-radiance-field-optimization.md]:
            $$
            \hat{C}(\mathbf{r}) = \sum_{i=1}^K T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
            $$
            where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [44-radiance-field-optimization.md].
        *   *Optional (Coarse-to-Fine):* If using a coarse network, render a coarse image $\hat{\mathbf{C}}_{coarse}$, sample additional points based on importance weights, and render a fine image $\hat{\mathbf{C}}_{fine}$ [S5, S11, unofficial].
    c.  **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between the rendered color $\hat{\mathbf{C}}$ and the ground truth color $\mathbf{C}_{gt}$ [S5, S14, unofficial]:
            $$
            \mathcal{L}_{MSE} = \frac{1}{B} \sum_{b=1}^B \| \hat{\mathbf{C}}_b - \mathbf{C}_{gt,b} \|^2
            $$
        *   *Optional:* Add auxiliary losses (e.g., coarse loss $\mathcal{L}_{coarse}$, regularization terms like Eikonal or Total Variation) [S7, unofficial][S8][S11, unofficial]. Total loss $\mathcal{L} = \mathcal{L}_{MSE} + \lambda_{aux} \mathcal{L}_{aux}$.
    d.  **Backpropagation and Update:**
        *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
        *   Compute gradients: `loss.backward()` [S18, unofficial].
        *   Update weights: `optimizer.step()` [S18, unofficial].
        *   Update learning rate according to the scheduler [S9, S18, unofficial].
    e.  **Logging and Checkpointing:**
        *   Log loss and PSNR ($PSNR = -10 \log_{10}(\mathcal{L}_{MSE})$) every $I_{print}$ steps [S14, unofficial].
        *   Render validation images every $I_{img}$ steps [S2, S11, unofficial].
        *   Save model checkpoint every $I_{weights}$ steps [S2, S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader

def train_nerf_loop(model, train_dataset, val_dataset, config):
    """
    Standard NeRF training loop.
    
    Args:
        model: Neural Radiance Field MLP.
        train_dataset: Dataset returning (rays_o, rays_d, rgb_gt).
        val_dataset: Validation dataset.
        config: Dictionary containing hyperparameters.
    """
    # 1. Initialization
    optimizer = optim.Adam(model.parameters(), lr=config['lr'])
    scheduler = optim.lr_scheduler.ExponentialLR(optimizer, gamma=config['decay_rate'])
    mse_loss = nn.MSELoss()
    
    # Pre-compute rays if batching is enabled
    if config['use_batching']:
        rays_rgb = precompute_rays(train_dataset) # Shape: [N_total_rays, 3, 3]
        rays_rgb = shuffle(rays_rgb)
        i_batch = 0
    else:
        data_loader = DataLoader(train_dataset, batch_size=config['batch_size'], shuffle=True)

    n_iters = config['n_iters']
    
    for i in range(n_iters):
        # 2a. Ray Sampling
        if config['use_batching']:
            # Slice batch from pre-computed array
            batch = rays_rgb[i_batch : i_batch + config['batch_size']]
            i_batch += config['batch_size']
            if i_batch >= len(rays_rgb):
                rays_rgb = shuffle(rays_rgb)
                i_batch = 0
            # batch shape: [B, 3, 3] -> [origins, dirs, targets]
            rays_o = batch[:, 0, :]
            rays_d = batch[:, 1, :]
            target_s = batch[:, 2, :]
        else:
            # Sample from DataLoader
            rays_o, rays_d, target_s = next(iter(data_loader))
            
        # Move to device
        rays_o = rays_o.to(config['device'])
        rays_d = rays_d.to(config['device'])
        target_s = target_s.to(config['device'])

        # 2b. Forward Pass
        # Render rays
        # render_rays returns rgb, depth, acc, extras (including coarse rgb if applicable)
        rgb, depth, acc, extras = render_rays(
            model, rays_o, rays_d, 
            near=config['near'], far=config['far'],
            N_samples=config['N_samples'],
            perturb=config['perturb'],
            chunk=config['chunk'] # Memory management
        )

        # 2c. Loss Computation
        loss = mse_loss(rgb, target_s)
        
        # Add coarse loss if using coarse-to-fine
        if 'rgb0' in extras:
            loss = loss + mse_loss(extras['rgb0'], target_s)
            
        # Add regularization if defined
        if 'eikonal_loss' in extras:
            loss = loss + config['eikonal_weight'] * extras['eikonal_loss']

        # 2d. Backpropagation and Update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Update scheduler
        scheduler.step()

        # 2e. Logging
        if i % config['i_print'] == 0:
            psnr = -10.0 * torch.log10(loss)
            print(f"Iter {i}: Loss {loss.item():.6f}, PSNR {psnr.item():.2f}")
            
        if i % config['i_img'] == 0:
            render_validation(model, val_dataset, i)
            
        if i % config['i_weights'] == 0:
            save_checkpoint(model, optimizer, i, config['checkpoint_path'])

    return model
```

**Parameters with Typical Values:**
*   `lr`: $5 \times 10^{-4}$ [S11, unofficial].
*   `N_rand` (batch size): 1024 or 2048 [S5, S11, unofficial].
*   `N_samples` (points per ray): 128–256 [S17, S13, unofficial].
*   `decay_rate`: 0.1 [S6, S18, unofficial].
*   `decay_steps`: `args.lrate_decay * 1000` [S5, S18, unofficial].
*   `chunk`: Max rays per forward pass to limit VRAM (e.g., 4096) [S5, S7, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by chunking rays [S5, S7, unofficial].
*   **Gradient Graph Freeing:** Higher-order derivatives (e.g., for logging gradients) must be computed before `loss.backward()` [S3, unofficial].
*   **Checkpoint Resumption:** When resuming, the scheduler's last epoch must be set to `start_epoch - 1` to maintain correct decay [S9, unofficial].
*   **Instability in Growth:** In dynamic growth methods, randomly initialized sub-networks may collapse to zero density. Initialize grown sub-network alpha weights to match the parent [S1].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images from a neural radiance field (NeRF) during optimization. It solves the problem of computing gradients for scene parameters by sampling points along camera rays, querying a neural network for volume density and color, and integrating them via alpha compositing. It sets itself apart from evaluation by using stochastic perturbations, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the static and dynamic hyperparameters, optimizer settings, and schedule strategies required to stabilize and accelerate the differentiable volume rendering optimization loop. It solves the problem of divergent or slow convergence in non-convex radiance field objectives by managing learning rates, batch sizes, sampling densities, and regularization weights. What sets this method apart from its siblings is that it provides the *initialization and scheduling* parameters (e.g., base learning rates, decay schedules, initial alpha values) rather than the execution logic ([Training Loop Setup](1522-training-loop-setup.md)) or the specific loss formulations ([Regularization and Loss Functions](8586-regularization-and-loss-functions.md)).

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Regularization and Loss Functions define the objective criteria used to optimize Neural Radiance Fields (NeRFs) beyond simple photometric reconstruction. They solve the under-constrained nature of volume rendering, particularly in sparse-view settings, by preventing artifacts such as "floaters," enforcing geometric smoothness, and aligning semantic content. This family is distinguished from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the *what* (the loss terms) rather than the *how* (the integration method), and from [Training Configuration](2332-training-configuration.md) by providing the mathematical definitions of the optimization targets.

### [KNN-NeRF](8672-knn-nerf.md)
KNN-NeRF is a baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Progressive and Dynamic Scene Growth addresses the limitations of static neural radiance field (NeRF) architectures by iteratively expanding the model's capacity or training data during optimization. This family of methods solves the problem of representing complex, large-scale, or dynamically changing scenes with a single fixed network by coupling network topology or view selection to training uncertainty and geometric coherence. It sets itself apart from static NeRF variants by dynamically adding capacity where the current model exhibits high uncertainty or by progressively incorporating new views to resolve occlusions and scale to urban environments.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Adaptive ray sampling refers to a family of training procedures in neural radiance fields (NeRF) that replace uniform ray sampling with strategies that concentrate computational resources on regions of high geometric or photometric complexity. It solves the inefficiency of uniform sampling, which wastes evaluations on empty space or smooth regions, by dynamically adjusting sample locations or ray selection based on density, error, depth priors, or spatial partitions. This family is distinguished from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on *where* to sample rather than *how* to integrate, and from [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) by encompassing broader strategies including error-based mining, depth-guided selection, and superpixel partitioning, not just hierarchical point clustering.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
This family addresses the degradation of NeRF quality—such as blurred geometry, slow convergence, and floaters—when trained on sparse input views (typically 15–20 images or fewer than 50–100 views). It distinguishes itself from standard NeRF training by decoupling the initialization and regularization processes from the default random or empty-space assumptions, instead leveraging external geometric scaffolds, semantic masks, or visibility priors to anchor the optimization.

### [NeRV Training](10253-nerv-training.md)
NeRV training optimizes a continuous volumetric representation that decouples scene geometry (shape) from material appearance (reflectance) and lighting interactions (visibility) to enable efficient relighting and view synthesis. It differs from standard NeRF training by introducing a separate "visibility MLP" that predicts ray visibility and termination depth, supervised by consistency terms with the primary shape MLP rather than ground-truth labels, allowing the simulation of direct and one-bounce indirect illumination within the training loop.

### [Aug-NeRF](10294-aug-nerf.md)
Aug-NeRF is a training framework for Neural Radiance Fields (NeRF) that injects worst-case adversarial perturbations into three distinct levels of the pipeline—input coordinates, intermediate features, and pre-rendering output—to implicitly regularize the model with physical grounds. It addresses the generalization gap in NeRF where interpolation between seen views yields inconsistent and non-smooth geometries. Unlike explicit regularizers like Laplacian or Total Variation which constrain MLP flexibility aggressively, Aug-NeRF uses data-adaptive smoothness priors to enhance generalization to unseen views without sacrificing representation flexibility.

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
Cross-Guided Optimization of Radiance Fields (CROP) is a framework that performs multi-view image super-resolution (MVSR) on training-view images during the radiance field optimization process to improve high-resolution novel view synthesis (HRNVS). It fuses single-image super-resolution (SISR) features with voxel-based uncertainty fields to ensure multi-view consistency and high-frequency details. Unlike methods that rely solely on the spectral characteristics of coordinate-based networks for upscaling, CROP explicitly bridges 2D SISR and 5D radiance fields by iteratively refining input images based on rendering errors.

### [Application-Specific Pipelines](11200-application-specific-pipelines.md)
Application-Specific Pipelines are specialized training and optimization procedures tailored to distinct downstream tasks, diverging from the standard NeRF reconstruction loop to address specific constraints such as monocular pose estimation or text-guided scene editing. Unlike the general [NeRF Training Loop](970-nerf-training-loop.md) which optimizes a single radiance field for novel view synthesis, these pipelines decouple or augment the optimization objectives to handle tasks like 6-DoF camera pose recovery in Plenoxels environments or compositional object insertion into existing NeRF scenes.

### [NeRF-GT Baseline Training Setup](11349-nerf-gt-baseline-training-setup.md)
NeRF-GT is a baseline training protocol for Neural Radiance Fields that utilizes High Dynamic Range (HDR) input views to synthesize novel HDR imagery. It addresses the limitation of standard NeRF, which typically trains on Low Dynamic Range (LDR) images, by directly optimizing the radiance field against HDR ground truth. It differs from standard LDR-based training by explicitly handling high dynamic range data, though it faces challenges in encoding all scene areas uniformly due to the intensity variance.

### [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)
This family defines the neural network structures and data representations used to approximate the continuous volumetric radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ within the NeRF Training Loop [970-nerf-training-loop.md]. It solves the problem of how to parameterize scene geometry and appearance for differentiable optimization. It is distinguished from sibling methods by focusing on the internal topology of the scene model itself, rather than the sampling strategy [2199-training-ray-marching-and-integration.md], hyperparameters [2332-training-configuration.md], or loss functions [8586-regularization-and-loss-functions.md].

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
This family of methods accelerates NeRF training and rendering by reducing the computational cost of evaluating the neural radiance field. It addresses the bottleneck of querying high-dimensional MLPs for every sample point along every ray. Unlike standard uniform or adaptive sampling over the entire scene bounding box, these methods restrict evaluation to sparse structures (visual hulls) or use lightweight approximations (caches) to predict radiance, thereby reducing the number of expensive neural network queries.

### [Deceptive-NeRF/3DGS Pipeline](11860-deceptive-nerf-3dgs-pipeline.md)
Deceptive-NeRF/3DGS is a sparse-view reconstruction method that enhances NeRF or 3D Gaussian Splatting (3DGS) by continuously densifying input observations with synthetic pseudo-observations generated by a diffusion model during training. It distinguishes itself from sibling methods by introducing an uncertainty map derived from view-consistency warping to guide the diffusion process, ensuring that generated pseudo-observations adhere to epipolar constraints and 3D geometry, rather than relying on ensemble learning or variational inference for uncertainty estimation.

### [Urban Radiance Fields](11973-urban-radiance-fields.md)
Urban Radiance Fields is a method for 3D reconstruction and view synthesis in large-scale urban environments using sparse multi-view images supplemented by LiDAR data. It addresses the challenges of unbounded outdoor scenes, limited camera pose variety, and large-scale geometry by incorporating specific depth, sight, and sky segmentation losses into the standard NeRF training loop. Unlike standard NeRF which relies solely on photometric consistency, this method enforces geometric constraints via LiDAR and semantic constraints via sky segmentation to stabilize training in open environments.

### [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md)
This family of methods addresses the problem of optimizing camera parameters (extrinsics and intrinsics) simultaneously with Neural Radiance Field (NeRF) weights, eliminating the reliance on pre-computed Structure-from-Motion (SfM) pipelines like COLMAP. By treating camera poses as learnable parameters, these methods enable 3D reconstruction from sparse or uncalibrated image sets. What sets this family apart from siblings like [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md) is the explicit optimization of discrete camera poses per image rather than continuous motion functions, and from [Camera Preconditioning](12545-camera-preconditioning.md) by focusing on the joint gradient-based update of poses and scene geometry rather than preconditioning the optimizer for faster convergence.

### [DreamFusion](12087-dreamfusion.md)
DreamFusion is a text-to-3D generation method that optimizes a Neural Radiance Field (NeRF) to render images with high scores from a 2D text-to-image diffusion model, requiring no 3D training data. It differs from standard NeRF training by using a pretrained diffusion model as a perceptual loss function instead of ground-truth pixel comparisons, enabling the synthesis of novel 3D objects from text prompts.

### [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md)
This family of methods integrates Monocular Depth Estimation (MDE) priors into Neural Radiance Field (NeRF) training to resolve the shape-radiance ambiguity and improve geometric accuracy, particularly in sparse-view or few-shot settings. Unlike standard NeRF which relies solely on photometric consistency, these methods use MDE to guide volume sampling bounds, regularize depth distributions via Earth Mover’s Distance, or adapt the MDE network itself to the scene's absolute geometry through bidirectional knowledge transfer. They differ from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by explicitly injecting external geometric constraints derived from 2D depth priors into the 3D optimization loop.

### [Adaptive View Selection for Inpainting](12405-adaptive-view-selection-for-inpainting.md)
This method addresses the inconsistency of single-view 2D inpainting in 3D scenes by integrating a 2D appearance prior with multi-view geometric consistency. It differs from standard NeRF training [NeRF Training Loop](970-nerf-training-loop.md) by dynamically switching the supervision signal for occluded regions between a 2D inpainter (LaMa) and other training views based on local texture richness. It builds on [NeRF Training Loop](970-nerf-training-loop.md) and [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md) by adding a gradient-based view selection mechanism to resolve conflicts between 2D priors and 3D geometry.

### [RehearsalNeRF](12477-rehearsalnerf.md)
RehearsalNeRF is a method for learning disentangled neural fields under severe illumination changes by leveraging scenes captured under stable lighting (rehearsal stages) to enforce geometric consistency between different lighting conditions. It jointly optimizes five neural fields—static objects, dynamic objects, and dynamically changing illuminations—to decouple dynamic lighting from moving/static objects, addressing the "illumination-radiance ambiguity" where illumination radiance is entangled with object radiance. Unlike methods that assume stable light sources, RehearsalNeRF explicitly handles varying illuminations and dynamic objects, enabling applications such as controlling lighting while stopping motion.

### [NeuMan](12480-neuman.md)
NeuMan is a method for novel view synthesis of a human subject interacting with a static scene, driven by a single video input. It jointly optimizes a background scene NeRF and a pose-driven human NeRF, allowing for the generation of novel views with novel human poses. It distinguishes itself from standard NeRF training by explicitly modeling the human as a deformable entity in a canonical space using SMPL parameters and an error-correction network, while simultaneously separating the human from the background to enable clean compositing.

### [AR-NeRF](12493-ar-nerf.md)
AR-NeRF is an extension of Neural Radiance Fields (NeRF) that employs an aperture camera model instead of a pinhole camera model to represent defocus effects and viewpoint changes in a unified manner within a common ray-tracing framework. It enables the fully unsupervised learning of depth and defocus effects from single natural images by jointly leveraging defocus and viewpoint cues without explicit supervision or pretrained models. Unlike standard NeRF, which assumes a pinhole camera, AR-NeRF represents each pixel using a collection of rays converging at a focus plane, allowing the model to disentangle defocus-aware and defocus-independent representations through aperture randomized training.

### [NeRF Editing and Inpainting Procedures](12494-nerf-editing-and-inpainting-procedures.md)
NeRF Editing and Inpainting Procedures are optimization-based methods that modify a pre-trained Neural Radiance Field (NeRF) to remove unwanted objects, insert new content, or decompose scenes into semantic regions, without requiring retraining from scratch. Unlike standard training loops that optimize for photometric reconstruction of input views, these methods optimize the model parameters $\Theta$ using auxiliary guidance signals—such as 2D inpainted images, semantic feature maps, or text prompts—to enforce specific visual or geometric constraints in novel views. They are distinguished from sibling methods like [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md) (which builds geometry from a single view) and [Blended-NeRF](12273-blended-nerf.md) (which uses a generator NeRF) by their direct parameter optimization of the existing radiance field using explicit loss functions derived from external guidance.

### [NeRF Ensemble with Conjoint Pruning](12608-nerf-ensemble-with-conjoint-pruning.md)
NeRF Ensemble with Conjoint Pruning is a training strategy that improves novel view synthesis quality by combining two Neural Radiance Field (NeRF) models of different grid resolutions, specifically using the Direct Voxel Grid Optimization (DVGO) architecture. It addresses the high computational requirements and optimization difficulties of fine-grained voxel grids by achieving superior performance compared to single models while maintaining a similar parameter count and memory footprint. Unlike vanilla ensembling, which may yield suboptimal results, this method employs a specific three-stage process involving independent training, fine-tuning, and conjoint pruning to prevent performance degradation.

### [Neural SAR Surface Reconstruction via Inverse Rendering](12668-neural-sar-surface-reconstruction-via-inverse-rendering.md)
This method reconstructs a Digital Surface Model (DSM) and a backscattering coefficient map from Synthetic Aperture Radar (SAR) images by minimizing the difference between observed SAR intensities and synthesized intensities using a differentiable rendering model. It models elevation $Z$ and backscattering $B$ within a bounding box using a Multi-Layer Perceptron (MLP), reducing sampling degrees of freedom from a 3D volume to a 2D surface. Compared to standard volume rendering, this surface representation is more constrained, leading to faster and easier reconstruction of surfaces.

### [SPARF](12705-sparf.md)
SPARF is a joint pose-NeRF training strategy designed for novel view synthesis using a sparse set of input images (as few as 3) with noisy camera pose estimates. It solves the overfitting and geometric inconsistency inherent in sparse-view regimes by replacing the standard photometric loss with multi-view geometry constraints, specifically a multi-view correspondence loss and a depth consistency loss. Unlike sibling methods like BARF or SCNeRF that rely on independent photometric supervision, SPARF explicitly enforces global geometric consistency across all training views through pixel correspondences.

### [MLI-NeRF](12709-mli-nerf.md)
MLI-NeRF is a two-stage optimization procedure that learns an intrinsic-aware Neural Radiance Field (NeRF) capable of novel view synthesis, relighting, and intrinsic decomposition (separating reflectance and shading) without relying on statistical priors or ground-truth intrinsic labels. It differs from standard NeRF variants by explicitly modeling multiple light sources to generate physics-based pseudo-labels, and it differs from prior intrinsic NeRF methods (like [IntrinsicNeRF](10253-nerv-training.md)) by leveraging 3D geometric constraints and multi-light visibility rather than chromaticity or semantic statistical constraints. The method solves the ill-posed nature of inverse rendering by decoupling the geometry learning (Stage 1) from the intrinsic component separation (Stage 2), using pseudo-labels derived from the Stage 1 model to guide the decomposition.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. The full-resolution supersampling baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### RapNeRF
A NeRF model tailored for view extrapolation, contrasting with standard NeRFs that excel at interpolation [S24]. Training begins with a baseline NeRF to recover a rough mesh [S24]. A Ray Atlas (RA) is created by extracting a rough 3D mesh from a pretrained NeRF and mapping training ray directions onto its vertices [S24]. Random Ray Casting (RRC) is employed: for a training ray hitting a surface point $v = o + tz d$, a backward ray is cast from $v$ toward a new origin $o'$ with uniformly sampled angular perturbations [S24]. RRC and RA augment training rays with a predetermined probability [S24].

### JaxNeRF
A re-implementation of the original NeRF in JAX, used as a benchmark comparison for early works seeking to improve on training and rendering speed [S24]. It is slightly faster and more suited for distributed computing than the original TensorFlow implementation of NeRF [S24].

### SNeS
A method improving geometry by learning probable symmetries for partly symmetric and partly hidden in-scene objects through soft symmetry constraints on geometry and material properties [S24].

### DreamAvatar
A diffusion-based 3D avatar model generation method with text guidance, using an SMPL model as a shape prior and text-guided 2D image generation via diffusion to create training data in a DreamFusion-like 3D generation pipeline combining NeRF and diffusion [S24].

### FlipNeRF Training Pipeline
Starts with a baseline mip-NeRF representation [S25]. Adds flipped reflection rays to the training loop [S25]. Applies LUE to both original and flipped rays [S25]. Applies LBFC with JSD on bottleneck features [S25]. Enforces bottleneck feature consistency between original and flipped ray pairs [S25]. Integrates LOri from Ref-NeRF to refine normal estimation [S25]. The full model achieves PSNR 19.55, SSIM 0.767, LPIPS 0.180, Average Error 0.101, and NLL 2.56 on Realistic Synthetic 360◦ [S25].

## Operating characteristics

*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot K \cdot C)$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), $K$ is the number of samples per ray, and $C$ is the cost of the neural network forward pass [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot K \cdot D)$, where $D$ is the dimensionality of the network activations. VRAM usage is mitigated by chunking rays [S5, unofficial].
*   **Measured Performance:** Standard NeRF training on Blender scenes (800x800) takes approximately 5–10 hours on a single GPU (e.g., RTX 2080 Ti) for 500,000 iterations [S5, S11, unofficial].
*   **Scaling:** Performance scales with batch size, but larger batches hinder data loading performance and prevent full GPU occupancy. A batch size of $2^{10}$ (1024) is often chosen as a trade-off [S15, unofficial].
*   **Strengths:** Simple to implement, robust convergence with Adam optimizer, high-quality novel view synthesis.
*   **Limitations:** Slow training due to MLP queries, prone to overfitting on sparse views, requires pre-computed camera poses (unless optimized jointly).
*   **When to Use:** Standard novel view synthesis from multi-view images.
*   **When Not to Use:** When real-time training is required (use [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)), or when only sparse views are available (use [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)).

## Background

The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial]. It builds on the concept of coordinate-based neural networks for implicit scene representation [S3, unofficial]. The method was initially implemented in TensorFlow [S5, unofficial] and later ported to PyTorch [S6, S13, unofficial]. It has become the standard baseline for subsequent NeRF variants and 3D generation methods like DreamFusion [S2, unofficial].

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
[S23] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S24] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S25] FlipNeRF: Flipped Reflection Rays for Few-shot Novel View Synthesis, https://doi.org/10.1109/iccv51070.2023.02092, fetched 2026-10-29
