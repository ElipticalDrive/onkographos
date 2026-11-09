# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth observations by updating the parameters of a coordinate-based neural network [S5, S6, S14, unofficial]. It solves the inverse problem of recovering a continuous volumetric scene representation from a set of 2D images by leveraging the differentiability of the volume rendering equation [S5, S14, unofficial]. It sets itself apart from evaluation by using stochastic ray sampling, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results [S2, S4, S9, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics/intrinsics) and pixel coordinates; a neural network model $f_\theta$ (typically an MLP) parameterized by $\theta$; optimization hyperparameters (learning rate, batch size, iterations).
*   **Outputs:** Optimized network weights $\theta^*$; optionally, a trained radiance field capable of rendering novel views.

**Data Structures:**
*   **Rays:** A batch of rays is typically pre-computed or generated on-the-fly. A common layout for pre-computed rays is a numpy array `rays_rgb` of shape $[(N-1) \times H \times W, 3, 3]$ [S5, S14, unofficial].
    *   Axis 0: Flattened pixel index.
    *   Axis 1: Channel type (0: ray origin, 1: ray direction, 2: target RGB color).
    *   Axis 2: XYZ/RGB components.
    *   Data type: `float32` [S5, S14, unofficial].
*   **Batch:** During training, a subset of rays is selected. The batch tensor is often transposed to shape $[3, B, 3]$ where axis 0 is channel type (origin, direction, target), axis 1 is batch index, and axis 2 is component [S14, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S9, S11, unofficial].
    *   Initialize the neural network model $f_\theta$. For standard NeRF, this is an MLP mapping 3D position and viewing direction to color and density [S5, S6, unofficial].
    *   Initialize the optimizer (typically Adam) with a learning rate $lr$ [S5, S11, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Precompute ray bundles for all training pixels if using batching [S5, S14, unofficial].

2.  **Iteration Loop:** For iteration $i$ from 1 to $N_{iters}$:
    a.  **Ray Sampling:**
        *   Select a batch of $N_{rand}$ rays from the pre-computed `rays_rgb` array or generate random pixel coordinates from training images [S5, S14, unofficial].
        *   If using batching, slice `rays_rgb[i_batch : i_batch + N_{rand}]`, transpose to $[3, N_{rand}, 3]$, and split into `batch_rays` (origins/directions) and `target_s` (colors) [S5, S14, unofficial].
        *   Increment `i_batch`. If `i_batch` exceeds the total number of rays, shuffle `rays_rgb` and reset `i_batch` to 0 [S5, S11, S14, unofficial].
        *   *Edge Case:* If `use_batching` is False, randomly select one training image, generate all rays for that image, optionally apply a central crop (precrop) for the first $P_{precrop}$ iterations, and sample $N_{rand}$ unique pixel coordinates [S14, unofficial].
        *   *Perturbation:* If enabled, jitter the sample positions along the ray to break symmetry and improve high-frequency reconstruction [S2, unofficial].

    b.  **Forward Pass (Rendering):**
        *   Sample points along each ray in `batch_rays` using stratified sampling between near and far bounds [S5, S6, unofficial].
        *   Query the neural network $f_\theta$ at these points to obtain volume density $\sigma$ and color $\mathbf{c}$ [S5, S6, unofficial].
        *   Compute the rendered color $\hat{C}$ using the emission-absorption model (DVR) [S5, S6, S14, unofficial]:
            $$
            \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
            $$
            where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [S1][S5, S6, S14, unofficial].
        *   *Coarse/Fine:* If using a coarse-to-fine strategy, render a coarse image, sample additional points based on weights, and render a fine image [S5, S11, unofficial].

    c.  **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between the predicted RGB $\hat{C}$ and the ground truth target colors $C_{gt}$ [S5, S6, S14, unofficial]:
            $$
            \mathcal{L}_{MSE} = \frac{1}{N_{pixels}} \sum_{p} (\hat{C}_p - C_{gt, p})^2
            $$
            [S14, S17, unofficial].
        *   *Coarse Loss:* If a coarse network output $\hat{C}_0$ exists, compute its MSE loss and add it to the total loss [S6, S11, S18, unofficial]:
            $$
            \mathcal{L}_{total} = \mathcal{L}_{MSE}(\hat{C}, C_{gt}) + \lambda_{coarse} \mathcal{L}_{MSE}(\hat{C}_0, C_{gt})
            $$
            [S11, S18, unofficial].
        *   *Regularization:* Add other loss terms if applicable (e.g., Eikonal loss, sparsity loss) [S7, unofficial][S8].

    d.  **Backpropagation and Update:**
        *   Zero the gradients of the optimizer [S3, S18, unofficial].
        *   Compute gradients: `loss.backward()` [S3, S18, unofficial].
        *   *Pitfall:* Higher-order derivative computations (e.g., for logging gradients or Laplacians) must occur *before* `loss.backward()` because the backward pass frees the autograd graph [S3, unofficial].
        *   Step the optimizer: `optimizer.step()` [S3, S18, unofficial].
        *   Update the learning rate according to the scheduler [S5, S9, S18, unofficial].

    e.  **Logging and Checkpointing:**
        *   Log loss and PSNR every $I_{print}$ steps [S3, S11, unofficial].
        *   Render validation images every $I_{img}$ steps [S2, S11, unofficial].
        *   Save model checkpoints every $I_{weights}$ steps [S2, S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader

def train_nerf(model, train_dataset, val_dataset, config):
    """
    Main training loop for NeRF.
    
    Args:
        model: Neural Radiance Field model (MLP).
        train_dataset: Dataset containing images, poses, and intrinsics.
        val_dataset: Validation dataset.
        config: Dictionary containing hyperparameters.
    """
    # 1. Initialization
    optimizer = optim.Adam(model.parameters(), lr=config['lr'])
    # Example: Exponential Decay Scheduler
    scheduler = optim.lr_scheduler.ExponentialLR(optimizer, gamma=config['decay_rate'])
    
    # Precompute rays if batching is enabled
    use_batching = config.get('use_batching', True)
    rays_rgb = None
    if use_batching:
        # Generate all rays from training images
        # Shape: [Total_Pixels, 3, 3] -> [Origin, Direction, Target_RGB]
        rays_rgb = generate_all_rays(train_dataset) 
        rays_rgb = rays_rgb.float()
        i_batch = 0
        N_rand = config['batch_size'] # e.g., 1024

    n_iters = config['n_iters'] # e.g., 100,000
    log_every = config['log_every'] # e.g., 100
    render_every = config['render_every'] # e.g., 5000
    save_every = config['save_every'] # e.g., 10000

    model.train()
    
    for i in range(1, n_iters + 1):
        start_time = torch.cuda.Event(enable_timing=True)
        end_time = torch.cuda.Event(enable_timing=True)
        start_time.record()

        # 2. Ray Sampling
        if use_batching:
            # Slice batch
            batch_data = rays_rgb[i_batch : i_batch + N_rand]
            # Transpose to [3, Batch, 3]
            batch_data = batch_data.transpose(0, 1)
            batch_rays = batch_data[:2] # Origins and Directions
            target_s = batch_data[2]    # Target Colors
            
            i_batch += N_rand
            if i_batch + N_rand > rays_rgb.shape[0]:
                # Shuffle and reset
                indices = torch.randperm(rays_rgb.shape[0])
                rays_rgb = rays_rgb[indices]
                i_batch = 0
        else:
            # Random image and pixel sampling
            img_idx = torch.randint(0, len(train_dataset), (1,)).item()
            # Get rays and targets for this image
            batch_rays, target_s = get_random_rays_from_image(train_dataset, img_idx, N_rand)

        # 3. Forward Pass
        optimizer.zero_grad()
        
        # Render rays
        # model.render_rays returns predicted RGB, depth, etc.
        # If coarse/fine, it returns both
        render_output = model.render_rays(batch_rays)
        pred_rgb = render_output['rgb']
        
        # 4. Loss Computation
        # MSE Loss
        mse_loss = nn.functional.mse_loss(pred_rgb, target_s)
        loss = mse_loss
        
        # Add coarse loss if available
        if 'rgb_coarse' in render_output:
            coarse_loss = nn.functional.mse_loss(render_output['rgb_coarse'], target_s)
            loss = loss + coarse_loss

        # 5. Backpropagation
        loss.backward()
        optimizer.step()
        
        # Update scheduler
        scheduler.step()

        end_time.record()
        torch.cuda.synchronize()
        elapsed = start_time.elapsed_time(end_time)

        # 6. Logging and Checkpointing
        if i % log_every == 0:
            psnr = -10.0 * torch.log10(mse_loss).item()
            print(f"Iter {i}: Loss {loss.item():.6f}, PSNR {psnr:.2f}, Time {elapsed:.2f}ms")
            
        if i % render_every == 0:
            with torch.no_grad():
                # Render validation image
                val_ray, val_target = get_val_ray(val_dataset)
                val_pred = model.render_rays(val_ray)['rgb']
                val_psnr = -10.0 * torch.log10(nn.functional.mse_loss(val_pred, val_target)).item()
                print(f"Val PSNR at iter {i}: {val_psnr:.2f}")

        if i % save_every == 0:
            torch.save({
                'step': i,
                'model_state_dict': model.state_dict(),
                'optimizer_state_dict': optimizer.state_dict(),
                'scheduler_state_dict': scheduler.state_dict()
            }, f'checkpoint_{i}.pth')

    return model
```

**Parameters with Typical Values:**
*   `lr` (Learning Rate): $5 \times 10^{-4}$ [S11, unofficial].
*   `batch_size` ($N_{rand}$): 1024 or 2048 [S5, S11, unofficial].
*   `n_iters`: 100,000 [S11, unofficial].
*   `decay_rate`: 0.1 [S6, S18, unofficial].
*   `decay_steps`: `args.lrate_decay * 1000` [S5, S18, unofficial].
*   `near` / `far`: Scene-dependent (e.g., 2.0 / 6.0 for Blender) [S13, S2, unofficial].
*   `num_samples`: 128 (coarse), 48 (fine) [S5, S13, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by chunking rays into smaller batches during the forward pass [S5, S14, unofficial].
*   **Instability in Growing Networks:** Directly growing a randomly initialized sub-network causes density to drop to zero. Initialize alpha linear weights of grown sub-networks to match the parent to inherit density information [S1].
*   **Empty Checkpoints:** If a checkpoint is loaded but the step counter is 0, it is considered empty and should trigger an error [S15, unofficial].
*   **Gradient Graph Freeing:** Higher-order derivatives (gradients, Laplacians) for logging must be computed before `loss.backward()` [S3, unofficial].
*   **Learning Rate Scheduling Resumption:** When resuming from a checkpoint, the scheduler's last epoch must be set to `start_epoch - 1` to ensure correct scheduling [S9, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a neural radiance field (NeRF) during optimization. It solves the problem of computing gradients for scene parameters by sampling points along camera rays, querying a neural network for volume density and color, and integrating them via alpha compositing. It sets itself apart from evaluation by using stochastic perturbations, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results.

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
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. This baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21]. It relies on 'oracle' scale information that does not exist for most datasets [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S22, unofficial]. While training from scratch sometimes works, two-stage training is consistently effective [S22, unofficial].

### Guided NeRF Optimization for MVS
A multi-view depth estimation method that utilizes conventional reconstruction (sparse SfM+MVS) and learning-based priors over Neural Radiance Fields (NeRF) [S23]. It performs test-time optimization directly over implicit volumes to eliminate the need for accurate pixel matching across views, addressing the shape-radiance ambiguity of NeRF in indoor scenes [S23]. Step 1 runs COLMAP on the test scene to acquire per-view sparse depth maps by projecting fused 3D point clouds after multi-view stereo [S23]. Step 2 adapts a monocular depth network to the target scene by finetuning it on the sparse depth maps from COLMAP using a scale-invariant loss [S23].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S24]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S24]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S24].

## Operating characteristics
*   **Time Complexity:** The training time is dominated by the forward pass of the neural network and the volume rendering integration. With $N_{rays}$ rays and $N_{samples}$ samples per ray, the complexity is $O(N_{rays} \cdot N_{samples})$ per iteration [S5, S14, unofficial].
*   **Memory Complexity:** Memory usage scales with the number of rays processed per batch and the number of samples per ray. Chunking rays limits peak VRAM usage to $O(N_{chunk} \cdot N_{samples})$ [S5, S14, unofficial].
*   **Measured Performance:** In standard implementations (e.g., Blender dataset), training typically runs for 100,000 iterations [S11, unofficial]. With a batch size of 1024 and 128 samples, this can take several hours on a modern GPU [S17, unofficial].
*   **Scaling:** Performance scales with the number of training images and the resolution of the input images. Higher resolutions require more rays or larger chunks, increasing memory pressure [S15, unofficial].
*   **Strengths:** Simple to implement; leverages standard deep learning frameworks; produces high-quality novel views when given sufficient input views [S5, S14, unofficial].
*   **Limitations:** Slow training convergence; high memory consumption for high-resolution scenes; sensitive to hyperparameters (learning rate, batch size) [S15, S17, unofficial].
*   **When to Use:** When high-quality novel view synthesis is required and computational resources are available.
*   **When Not to Use:** When real-time training or inference is required; when input views are extremely sparse (without additional regularization or priors) [S1][S17, unofficial].

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, S18, unofficial]. It builds upon the concept of coordinate-based neural networks for implicit scene representation [S3, S18, unofficial]. The method was developed to address the limitations of traditional 3D reconstruction methods (like MVS) in handling complex geometry and appearance, particularly for novel view synthesis [S5, S18, unofficial]. It has since become a foundational technique in 3D computer vision and graphics, inspiring numerous variants and extensions [S1, S24].

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
[S23] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S24] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
