# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth training views by updating the parameters of a continuous volumetric representation [S5, S11, S14, unofficial]. It solves the problem of learning a scene's geometry and appearance from unposed or posed 2D images by differentiating the volume rendering equation with respect to the neural network weights [S4, S14, unofficial]. It sets itself apart from evaluation rendering by using stochastic ray sampling and perturbation to facilitate gradient-based optimization, whereas evaluation uses deterministic rendering for reproducible metrics [S2, S9, unofficial].

## How it works
The training loop optimizes a coordinate-based neural network (MLP or SIREN) to map 3D spatial coordinates and viewing directions to volume density ($\sigma$) and view-dependent color ($\mathbf{c}$) [S3, S5, unofficial]. The core mechanism involves sampling rays from training images, rendering them via differentiable volume integration, computing a loss against ground-truth pixels, and updating weights via gradient descent [S5, S6, S14, unofficial].

### Inputs and Outputs
- **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics $C2W$ and intrinsics $K$) [S5, S11, unofficial].
- **Outputs:** A trained neural network model (state dictionary) and optimizer state [S11, S14, unofficial].

### Data Structures
- **Rays Array (`rays_rgb`):** A pre-computed or on-the-fly generated array containing ray origins, directions, and target RGB colors.
    - Layout: Shape $[(N-1) \times H \times W, 3, 3]$ or $[3, B, 3]$ for batches [S5, S14, unofficial].
    - Axis 0 (or 2 in batch): Flattened pixel index or batch index.
    - Axis 1 (or 0 in batch): Channel type: 0=Ray Origin, 1=Ray Direction, 2=Target RGB [S5, S14, unofficial].
    - Axis 2 (or 1 in batch): XYZ/RGB components [S5, S14, unofficial].
    - Type: `float32` [S5, S14, unofficial].
- **Checkpoint:** A dictionary storing model weights, optimizer state, and training step/epoch [S11, S14, unofficial].
    - Keys: `'step'` (int), `'coarse'` (state_dict), `'fine'` (state_dict), `'optim'` (optimizer state_dict) [S11, unofficial].

### Numbered Steps
1. **Initialization:**
    - Load training and validation datasets [S9, S11, unofficial].
    - Initialize the NeRF model (coarse and fine MLPs if using two-stage rendering) [S11, unofficial].
    - Initialize the optimizer (typically Adam) and learning rate scheduler [S5, S11, unofficial].
    - Precompute ray bundles or set up data loaders [S5, S11, S17, unofficial].

2. **Iteration Loop (for $i = 1$ to $N_{iters}$):**
    a. **Ray Sampling:**
        - Randomly select $N_{rand}$ pixel coordinates from the training images [S5, S14, unofficial].
        - Construct ray bundles (origins, directions) using camera parameters [S4, S5, unofficial].
        - Gather target RGB colors for these pixels [S5, S14, unofficial].
        - *Note:* If `use_batching` is true, slice from pre-computed `rays_rgb`; otherwise, sample dynamically [S5, S14, unofficial].
        - *Perturbation:* If enabled, jitter sample positions along rays to reduce aliasing [S2, S5, unofficial].

    b. **Forward Pass (Rendering):**
        - Query the NeRF model with sampled points $(\mathbf{x}, \mathbf{d})$ to get raw outputs (density $\sigma$, color $\mathbf{c}$) [S5, S6, unofficial].
        - Perform volume rendering (alpha compositing) to synthesize predicted RGB $\hat{C}$ and depth [S5, S6, unofficial].
        - If using coarse-to-fine rendering, render coarse rays, then sample additional points based on coarse weights for fine rendering [S5, S11, unofficial].

    c. **Loss Computation:**
        - Compute Mean Squared Error (MSE) between predicted RGB $\hat{C}$ and target RGB $C_{gt}$ [S5, S6, S14, unofficial].
        - Formula: $\mathcal{L}_{rgb} = \frac{1}{N} \sum_{i=1}^N (\hat{C}_i - C_{gt,i})^2$ [S14, S17, unofficial].
        - If coarse output $\hat{C}_0$ is available, add coarse loss: $\mathcal{L}_{coarse} = \text{MSE}(\hat{C}_0, C_{gt})$ [S6, S11, unofficial].
        - Total Loss: $\mathcal{L} = \mathcal{L}_{rgb} + \lambda_{coarse} \mathcal{L}_{coarse} + \mathcal{L}_{reg}$ [S11, S14, unofficial].
        - *Note:* $\mathcal{L}_{reg}$ may include sparsity, total variation, or eikonal terms depending on the variant [S7, unofficial][S8].

    d. **Backpropagation and Update:**
        - Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
        - Compute gradients: `loss.backward()` [S18, unofficial].
        - Update weights: `optimizer.step()` [S18, unofficial].
        - *Pitfall:* Higher-order derivative logging (e.g., gradients, Laplacians) must occur *before* `loss.backward()` as the autograd graph is freed [S3, unofficial].

    e. **Learning Rate Scheduling:**
        - Update learning rate based on the scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
        - Exponential Decay: $lr_{new} = lr_{old} \times \text{decay\_rate}^{(step / \text{decay\_steps})}$ [S5, S18, unofficial].

    f. **Logging and Checkpointing:**
        - Log loss and PSNR every $I_{print}$ steps [S2, S14, unofficial].
        - Render validation images every $I_{img}$ steps [S2, S11, unofficial].
        - Save checkpoint every $I_{weights}$ steps [S2, S11, unofficial].

