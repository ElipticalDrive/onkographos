# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview

The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth observations by updating the parameters of a continuous volumetric radiance field [S5, S6, S14, unofficial]. It solves the inverse rendering problem by differentiating the volume rendering equation through automatic differentiation, enabling gradient-based learning of scene geometry and appearance [44-radiance-field-optimization.md]. It sets itself apart from evaluation rendering by using stochastic ray sampling and perturbation to facilitate gradient descent, whereas evaluation uses deterministic, dense sampling for reproducible metrics [S2, S9, unofficial].

## How it works

**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics $C2W$ and intrinsics $K$), and a neural network model (typically an MLP) parameterizing the radiance field $\mathbf{F}_\theta$.
*   **Outputs:** Updated network weights $\theta$ that minimize the loss between predicted and observed pixel colors.

**Data Structures:**
*   **Rays:** A ray bundle is represented as a tensor of shape $[B, 3, 3]$ or $[B, 6]$ depending on implementation, where $B$ is the batch size (number of rays). The first 3 components are the ray origin $(o_x, o_y, o_z)$, the next 3 are the ray direction $(d_x, d_y, d_z)$ [S5, S14, unofficial].
*   **Pre-computed Rays:** To accelerate data loading, rays are often pre-computed and stored in a flat array `rays_rgb` of shape $[(N-1) \cdot H \cdot W, 3, 3]$ (or $[N_{total}, 9]$), where axis 1 contains [origin, direction, target_rgb] and axis 2 contains the XYZ/RGB components [S5, S14, unofficial]. This array is typically shuffled at the start of each epoch [S11, unofficial].
*   **Checkpoint:** A dictionary containing the model state dict (`coarse`, `fine`), optimizer state dict (`optim`), and current step/epoch count [S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the NeRF model(s). For coarse-to-fine training, initialize both coarse and fine MLPs [S11, unofficial].
    *   Initialize the optimizer (typically Adam) with a learning rate $lr$ (default $5 \times 10^{-4}$) [S11, S17, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   If resuming, load checkpoint and set scheduler last epoch to `start_epoch - 1` [S9, unofficial].

2.  **Ray Sampling (Per Iteration):**
    *   Select a batch of $N_{rand}$ rays (default 1024 or 2048) [S5, S11, unofficial].
    *   If using pre-computed batching: Slice `rays_rgb[i_batch : i_batch + N_{rand}]`, transpose to $[3, B, 3]$, and split into `batch_rays` (origins/dirs) and `target_s` (colors) [S5, S14, unofficial]. Increment `i_batch`. If `i_batch` exceeds total rays, shuffle `rays_rgb` and reset [S11, unofficial].
    *   If not using pre-computed batching: Randomly select an image index, generate all rays for that image, optionally apply a central crop (precrop) for the first $P$ iterations, and sample $N_{rand}$ unique pixel coordinates without replacement [S14, unofficial].
    *   Perturb sampling times $t$ along each ray by adding uniform noise in $[0, 1]$ to break symmetry and aid optimization [S5, S2, unofficial].

3.  **Forward Pass (Rendering):**
    *   Sample points along each ray in `batch_rays` using stratified sampling between near and far bounds [S5, unofficial].
    *   Query the neural network for density $\sigma$ and color $\mathbf{c}$ at these points [S5, unofficial].
    *   Compute the rendered color $\hat{C}$ using the emission-absorption model (DVR) [44-radiance-field-optimization.md]:
        $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
        where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [44-radiance-field-optimization.md].
    *   If using coarse-to-fine training, render a coarse image $\hat{C}_{coarse}$, then sample additional points based on coarse weights (importance sampling) and render a fine image $\hat{C}_{fine}$ [S5, S11, unofficial].

4.  **Loss Computation:**
    *   Compute the Mean Squared Error (MSE) between predicted RGB $\hat{C}$ and ground truth RGB $C_{gt}$ [S5, S14, unofficial]:
        $$ \mathcal{L}_{rgb} = \frac{1}{B} \sum_{b=1}^B \| \hat{C}_b - C_{gt,b} \|^2 $$
        [S14, S17, unofficial].
    *   If coarse-to-fine training is used, add the coarse loss $\mathcal{L}_{coarse}$ to the total loss: $\mathcal{L} = \mathcal{L}_{fine} + \mathcal{L}_{coarse}$ [S11, unofficial].
    *   (Optional) Add regularization terms (e.g., sparsity, TV, Eikonal) if specified in the configuration [S7, unofficial][S8].

5.  **Backpropagation and Update:**
    *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
    *   Compute gradients: `loss.backward()` [S18, unofficial].
    *   Update weights: `optimizer.step()` [S18, unofficial].
    *   Update learning rate according to the scheduler [S9, S18, unofficial].

6.  **Logging and Checkpointing:**
    *   Log loss and PSNR every $I_{print}$ steps (default 100) [S2, S14, unofficial].
    *   Render validation images every $I_{img}$ steps (default 500) [S2, unofficial].
    *   Save checkpoint every $I_{weights}$ steps (default 10,000) [S2, S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader

def train_nerf(model, train_dataset, val_dataset, config):
    """
    Standard NeRF Training Loop.
    
    Args:
        model: Neural Radiance Field MLP (coarse and/or fine).
        train_dataset: Dataset yielding (rays_o, rays_d, target_rgb, camera_pose).
        val_dataset: Dataset for validation.
        config: Dictionary containing hyperparameters.
    """
    # 1. Initialization
    optimizer = optim.Adam(model.parameters(), lr=config['lr'])
    # Example: Exponential Decay Scheduler
    scheduler = optim.lr_scheduler.ExponentialLR(optimizer, gamma=config['decay_rate'])
    
    # Pre-compute rays if batching is enabled
    if config['use_batching']:
        rays_rgb = precompute_rays(train_dataset) # Shape: [N_total, 9]
        rays_rgb = rays_rgb.shuffle() # Shuffle indices
        i_batch = 0
    
    n_iters = config['n_iters'] # e.g., 100,000
    batch_size = config['batch_size'] # e.g., 1024
    
    for step in range(n_iters):
        # 2. Ray Sampling
        if config['use_batching']:
            # Slice pre-computed batch
            batch_indices = slice(i_batch, i_batch + batch_size)
            batch_data = rays_rgb[batch_indices] # [B, 9]
            rays_o_d = batch_data[:, :6] # Origins and Directions
            target_rgb = batch_data[:, 6:] # Ground Truth Colors
            
            i_batch += batch_size
            if i_batch >= len(rays_rgb):
                rays_rgb.shuffle()
                i_batch = 0
        else:
            # Random image and pixel sampling
            rays_o_d, target_rgb = sample_random_rays(train_dataset, batch_size)
            
        # 3. Forward Pass
        optimizer.zero_grad()
        
        # Render coarse and fine if applicable
        render_kwargs = {
            'near': config['near'],
            'far': config['far'],
            'N_samples': config['N_samples'],
            'perturb': config['perturb'],
            'use_viewdirs': config['use_viewdirs']
        }
        
        outputs = model(rays_o_d, **render_kwargs)
        rgb_coarse = outputs['rgb_coarse']
        rgb_fine = outputs['rgb_fine']
        
        # 4. Loss Computation
        # MSE Loss
        loss_fine = nn.functional.mse_loss(rgb_fine, target_rgb)
        loss_coarse = nn.functional.mse_loss(rgb_coarse, target_rgb) if 'rgb_coarse' in outputs else torch.tensor(0.0)
        
        total_loss = loss_fine + loss_coarse
        
        # Optional: Add regularization (e.g., sparsity)
        if 'sparse_loss_weight' in config and config['sparse_loss_weight'] > 0:
            total_loss += config['sparse_loss_weight'] * compute_sparsity_loss(model)
            
        # 5. Backpropagation
        total_loss.backward()
        optimizer.step()
        
        # 6. Logging
        if step % config['i_print'] == 0:
            psnr = -10.0 * torch.log10(loss_fine.item())
            print(f"Step {step}: Loss={total_loss.item():.4f}, PSNR={psnr:.2f}")
            
        if step % config['i_weights'] == 0:
            save_checkpoint(model, optimizer, step, config['checkpoint_path'])
            
        scheduler.step()
```

**Parameters with Typical Values:**
*   `lr`: $5 \times 10^{-4}$ [S11, unofficial].
*   `batch_size` ($N_{rand}$): 1024 or 2048 [S5, S11, unofficial].
*   `n_iters`: 100,000 [S11, unofficial].
*   `decay_rate`: 0.1 [S6, S18, unofficial].
*   `decay_steps`: `args.lrate_decay * 1000` [S5, S18, unofficial].
*   `near` / `far`: Scene-dependent (e.g., 2.0/6.0 for Blender) [S2, S13, unofficial].
*   `perturb`: 1.0 for training (jitter), 0.0 for evaluation [S2, S5, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Large batches or high-resolution rendering can exceed GPU memory. Mitigated by chunking rays during the forward pass (`chunk` parameter) [S5, S14, unofficial].
*   **Gradient Freeing:** Higher-order derivatives (e.g., for logging gradients or Laplacians) must be computed *before* `loss.backward()` to avoid freeing the autograd graph [S3, unofficial].
*   **Instability in Growth:** In dynamic growth methods (e.g., Recursive-NeRF), initializing new sub-network weights to match the parent prevents density collapse to zero [S1].
*   **Empty Checkpoints:** Loading a checkpoint with step 0 should be treated as an error or re-initialization [S15, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images and depth maps from a Neural Radiance Field (NeRF) during optimization. It sets itself apart from evaluation rendering by using stochastic sampling with perturbation and adaptive step sizes to facilitate gradient-based optimization, whereas evaluation uses deterministic rendering for reproducible metrics.

### [Training Configuration](2332-training-configuration.md)
Defines the static and dynamic hyperparameters, optimization schedules, and environment settings required to converge a differentiable volume rendering model. It sets itself apart from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the gradient descent mechanics rather than the forward rendering pass, and from [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md) by defining the temporal evolution of the optimization state rather than the initial spatial representation.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Defines the objective landscape for optimizing Neural Radiance Fields (NeRFs), extending the standard photometric reconstruction loss to address specific failure modes such as sparse-view ambiguity, geometric artifacts (floaters), semantic incoherence, and dynamic scene reconstruction. It addresses the under-constrained nature of inverse rendering by incorporating priors from diffusion models, geometric consistency, semantic embeddings, and physical constraints.

### [KNN-NeRF](8672-knn-nerf.md)
A baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Refers to a family of training strategies that expand the capacity, data, or topology of a neural radiance field during optimization, rather than optimizing a static representation on a fixed dataset. This family solves the problems of representing complex, large-scale, or dynamic scenes with limited initial data, and mitigates issues such as "foggy" densities, floating artifacts, and catastrophic forgetting in online settings. It sets itself apart from static NeRF variants by coupling the network architecture or training data directly to the training error or temporal sequence, growing $k$ new branches per step based on uncertainty, iteratively expanding reference views for panoramic scenes [8666-progressive-view-synthesis-training.md], or continuously updating the field as new camera views arrive [12066-online-incremental-nerf-optimization.md].

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Refers to a family of training procedures in neural radiance fields (NeRF) that replace uniform ray sampling with strategies that concentrate computational resources on regions of high geometric or photometric complexity. It solves the inefficiency of uniform sampling, which wastes evaluations on empty space or smooth regions, by dynamically adjusting sample locations or ray selection based on density, error, depth priors, or spatial partitions. This family is distinguished from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on *where* to sample rather than *how* to integrate, and from [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) by encompassing broader strategies including error-based mining, depth-guided selection, and superpixel partitioning, not just hierarchical point clustering.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
This family addresses the degradation of NeRF quality—such as blurred geometry, slow convergence, and floaters—when trained on sparse input views (typically 15–20 images or fewer than 50–100 views). It distinguishes itself from standard NeRF training by decoupling the initialization and regularization processes from the default random or empty-space assumptions, instead leveraging external geometric scaffolds, semantic masks, or visibility priors to anchor the optimization.

### [NeRV Training](10253-nerv-training.md)
NeRV training optimizes a continuous volumetric representation that decouples scene geometry (shape) from material appearance (reflectance) and lighting interactions (visibility) to enable efficient relighting and view synthesis. It differs from standard NeRF training by introducing a separate "visibility MLP" that predicts ray visibility and termination depth, supervised by consistency terms with the primary shape MLP rather than ground-truth labels, allowing the simulation of direct and one-bounce indirect illumination within the training loop.

### [Aug-NeRF](10294-aug-nerf.md)
Aug-NeRF is a training framework for Neural Radiance Fields (NeRF) that injects worst-case adversarial perturbations into three distinct levels of the pipeline—input coordinates, intermediate features, and pre-rendering output—to implicitly regularize the model with physical grounds. It addresses the generalization gap in NeRF where interpolation between seen views yields inconsistent and non-smooth geometries. Unlike explicit regularizers like Laplacian or Total Variation which constrain MLP flexibility aggressively, Aug-NeRF uses data-adaptive smoothness priors to enhance generalization to unseen views without sacrificing representation flexibility.

### [Application-Specific Pipelines](11200-application-specific-pipelines.md)
Application-Specific Pipelines are specialized training and optimization procedures tailored to distinct downstream tasks, diverging from the standard NeRF reconstruction loop to address specific constraints such as monocular pose estimation or text-guided scene editing. Unlike the general [NeRF Training Loop](970-nerf-training-loop.md) which optimizes a single radiance field for novel view synthesis, these pipelines decouple or augment the optimization objectives to handle tasks like 6-DoF camera pose recovery in Plenoxels environments or compositional object insertion into existing NeRF scenes.

### [NeRF-GT Baseline Training Setup](11349-nerf-gt-baseline-training-setup.md)
NeRF-GT is a baseline training protocol for Neural Radiance Fields that utilizes High Dynamic Range (HDR) input views to synthesize novel HDR imagery. It addresses the limitation of standard NeRF, which typically trains on Low Dynamic Range (LDR) images, by directly optimizing the radiance field against HDR ground truth. It differs from standard LDR-based training by explicitly handling high dynamic range data, though it faces challenges in encoding all scene areas uniformly due to the intensity variance.

### [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)
Scene Representation and Architecture defines the neural network topologies, data structures, and input encodings used to approximate the continuous radiance field in differentiable volume rendering. While the canonical NeRF uses a monolithic Multi-Layer Perceptron (MLP) to map 6D coordinates to density and color, this family encompasses methods that modify the network topology, input representation, or underlying data structures to address limitations in rendering speed, memory consumption, generalization, dynamic scene modeling, and single-view synthesis. These architectures set themselves apart from the canonical baseline by introducing mechanisms such as sparse grids, tensor decompositions, transformer attention, or explicit geometric priors, enabling applications ranging from real-time rendering to 3D-aware image generation.

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
This family of methods accelerates NeRF training and rendering by reducing the computational cost of evaluating the neural radiance field. It addresses the bottleneck of querying high-dimensional MLPs for every sample point along every ray. Unlike standard uniform or adaptive sampling over the entire scene bounding box, these methods restrict evaluation to sparse structures (visual hulls) or use lightweight approximations (caches) to predict radiance, thereby reducing the number of expensive neural network queries.

### [Urban Radiance Fields](11973-urban-radiance-fields.md)
Urban Radiance Fields is a method for 3D reconstruction and view synthesis in large-scale urban environments using sparse multi-view images supplemented by LiDAR data. It addresses the challenges of unbounded outdoor scenes, limited camera pose variety, and large-scale geometry by incorporating specific depth, sight, and sky segmentation losses into the standard NeRF training loop. Unlike standard NeRF which relies solely on photometric consistency, this method enforces geometric constraints via LiDAR and semantic constraints via sky segmentation to stabilize training in open environments.

### [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md)
This family of methods addresses the problem of optimizing camera parameters (extrinsics and intrinsics) simultaneously with Neural Radiance Field (NeRF) weights, eliminating the reliance on pre-computed Structure-from-Motion (SfM) pipelines like COLMAP. By treating camera poses as learnable parameters, these methods enable 3D reconstruction from sparse or uncalibrated image sets. What sets this family apart from siblings like [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md) is the explicit optimization of discrete camera poses per image rather than continuous motion functions, and from [Camera Preconditioning](12545-camera-preconditioning.md) by focusing on the joint gradient-based update of poses and scene geometry rather than preconditioning the optimizer for faster convergence.

### [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md)
This family of methods integrates pre-trained Monocular Depth Estimation (MDE) networks into the NeRF training loop to resolve the shape-radiance ambiguity and scale uncertainty inherent in sparse-view or single-view reconstruction. Unlike standard NeRF which relies solely on photometric consistency, these methods use MDE outputs as geometric priors to guide volume sampling, regularize density distributions, or directly adapt the MDE network to the scene's absolute geometry. They are distinguished from sibling methods like [DäRF](12331-d-rf.md) by their specific strategies for handling the relative nature of monocular depth, ranging from sparse prior fine-tuning to distribution-based Earth Mover’s Distance supervision [11911-depth-guided-nerf-training-via-earth-mover-s-distance.md].

### [NeRF Editing and Inpainting Procedures](12494-nerf-editing-and-inpainting-procedures.md)
This family of procedures modifies a pre-trained or training Neural Radiance Field (NeRF) to alter its content, such as removing objects, changing colors, or filling occluded regions, without requiring full retraining from scratch. Unlike standard training loops that optimize for photometric consistency on input views, these methods introduce external priors (2D inpainting, diffusion models, or semantic features) and specialized loss functions or rendering pipelines to enforce 3D consistency while applying edits. They differ from sibling methods like [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md) by focusing on content modification rather than scene expansion, and from [Regularization and Loss Functions](8586-regularization-and-loss-functions.md) by integrating specific editing objectives (e.g., semantic segmentation, text prompts) into the optimization or rendering process.

### [FlipNeRF Training Pipeline](12501-flipnerf-training-pipeline.md)
FlipNeRF is a training procedure for few-shot novel view synthesis that enhances the mip-NeRF representation by introducing flipped reflection rays into the training loop. It distinguishes itself from standard NeRF pipelines by enforcing consistency between original and geometrically flipped rays through specific uncertainty-aware and bottleneck feature losses, rather than relying solely on color reconstruction.

### [NeRF Ensemble with Conjoint Pruning](12608-nerf-ensemble-with-conjoint-pruning.md)
NeRF Ensemble with Conjoint Pruning is a training strategy that improves novel view synthesis quality by combining two Neural Radiance Field (NeRF) models of different grid resolutions, specifically using the Direct Voxel Grid Optimization (DVGO) architecture. It addresses the high computational requirements and optimization difficulties of fine-grained voxel grids by achieving superior performance compared to single models while maintaining a similar parameter count and memory footprint. Unlike vanilla ensembling, which may yield suboptimal results, this method employs a specific three-stage process involving independent training, fine-tuning, and conjoint pruning to prevent performance degradation.

### [MLI-NeRF](12709-mli-nerf.md)
MLI-NeRF is a two-stage optimization procedure that learns an intrinsic-aware Neural Radiance Field (NeRF) capable of novel view synthesis, relighting, and intrinsic decomposition (separating reflectance and shading) without relying on statistical priors or ground-truth intrinsic labels. It differs from standard NeRF variants by explicitly modeling multiple light sources to generate physics-based pseudo-labels, and it differs from prior intrinsic NeRF methods (like [IntrinsicNeRF](10253-nerv-training.md)) by leveraging 3D geometric constraints and multi-light visibility rather than chromaticity or semantic statistical constraints. The method solves the ill-posed nature of inverse rendering by decoupling the geometry learning (Stage 1) from the intrinsic component separation (Stage 2), using pseudo-labels derived from the Stage 1 model to guide the decomposition.

### [Diffusion-guided Optimization](13230-diffusion-guided-optimization.md)
Diffusion-guided optimization methods enhance NeRF training by integrating a pretrained 2D diffusion model as a geometric and appearance prior. This approach solves the problem of underconstrained optimization in sparse-view settings or text-to-3D generation, where standard multi-view stereo fails due to lack of input data. Unlike standard NeRF training which relies solely on photometric consistency between observed images, these methods use the diffusion model to penalize renderings that deviate from the learned 2D data distribution, effectively regularizing the 3D solution space.

### [NeRF Training Setup](13271-nerf-training-setup.md)
This setup defines the training configuration for Control-NeRF, a method that replaces the standard positional encoding of 3D coordinates with sampled feature vectors from a volume representation. It solves the problem of enabling direct manipulation of scene content by optimizing a radiance field conditioned on these features rather than absolute spatial positions. It differs from standard NeRF by omitting positional encoding for the input feature vector and using a fixed 64-dimensional feature vector instead of a 60-parameter positionally encoded position.

### [Application-Specific Methods](13294-application-specific-methods.md)
Application-Specific Methods extend the standard NeRF training loop to address domain-specific challenges such as super-resolution, dynamic illumination, synthetic aperture radar (SAR) imaging, and single-view 360° reconstruction. Unlike general-purpose training procedures, these methods modify the optimization objective, scene representation, or input data pipeline to handle constraints like multi-view consistency in super-resolution, disentangling dynamic lighting from geometry, or reconstructing surfaces from sparse radar data.

### [Nerfbusters](13427-nerfbusters.md)
Nerfbusters is a 3D diffusion-based regularization method that improves scene geometry and removes ghostly artifacts (floaters) from Neural Radiance Fields (NeRFs) trained on casually captured images. It distinguishes itself from global 3D priors and 2.5D RGB-D diffusion priors by learning a category-agnostic local 3D geometric prior via a Denoising Diffusion Probabilistic Model (DDPM) and applying it through a novel Density Score Distillation Sampling (DSDS) loss. Unlike SDS, DSDS operates directly on discretized density values without adding noise to the input cube, and unlike visibility-based methods, it regularizes occluded regions by sampling local 3D patches independently of camera viewpoints.

### [Portrait-NeRF](13429-portrait-nerf.md)
Portrait-NeRF is a method for estimating Neural Radiance Fields from a single headshot portrait image. It addresses the fundamental ill-posedness of training a coordinate-based MLP from a single view by pretraining the network with a meta-learning framework on a light stage dataset and subsequently finetuning it on the input image. Unlike 3D model-based methods that often exclude upper head, hair, and torso, or existing single-image methods using point clouds, Portrait-NeRF uses an implicit neural representation to reconstruct the full portrait, including accessories like eyeglasses.

### [Cross-view Self-enhancement](13434-cross-view-self-enhancement.md)
Cross-view Self-enhancement (CSE) is a training strategy for semantic NeRFs that replaces view-inconsistent 2D semantic labels (e.g., from CLIP) with 3D-consistent pseudo-maps rendered from the model itself. It addresses the ambiguity where 2D vision-language models fail to maintain consistent object identities across diverse perspectives, which degrades multi-view consistent rendering. Unlike methods that rely solely on external supervision, CSE leverages the inherent 3D consistency of NeRFs to iteratively refine and synthesize semantic supervision, ensuring that the semantic field learns coherent structures across all views.

### [NeRF-Art](13438-nerf-art.md)
NeRF-Art is a text-guided method for stylizing pre-trained Neural Radiance Fields (NeRFs) by simultaneously altering both appearance and geometry without requiring mesh guidance. It distinguishes itself from prior approaches by employing a hybrid global-local contrastive learning strategy based on CLIP, which ensures strong style adherence and cross-view consistency, addressing the insufficiency of direct CLIP similarity constraints.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, multiple jittered rays are sampled uniformly at random through the spatial footprint of the pixel, rendered with NeRF, and averaged to predict the color in the loss function [S21]. This approach relies on 'oracle' scale information that does not exist for most datasets, performs worse than mip-NeRF, and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 steps to reduce shock from switching regimes [S22, unofficial]. While training from scratch sometimes works, two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### RapNeRF
A NeRF model tailored for view extrapolation, contrasting with standard NeRFs that excel at interpolation [S24]. Training begins with a baseline NeRF to recover a rough mesh. A Ray Atlas (RA) is created by extracting a rough 3D mesh from a pretrained NeRF and mapping training ray directions onto its vertices. Random Ray Casting (RRC) is employed: for a training ray hitting a surface point $v = o + tz d$, a backward ray is cast from $v$ toward a new origin $o'$ with uniformly sampled angular perturbations. RRC and RA augment training rays with a predetermined probability [S24].

### JaxNeRF
A re-implementation of the original NeRF in JAX, used as a benchmark comparison for early works seeking to improve on training and rendering speed [S24]. It is slightly faster and more suited for distributed computing than the original TensorFlow implementation of NeRF [S24].

### SNeS
A method improving geometry by learning probable symmetries for partly symmetric and partly hidden in-scene objects through soft symmetry constraints on geometry and material properties [S24].

### DreamAvatar
A diffusion-based 3D avatar model generation method with text guidance, using an SMPL model as a shape prior and text-guided 2D image generation via diffusion to create training data in a DreamFusion-like 3D generation pipeline combining NeRF and diffusion [S24].

## Operating characteristics

*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot N_{samples} \cdot D)$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), $N_{samples}$ is the number of samples per ray, and $D$ is the MLP depth/width [S5, S11, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot N_{samples} \cdot D)$ for storing intermediate activations during the forward pass, plus $O(N_{total})$ if pre-computing all rays [S5, S14, unofficial]. VRAM usage is a primary constraint, often mitigated by chunking rays [S5, unofficial].
*   **Measured Performance:** In standard Blender scenes, training typically runs for 100,000 iterations [S11, unofficial]. With a batch size of 1024 and 128 samples, this can take several hours on a modern GPU [S17, unofficial].
*   **Scaling:** Performance scales with batch size up to GPU memory limits; however, very large batches can hinder data loading performance and reduce throughput [S15, unofficial]. Smaller batch sizes cap iterations per second [S15, unofficial].
*   **Strengths:** Simple to implement; leverages standard deep learning frameworks; produces high-quality novel views for dense input sets.
*   **Limitations:** Slow convergence (requires many iterations); prone to floaters and artifacts in sparse-view settings; high memory usage for large scenes; sensitive to hyperparameters like learning rate and sampling strategy [S1][S17, unofficial].
*   **When to use:** When high-fidelity novel view synthesis is required for scenes with sufficient multi-view coverage (e.g., Blender, LLFF datasets) [S2, unofficial].
*   **When not to use:** For real-time applications (use [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)); for sparse-view scenarios without additional regularization (use [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)); for dynamic scenes (use [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)).

## Background

The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial][S24]. It established the standard protocol of optimizing a coordinate-based MLP using stochastic gradient descent on photometric loss. Early implementations used TensorFlow [S5, unofficial], later ported to PyTorch [S6, unofficial] and JAX [S15, unofficial][S24]. The method has become the foundational baseline for numerous extensions in 3D vision, including differentiable rendering, neural graphics primitives, and 3D generation [S24].

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
