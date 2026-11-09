# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth observations by updating the parameters of a coordinate-based neural radiance field [S5, S6, S14, unofficial]. It solves the inverse problem of recovering a continuous 3D scene representation from a set of 2D images by leveraging the differentiability of the volume rendering equation [S5, S14, unofficial]. It sets itself apart from evaluation by using stochastic ray sampling, adaptive strategies, and gradient-based updates, whereas evaluation disables these for deterministic results [S2, S4, S9, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics $C$ and intrinsics $K$), near/far bounds, and optional metadata (e.g., background color). The model parameters $\theta$ (weights of the MLPs for density $\sigma$ and color $\mathbf{c}$).
*   **Outputs:** Updated model parameters $\theta$ that minimize the loss function.

**Data Structures:**
*   **Rays:** A batch of rays is typically pre-computed or generated on-the-fly. A common layout for pre-computed rays is a flat array `rays_rgb` of shape $[(N-1) \times H \times W, 3, 3]$ where axis 0 is the flattened pixel index, axis 1 is `[origin, direction, target_rgb]`, and axis 2 contains the XYZ/RGB components [S5, S14, unofficial].
*   **Checkpoint:** A dictionary containing the model state (`state_dict`), optimizer state, and training step/epoch [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S9, S11, unofficial].
    *   Initialize the NeRF model (MLPs for density and color) [S5, unofficial][S8][S11, unofficial].
    *   Initialize the optimizer (typically Adam) [S5, unofficial][S8][S11, S13, unofficial].
    *   Initialize the learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, unofficial][S8][S11, unofficial].
    *   If resuming, load checkpoint and set scheduler last epoch to `start_epoch - 1` [S9, unofficial].

2.  **Ray Sampling (Per Iteration):**
    *   Select a batch of $N_{rand}$ rays [S5, S14, unofficial].
    *   If using pre-computed rays, slice the `rays_rgb` array [S5, S14, unofficial].
    *   If generating on-the-fly, randomly select pixel coordinates from training images, convert to ray origins/directions using camera matrices, and sample ground-truth colors via bilinear interpolation [S4, S14, unofficial].
    *   Apply perturbations to sampling positions if `--perturb` is enabled [S2, unofficial].
    *   Apply precropping if iteration $i <$ `precrop_iters` [S2, S14, unofficial].

3.  **Forward Pass (Rendering):**
    *   For each ray in the batch, sample $K$ points along the ray segment $[near, far]$ [S5, S11, unofficial].
    *   Sample positions $\mathbf{x}_i$ using stratified sampling or uniform sampling [S5, unofficial].
    *   Query the NeRF network $f_\theta(\mathbf{x}_i, \mathbf{d})$ to obtain volume density $\sigma_i$ and color $\mathbf{c}_i$ [S5, S11, unofficial].
    *   Compute transmittance $T_i$ and weights $w_i$ using the emission-absorption model [S5, S11, unofficial].
    *   Composite the final color $\hat{C}(\mathbf{r})$ [S5, S11, unofficial].
    *   If using coarse-fine training, render coarse predictions, sample additional points based on coarse weights, and render fine predictions [S5, S11, unofficial].

4.  **Loss Computation:**
    *   Compute the Mean Squared Error (MSE) between predicted color $\hat{C}$ and ground truth color $C_{gt}$ [S5, S6, S11, S14, unofficial]:
        $$ \mathcal{L}_{rgb} = \frac{1}{N_{batch}} \sum_{j=1}^{N_{batch}} (\hat{C}_j - C_{gt,j})^2 $$
    *   If coarse-fine training is used, add the coarse loss $\mathcal{L}_{coarse}$ to the total loss [S6, S11, S18, unofficial].
    *   Add regularization terms if applicable (e.g., sparsity, TV, Eikonal) [S7, unofficial][S8][S16, unofficial][S23].
    *   Total Loss: $\mathcal{L} = \mathcal{L}_{rgb} + \lambda_{coarse}\mathcal{L}_{coarse} + \mathcal{L}_{reg}$ [S11, S18, unofficial].

5.  **Backpropagation and Update:**
    *   Zero gradients [S18, unofficial].
    *   Compute gradients via backpropagation: $\mathcal{L}.backward()$ [S3, S18, unofficial].
    *   Step the optimizer: `optimizer.step()` [S3, S18, unofficial].
    *   Update learning rate according to the scheduler [S5, unofficial][S8][S18, unofficial].