### Formulas
- **MSE Loss:**
  $$ \mathcal{L}_{mse} = \frac{1}{N} \sum_{i=1}^N (\hat{C}_i - C_{gt,i})^2 $$
  [S14, S17, unofficial]

- **PSNR from MSE:**
  $$ \text{PSNR} = -10 \log_{10}(\mathcal{L}_{mse}) $$
  [S14, unofficial]

- **Exponential Decay:**
  $$ lr(t) = lr_0 \cdot \text{decay\_rate}^{\lfloor t / \text{decay\_steps} \rfloor} $$
  [S5, S18, unofficial]

- **Positional Encoding (Standard):**
  $$ \gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), \dots, \sin(2^{L-1} \pi p), \cos(2^{L-1} \pi p)) $$
  [S20]

### Parameters
- **$N_{rand}$ (Batch Size):** Number of rays sampled per iteration. Typical values: 1024 or 2048 [S5, S11, S14, unofficial].
- **Learning Rate ($lr$):** Adam optimizer learning rate. Typical value: $5 \times 10^{-4}$ [S11, unofficial].
- **Decay Rate:** Factor for exponential decay. Default: 0.1 [S6, S18, unofficial].
- **Decay Steps:** Number of steps between decays. Calculated as `args.lrate_decay * 1000` [S5, S18, unofficial].
- **$N_{iters}$:** Total number of gradient steps. Default: 100,000 [S11, unofficial].
- **Chunk Size:** Maximum rays processed per forward pass to limit VRAM. Passed to render function [S5, unofficial].
- **Perturbation:** Jitter amount for sampling. 0.0 for no jitter, 1.0 for full jitter [S2, S5, unofficial].
- **Raw Noise Std:** Standard deviation of noise added to density output for regularization. Default: 0.0, recommended: $10^{-4}$ [S2, S5, unofficial].