6.  **Logging and Checkpointing:**
    *   Log loss and PSNR every `i_print` steps [S3, S11, unofficial].
    *   Render validation images every `i_img` or `render_interval` steps [S9, S11, unofficial].
    *   Save checkpoint every `i_weights` or `save_every` steps [S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader

def train_nerf_loop(model, train_dataset, val_dataset, config):
    """
    Main training loop for NeRF.
    
    Args:
        model: Neural Radiance Field model (MLPs).
        train_dataset: Dataset returning (rays_o, rays_d, rgb_gt, near, far).
        val_dataset: Validation dataset.
        config: Dictionary containing hyperparameters.
    """
    # 1. Initialization
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model.to(device)
    
    optimizer = optim.Adam(model.parameters(), lr=config['lr'])
    
    # Scheduler: Exponential Decay or Cosine Annealing
    if config['scheduler'] == 'exponential':
        scheduler = optim.lr_scheduler.ExponentialLR(optimizer, gamma=config['decay_rate'])
    elif config['scheduler'] == 'cosine':
        scheduler = optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=config['n_iters'])
        
    loss_fn = nn.MSELoss()
    
    # DataLoader for batching
    dataloader = DataLoader(train_dataset, batch_size=config['batch_size'], shuffle=True)
    
    # Checkpointing setup
    start_epoch = 0
    if config.get('checkpoint_path') and os.path.exists(config['checkpoint_path']):
        checkpoint = torch.load(config['checkpoint_path'])
        model.load_state_dict(checkpoint['model'])
        optimizer.load_state_dict(checkpoint['optimizer'])
        start_epoch = checkpoint['epoch']
        scheduler.last_epoch = start_epoch - 1 # Adjust for resumption [S9, unofficial]

    # 2. Training Loop
    for epoch in range(start_epoch, config['num_epochs']):
        model.train()
        total_loss = 0.0
        
        for batch_idx, batch in enumerate(dataloader):
            # Unpack batch: rays_o, rays_d, rgb_gt, near, far
            rays_o, rays_d, rgb_gt, near, far = batch
            rays_o = rays_o.to(device)
            rays_d = rays_d.to(device)
            rgb_gt = rgb_gt.to(device)
            near = near.to(device)
            far = far.to(device)
            
            # Zero gradients
            optimizer.zero_grad()
            
            # 3. Forward Pass
            # Render rays using the model's render function
            # This function handles ray marching, network queries, and volume integration
            render_out = model.render(rays_o, rays_d, near, far, config)
            
            # Extract predicted RGB
            rgb_pred = render_out['rgb']
            
            # Handle coarse-fine training if applicable
            loss = loss_fn(rgb_pred, rgb_gt)
            if 'rgb_coarse' in render_out:
                loss += loss_fn(render_out['rgb_coarse'], rgb_gt)
                
            # Add regularization if defined in model/config
            if 'reg_loss' in render_out:
                loss += config.get('reg_weight', 0.0) * render_out['reg_loss']
            
            # 4. Backpropagation
            loss.backward()
            optimizer.step()
            
            # 5. Logging
            if batch_idx % config['log_interval'] == 0:
                psnr = -10.0 * torch.log10(loss).item()
                print(f"Epoch {epoch}, Batch {batch_idx}, Loss: {loss.item():.6f}, PSNR: {psnr:.2f}")
            
            total_loss += loss.item()
            
        # Update scheduler
        scheduler.step()
        
        # 6. Validation and Checkpointing
        if epoch % config['val_interval'] == 0:
            validate(model, val_dataset, device, epoch)
            
        if epoch % config['save_interval'] == 0:
            save_checkpoint(model, optimizer, epoch, config['checkpoint_dir'])

def validate(model, val_dataset, device, epoch):
    """Render validation images and compute metrics."""
    model.eval()
    with torch.no_grad():
        # Sample a subset of rays or full validation set
        # Render and compute PSNR/SSIM
        pass

def save_checkpoint(model, optimizer, epoch, dir_path):
    """Save model and optimizer state."""
    checkpoint = {
        'model': model.state_dict(),
        'optimizer': optimizer.state_dict(),
        'epoch': epoch
    }
    torch.save(checkpoint, os.path.join(dir_path, f'epoch_{epoch}.pth'))
```

**Parameters:**
*   `lr`: Learning rate for Adam. Typical value: $5 \times 10^{-4}$ [S11, unofficial].
*   `batch_size` ($N_{rand}$): Number of rays sampled per iteration. Typical value: 1024 or 2048 [S5, S11, unofficial].
*   `n_iters`: Total number of gradient steps. Default: 100,000 [S11, unofficial].
*   `decay_rate`: Factor for exponential decay. Default: 0.1 [S6, S18, unofficial].
*   `decay_steps`: Steps for decay. Calculated as `args.lrate_decay * 1000` [S5, S18, unofficial].
*   `chunk`: Maximum rays processed per forward pass to limit VRAM [S5, unofficial].
*   `perturb`: Jitter amount for sampling. 0. for no jitter, 1. for jitter [S2, unofficial].
*   `raw_noise_std`: Standard deviation of noise added to $\sigma_a$ output. Default: 0., 1e0 recommended [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Mitigated by chunking ray batches [S5, unofficial].
*   **Instability in Growing Networks:** Direct growth of randomly initialized sub-networks causes density to drop to 0. Initialize alpha linear weights of grown sub-networks to match the parent [S1].
*   **Gradient Computation Order:** Higher-order derivatives (for logging gradients/laplacians) must be computed *before* `loss.backward()` to avoid freeing the autograd graph [S3, unofficial].
*   **Checkpoint Resumption:** When resuming, the scheduler's last epoch must be set to `start_epoch - 1` to ensure correct scheduling [S9, unofficial].
*   **Empty Checkpoints:** If a checkpoint is loaded but the step is 0, it is considered empty and should trigger an error [S15, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images from a neural radiance field (NeRF) during optimization. It solves the problem of computing gradients for scene parameters by sampling points along camera rays, querying a neural network for volume density and color, and integrating them via alpha compositing. It sets itself apart from evaluation by using stochastic perturbations, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the hyperparameters, optimization routines, and architectural constraints required to stabilize and accelerate the differentiable volume rendering optimization process. It addresses the sensitivity of radiance field training to initialization, sampling density, and activation function choices. Unlike general optimization loops, it specifies implementation-specific details such as learning rate schedules, loss weights, and batch sampling strategies that are critical for convergence in methods like mip-NeRF, Instant-NGP, and NeuralVDB.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Camera Pose and Parameter Optimization refers to the family of methods that jointly optimize the parameters of a Neural Radiance Field (NeRF) and the camera poses (extrinsics) and/or intrinsics (e.g., focal length) from a set of input images. It solves the problem of reconstructing 3D scenes when camera parameters are unknown, inaccurate, or when the system requires online pose estimation for navigation. Unlike baseline NeRF, which relies on pre-estimated poses from Structure-from-Motion (SfM) pipelines like COLMAP, these methods treat camera parameters as learnable variables updated via gradient descent alongside the NeRF weights. This family encompasses static scene reconstruction with unknown poses, hybrid 2D/3D registration, and online robot navigation filters.

### [Specific Model Architectures](8585-specific-model-architectures.md)
This family encompasses specific neural network architectures and structural modifications applied to the Neural Radiance Field (NeRF) representation to address limitations in rendering speed, geometric accuracy, generalization, and semantic understanding. While the parent [NeRF Training Loop](970-nerf-training-loop.md) defines the optimization procedure, these architectures define the function $f_\theta(\mathbf{x}, \mathbf{d})$ itself. They are distinguished from siblings like [AdaNeRF](6195-adanerf.md) (adaptive sampling) and [Zip-NeRF](9538-zip-nerf-training-procedure.md) (hash grids) by their focus on the internal network topology, input/output mappings, and auxiliary modules (e.g., discriminators, diffusion priors, or geometric heads) rather than just the sampling strategy or data structure.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
This family encompasses the mathematical terms added to the standard photometric reconstruction loss to constrain the optimization of Neural Radiance Fields (NeRFs), addressing issues such as under-constrained geometry, floaters, and color artifacts in sparse or few-view settings. It distinguishes itself from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the objective function's structure rather than the sampling strategy, and from [Training Configuration](2332-training-configuration.md) by defining the specific gradients that drive parameter updates.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Progressive Inpainting-and-Erasing (PERF) is a training strategy for single-view NeRF that avoids geometry conflicts between visible and invisible regions by computing a mask of conflicted regions and eliminating them from NeRF training. It sets itself apart from standard single-view NeRF methods by integrating a collaborative RGBD inpainting step with a progressive erasing mechanism to handle the ambiguity of unseen geometry, building on the [NeRF Training Loop](970-nerf-training-loop.md) by dynamically expanding the reference view set.

### [KNN-NeRF](8672-knn-nerf.md)
KNN-NeRF is a baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Progressive and Dynamic Scene Growth addresses the challenge of reconstructing large-scale or complex 3D scenes from limited input data by incrementally expanding the neural representation's capacity or the training dataset. Unlike static NeRF architectures that require pre-defining the entire hierarchy or fixed network size, these methods adaptively grow the model structure or add synthetic reference views during training to resolve occlusions and capture high-frequency details. This family distinguishes itself from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) (which focuses on ray selection) and [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md) (which relies on external priors) by actively modifying the model's topology or the set of supervision signals to improve reconstruction quality in complex regions.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Adaptive ray sampling refers to a family of optimization procedures in neural radiance field (NeRF) training that dynamically determine the spatial distribution of sample points along viewing rays to improve reconstruction quality and training efficiency. Unlike uniform sampling, which treats all regions of a ray equally, these methods allocate computational resources to areas of high geometric or photometric complexity, such as object surfaces or textured regions. This family is distinguished from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by its focus on the *selection* of sample locations based on learned or heuristic priors, rather than the integration technique itself.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
This family addresses the degradation of NeRF quality—such as blurred geometry, slow convergence, and floaters—when trained on sparse input views (typically 15–20 images or fewer than 50–100 views). It distinguishes itself from standard NeRF training by decoupling the initialization and regularization processes from the default random or empty-space assumptions, instead leveraging external geometric scaffolds, semantic masks, or visibility priors to anchor the optimization.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Scene Representation and Partitioning addresses the scalability limits of global NeRF models, which suffer from high memory consumption, slow training convergence, and rendering artifacts when applied to large-scale environments or dynamic scenes. By decomposing the 3D spatial domain into disjoint regions (blocks, tiles, or cells), each managed by a dedicated local neural network or data structure, these methods enable distributed training, faster inference, and the handling of scenes spanning thousands of square meters. Unlike global approaches, partitioning allows for independent optimization of local regions, though it introduces challenges regarding boundary continuity and temporal consistency in dynamic settings.

### [NerfingMVS](9518-nerfingmvs.md)
NerfingMVS is a multi-view depth estimation method that optimizes Neural Radiance Fields (NeRF) directly over implicit volumes, guided by learning-based depth priors derived from sparse Structure-from-Motion and Multi-View Stereo (SfM+MVS) reconstructions. It addresses the shape-radiance ambiguity inherent in NeRF for indoor scenes by eliminating the need for accurate pixel matching across views, instead using external depth estimates to guide the sampling and optimization process. Unlike standard NeRF which samples uniformly or stratified along rays, NerfingMVS incorporates depth priors and their associated confidences into the rendering pipeline to bias sampling towards geometrically plausible regions.

### [Zip-NeRF Training Procedure](9538-zip-nerf-training-procedure.md)
The Zip-NeRF Training Procedure is a two-stage optimization method for inverse rendering that jointly trains a physically-based scene model, environment map, and radiance caches (Zip-NeRF and Fast Cache) to estimate incoming radiance. It solves the bias introduced by approximating incoming radiance using neural caches by employing a control variate strategy and a specific gradient trick for the photometric loss. It sets itself apart from standard NeRF training by decoupling the cache initialization from the joint optimization of the physical model and by using an occlusion-aware importance sampler to reduce variance in the radiance estimation.

### [NeRV Training](10253-nerv-training.md)
NeRV training optimizes a continuous volumetric representation that decouples scene geometry (shape) from material appearance (reflectance) and lighting interactions (visibility) to enable efficient relighting and view synthesis. It differs from standard NeRF training by introducing a separate "visibility MLP" that predicts ray visibility and termination depth, supervised by consistency terms with the primary shape MLP rather than ground-truth labels, allowing the simulation of direct and one-bounce indirect illumination within the training loop.

### [NeRV Rendering](10264-nerv-rendering.md)
NeRV rendering computes the pixel color for a camera ray by integrating reflected radiance along the ray, where the radiance at each sample point is derived from a physically-based shading model that accounts for direct and indirect illumination. It differs from standard NeRF volume rendering by explicitly modeling surface normals and microfacet BRDF parameters via separate MLPs, and by estimating indirect lighting through secondary visibility queries rather than simple volumetric absorption.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
This family defines specific neural network architectures and configuration parameters for the density and texture (color) networks within Neural Scene Vector Fields (NSVF) and related radiance field models. It addresses the problem of how to structure the input features (position, direction, embeddings) and the network topology to optimize reconstruction quality and training stability. It sets itself apart from general training loops by focusing on the internal MLP structure and feature concatenation strategies rather than the optimization schedule or ray sampling strategy.

### [Aug-NeRF](10294-aug-nerf.md)
Aug-NeRF is a training framework for Neural Radiance Fields (NeRF) that injects worst-case adversarial perturbations into three distinct levels of the pipeline—input coordinates, intermediate features, and pre-rendering output—to implicitly regularize the model with physical grounds. It addresses the generalization gap in NeRF where interpolation between seen views yields inconsistent and non-smooth geometries. Unlike explicit regularizers like Laplacian or Total Variation which constrain MLP flexibility aggressively, Aug-NeRF uses data-adaptive smoothness priors to enhance generalization to unseen views without sacrificing representation flexibility.

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
Cross-Guided Optimization of Radiance Fields (CROP) is a framework that performs multi-view image super-resolution (MVSR) on training-view images during the radiance field optimization process to improve high-resolution novel view synthesis (HRNVS). It fuses single-image super-resolution (SISR) features with voxel-based uncertainty fields to ensure multi-view consistency and high-frequency details. Unlike methods that rely solely on the spectral characteristics of coordinate-based networks for upscaling, CROP explicitly bridges 2D SISR and 5D radiance fields by iteratively refining input images based on rendering errors.

### [Application-Specific Pipelines](11200-application-specific-pipelines.md)
Application-Specific Pipelines are specialized training and optimization procedures designed to address distinct downstream tasks—such as rapid reconstruction, pose estimation, or scene editing—rather than generic radiance field optimization. They diverge from standard NeRF training by integrating external priors (e.g., multi-view stereo data, diffusion models, or dynamics models) and modifying the loss landscape or rendering loop to suit specific constraints like speed, geometric accuracy, or semantic control.

### [Superpixel-guide Efficient Sampling (SES)](11252-superpixel-guide-efficient-sampling-ses.md)
Superpixel-guide Efficient Sampling (SES) is a training strategy for neural radiance fields that uses SLIC superpixels as the smallest processing units to guide ray sampling, addressing slow convergence and scattered sampling in high-resolution NeRF training by focusing computation on complex regions. Unlike quadtrees which cause redundant computations and fragmented information due to fixed shapes, superpixels resolve inadequate partitioning issues and reduce computational overhead per instance.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. The full-resolution supersampling baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot K)$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), and $K$ is the number of samples per ray [S5, S11, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot K)$ for storing intermediate rendering states per batch, plus model memory [S5, unofficial]. VRAM overflow is a common pitfall, mitigated by chunking [S5, unofficial].
*   **Measured Performance:** Training typically requires 100,000 iterations [S11, unofficial]. For a 199-image dataset, 5 epochs with batch size 1024 results in ~39,000 iterations [S17, unofficial].
*   **Scaling:** Larger batch sizes (e.g., $2^{14}$) hinder data loading performance and prevent full GPU occupancy, while smaller batch sizes (e.g., $2^{10}$) cap iterations per second [S15, unofficial].
*   **Strengths:** End-to-end differentiable, handles complex geometry and appearance, produces high-quality novel views.
*   **Limitations:** Slow training convergence, high memory usage, prone to floaters and artifacts in sparse views [S1][S17, unofficial].

## Background
The NeRF Training Loop was introduced in the seminal paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" by Mildenhall et al. (2020) [S5, S18, unofficial]. It builds upon the concept of coordinate-based neural networks for implicit representations [S3, unofficial]. The method has since been extended with various optimization strategies, including coarse-fine training [S5, unofficial], adaptive sampling [S22, unofficial], and different rendering models [S19, unofficial][S23]. It is widely used in computer vision and graphics for novel view synthesis, 3D reconstruction, and scene understanding [S1][S18, unofficial].

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