### Edge Cases and Pitfalls
- **VRAM Overflow:** Large batch sizes or high-resolution images can exceed GPU memory. Mitigated by chunking ray batches during rendering [S5, unofficial].
- **Instability in Growth:** In dynamic growing networks (e.g., Recursive-NeRF), randomly initialized sub-networks may collapse to zero density. Solution: Initialize alpha linear weights of grown sub-networks to match the parent [S1].
- **Checkpoint Resume:** When resuming, the scheduler's last epoch must be set to `start_epoch - 1` to maintain correct scheduling [S9, unofficial]. Empty checkpoints (step 0) should be detected and handled [S15, unofficial].
- **Precrop:** For early training, restricting sampling to a central crop (e.g., 50% of image) can stabilize optimization [S2, S14, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_loop(model, optimizer, scheduler, train_dataset, config):
    """
    Main training loop for NeRF.
    
    Args:
        model: Neural Radiance Field model (MLP).
        optimizer: Optimizer (e.g., Adam).
        scheduler: Learning rate scheduler.
        train_dataset: Dataset containing images, poses, and intrinsics.
        config: Configuration object with hyperparameters.
    """
    n_iters = config.n_iters  # e.g., 100,000
    n_rand = config.n_rand    # e.g., 1024
    decay_rate = config.decay_rate  # e.g., 0.1
    decay_steps = config.decay_steps # e.g., 1000 * lrate_decay
    
    # Precompute rays if batching is enabled
    if config.use_batching:
        rays_rgb = precompute_rays(train_dataset) # Shape: [N_pixels, 3, 3]
        i_batch = 0
        shuffle_rays = True
    else:
        rays_rgb = None

    for step in range(1, n_iters + 1):
        # 1. Ray Sampling
        if config.use_batching:
            if i_batch + n_rand >= rays_rgb.shape[0]:
                if shuffle_rays:
                    np.random.shuffle(rays_rgb)
                    shuffle_rays = False
                i_batch = 0
            
            # Slice batch: [n_rand, 3, 3] -> Transpose to [3, n_rand, 3]
            batch = rays_rgb[i_batch : i_batch + n_rand].transpose(1, 0, 2)
            rays_o_d = batch[:2] # Origins and Directions
            target_rgb = batch[2] # Target Colors
            i_batch += n_rand
        else:
            # Dynamic sampling
            img_idx = np.random.randint(0, len(train_dataset))
            rays_o_d, target_rgb = sample_rays_from_image(train_dataset[img_idx], n_rand)

        # 2. Forward Pass
        # Render rays using the model
        # Returns: rgb (predicted color), extras (raw outputs, coarse rgb, etc.)
        rgb, extras = model.render(rays_o_d, config=config)

        # 3. Loss Computation
        # Primary RGB Loss
        loss_rgb = torch.mean((rgb - target_rgb) ** 2)
        loss = loss_rgb

        # Coarse Loss (if two-stage rendering is used)
        if 'rgb0' in extras:
            loss_coarse = torch.mean((extras['rgb0'] - target_rgb) ** 2)
            loss += loss_coarse

        # Regularization Losses (Optional, e.g., TV, Eikonal)
        # loss += config.tv_loss_weight * compute_tv_loss(model)
        # loss += config.eikonal_weight * compute_eikonal_loss(model)

        # 4. Backpropagation
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

        # 5. Learning Rate Scheduling
        # Example: Exponential Decay
        current_lr = config.lr * (decay_rate ** (step / decay_steps))
        for param_group in optimizer.param_groups:
            param_group['lr'] = current_lr
        # Or use scheduler.step() if using PyTorch schedulers

        # 6. Logging and Checkpointing
        if step % config.i_print == 0:
            psnr = -10 * torch.log10(loss_rgb)
            print(f"Step {step}: Loss {loss.item():.6f}, PSNR {psnr.item():.2f}")

        if step % config.i_img == 0:
            render_validation_images(model, train_dataset, step)

        if step % config.i_weights == 0:
            save_checkpoint(model, optimizer, step, config.checkpoint_dir)
```

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images and depth maps from a Neural Radiance Field (NeRF) during optimization. It sets itself apart from evaluation rendering by using stochastic sampling with perturbation and adaptive step sizes to facilitate gradient-based optimization, whereas evaluation uses deterministic rendering for reproducible metrics.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the static and dynamic hyperparameters, optimization schedules, and environment settings required to converge a differentiable volume rendering model. It solves the problem of balancing convergence speed, reconstruction fidelity, and computational stability by specifying optimizer states, learning rate decay strategies, batch sampling rates, and regularization weights. It sets itself apart from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the gradient descent mechanics rather than the forward rendering pass, and from [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md) by defining the temporal evolution of the optimization state rather than the initial spatial representation.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Regularization and Loss Functions define the objective criteria used to optimize Neural Radiance Fields (NeRF) beyond standard photometric reconstruction. While standard NeRF training minimizes the Mean Squared Error (MSE) between rendered and ground-truth pixel colors, this approach often leads to degenerate solutions such as "floaters" (spurious geometry in empty space), overfitting in sparse-view scenarios, or semantic incoherence. This family introduces auxiliary terms—such as geometric smoothness, diffusion priors, semantic consistency, and structural similarity—to constrain the density and color fields, ensuring physically plausible geometry and high-fidelity appearance. It sets itself apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the *objective function* rather than the *sampling strategy*, and from [Training Configuration](2332-training-configuration.md) by providing the mathematical definitions of the penalties applied during optimization.

### [KNN-NeRF](8672-knn-nerf.md)
KNN-NeRF is a baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Progressive and Dynamic Scene Growth refers to a family of training strategies that expand the capacity, data, or topology of a neural radiance field during optimization, rather than optimizing a static representation on a fixed dataset. This family solves the problems of representing complex, large-scale, or dynamic scenes with limited initial data, and mitigates issues such as "foggy" densities, floating artifacts, and catastrophic forgetting in online settings. It sets itself apart from static NeRF variants by coupling the network architecture or training data directly to the training error or temporal sequence, growing $k$ new branches per step based on uncertainty, iteratively expanding reference views for panoramic scenes [8666-progressive-view-synthesis-training.md], or continuously updating the field as new camera views arrive [12066-online-incremental-nerf-optimization.md].

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Adaptive ray sampling refers to a family of training procedures in neural radiance fields (NeRF) that replace uniform ray sampling with strategies that concentrate computational resources on regions of high geometric or photometric complexity. It solves the inefficiency of uniform sampling, which wastes evaluations on empty space or smooth regions, by dynamically adjusting sample locations or ray selection based on density, error, depth priors, or spatial partitions. This family is distinguished from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on *where* to sample rather than *how* to integrate, and from [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) by encompassing broader strategies including error-based mining, depth-guided selection, and superpixel partitioning, not just hierarchical point clustering.

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

### [Cross-view Self-enhancement](13434-cross-view-self-enhancement.md)
Cross-view Self-enhancement (CSE) is a training strategy for semantic NeRFs that replaces view-inconsistent 2D semantic labels (e.g., from CLIP) with 3D-consistent pseudo-maps rendered from the model itself. It addresses the ambiguity where 2D vision-language models fail to maintain consistent object identities across diverse perspectives, which degrades multi-view consistent rendering. Unlike methods that rely solely on external supervision, CSE leverages the inherent 3D consistency of NeRFs to iteratively refine and synthesize semantic supervision, ensuring that the semantic field learns coherent structures across all views.

### [NeRF-Art](13438-nerf-art.md)
NeRF-Art is a text-guided method for stylizing pre-trained Neural Radiance Fields (NeRFs) by simultaneously altering both appearance and geometry without requiring mesh guidance. It distinguishes itself from prior approaches by employing a hybrid global-local contrastive learning strategy based on CLIP, which ensures strong style adherence and cross-view consistency, addressing the insufficiency of direct CLIP similarity constraints.

### [Neural Implicit Surfaces with PBR](13481-neural-implicit-surfaces-with-pbr.md)
This method integrates a Signed Distance Function (SDF) with a Physically-Based Rendering (PBR) material model to achieve accurate scene representation, explicitly separating specular and diffuse reflections. It differs from standard NeRF by deriving normals from the SDF gradient rather than direct optimization, and from basic SDF methods by employing a precomputed BRDF LUT for specular calculation. It builds on [NeRF Training Loop](970-nerf-training-loop.md) but replaces the generic radiance field with a geometry-aware implicit surface representation.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. This approach relies on 'oracle' scale information that does not exist for most datasets [S21]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### JaxNeRF
A re-implementation of the original NeRF in JAX, used as a benchmark comparison for early works seeking to improve on training and rendering speed [S24]. It is slightly faster and more suited for distributed computing than the original TensorFlow implementation of NeRF [S24].

### DreamAvatar
A diffusion-based 3D avatar model generation method with text guidance, using an SMPL model as a shape prior and text-guided 2D image generation via diffusion to create training data in a DreamFusion-like 3D generation pipeline combining NeRF and diffusion [S24].

## Operating characteristics
- **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot N_{samples})$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the number of rays per batch, and $N_{samples}$ is the number of samples per ray [S5, S14, unofficial].
- **Memory Complexity:** $O(N_{rand} \cdot N_{samples} \cdot C)$, where $C$ is the number of channels (RGB + Depth + Extras) [S5, unofficial]. VRAM usage is mitigated by chunking ray batches [S5, unofficial].
- **Measured Performance:** Training on 128x128 images typically takes 250 epochs or 100,000 iterations [S4, S11, unofficial]. PSNR increases from ~10 dB to ~19 dB over 20 epochs in some implementations [S12].
- **Scaling:** Larger batch sizes ($2^{14}$) hinder data loading performance and prevent full GPU occupancy, while smaller batch sizes ($2^{10}$) cap iterations per second [S15, unofficial]. $2^{10}$ is often chosen as a trade-off [S15, unofficial].
- **Strengths:** Simple to implement, differentiable end-to-end, produces high-quality novel views [S5, S14, unofficial].
- **Limitations:** Slow convergence, prone to floaters in empty space, requires many views for high fidelity [S1][S17, unofficial].
- **When to Use:** Standard novel view synthesis with dense input views [S5, unofficial].
- **When Not to Use:** Sparse view scenarios without regularization, real-time applications without acceleration [S1][S17, unofficial].

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial][S24]. It builds on the concept of coordinate-based neural networks for implicit representations [S3, unofficial][S20]. The method was initially implemented in TensorFlow [S5, unofficial] and later ported to PyTorch [S6, S11, unofficial] and JAX [S15, unofficial][S24]. It has become the foundational training procedure for numerous NeRF variants and 3D deep learning applications [S24].

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
