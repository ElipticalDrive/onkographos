# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth training views by updating the parameters of a continuous volumetric radiance field [S5, S6, S14, unofficial]. It solves the problem of learning a 3D scene representation from 2D images by treating the volume rendering equation as a differentiable computational graph [44-radiance-field-optimization.md]. What sets it apart from its siblings is its foundational role as the standard stochastic gradient descent loop that samples random rays, performs differentiable volume integration, and updates network weights, whereas methods like [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md) or [Staggered Optimization Scheme](5832-staggered-optimization-scheme.md) introduce multi-stage or decoupled optimization logic, and [Photometric Optimization](5790-photometric-optimization.md) focuses specifically on the loss formulation rather than the loop structure itself.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics $C$ and intrinsics $K$), near/far bounds ($n, f$), and a neural network model $F_\Theta$ (typically an MLP) parameterized by $\Theta$.
*   **Outputs:** Updated network weights $\Theta$ and optionally a trained radiance field capable of synthesizing novel views.

**Data Structures:**
*   **Rays:** A batch of rays is represented as a tensor of shape $[B, 3, 3]$ or $[B, 6]$ depending on implementation, where $B$ is the batch size ($N_{rand}$). The channels typically correspond to ray origin $(o_x, o_y, o_z)$, ray direction $(d_x, d_y, d_z)$, and target RGB color $(r, g, b)$ [S5, S14, unofficial]. In some implementations, rays are pre-computed into a flat array of shape $[(N-1) \cdot H \cdot W, 3, 3]$ containing origins, directions, and target colors, shuffled at the start of each epoch [S5, S14, unofficial].
*   **Model State:** Checkpoints store the model state dictionary (`coarse`, `fine`), optimizer state, and current step/epoch [S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S9, S11, unofficial].
    *   Initialize the neural network model(s) (e.g., coarse and fine NeRFs) [S11, unofficial].
    *   Initialize the optimizer (typically Adam) with a base learning rate $lr$ [S5, S11, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Pre-compute all possible rays from training images into a flat array if batching is used [S5, S14, unofficial].

2.  **Iteration Loop:** For iteration $i$ from 1 to $N_{iters}$:
    *   **Ray Sampling:**
        *   If using pre-computed batching: Slice the next $N_{rand}$ rays from the shuffled ray array [S5, S14, unofficial]. If the array is exhausted, shuffle it again [S11, unofficial].
        *   If using on-the-fly sampling: Randomly select $N_{rand}$ pixel coordinates from training images [S4, S14, unofficial]. Convert these coordinates to ray origins and directions using camera intrinsics [S4, unofficial].
        *   Gather the corresponding ground-truth RGB colors for these pixels [S4, S14, unofficial].
    *   **Forward Pass (Rendering):**
        *   Pass the batch of rays through the volume rendering function [S6, S14, unofficial].
        *   The rendering function samples points along each ray, queries the neural network for density $\sigma$ and color $\mathbf{c}$, and integrates them using the emission-absorption model [44-radiance-field-optimization.md].
        *   If using a coarse-to-fine strategy, render with both coarse and fine networks [S11, unofficial].
        *   *Pitfall:* To manage VRAM, split the ray batch into chunks if the batch size exceeds memory limits [S5, S14, unofficial].
    *   **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between the predicted RGB color $\hat{C}$ and the ground-truth color $C_{gt}$ [S5, S14, unofficial]:
            $$ \mathcal{L}_{rgb} = \frac{1}{B} \sum_{j=1}^B (\hat{C}_j - C_{gt,j})^2 $$
        *   If a coarse network is used, compute the MSE loss for the coarse output $\hat{C}_{coarse}$ and add it to the total loss [S11, S18, unofficial]:
            $$ \mathcal{L}_{coarse} = \frac{1}{B} \sum_{j=1}^B (\hat{C}_{coarse,j} - C_{gt,j})^2 $$
        *   Total Loss: $\mathcal{L} = \mathcal{L}_{rgb} + \lambda_{coarse} \mathcal{L}_{coarse}$ [S11, unofficial].
    *   **Backpropagation and Update:**
        *   Zero the gradients of the optimizer [S18, unofficial].
        *   Compute gradients via backpropagation: $\mathcal{L}.backward()$ [S18, unofficial].
        *   Step the optimizer: $\Theta \leftarrow \Theta - \eta \nabla_\Theta \mathcal{L}$ [S18, unofficial].
        *   Update the learning rate according to the scheduler [S18, unofficial].
    *   **Logging and Checkpointing:**
        *   Log loss and PSNR metrics every $K$ steps [S11, unofficial].
        *   Render validation images every $M$ steps [S11, unofficial].
        *   Save model checkpoints every $P$ steps [S11, unofficial].

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
        model: Neural Radiance Field model (MLP).
        train_dataset: Dataset returning (rays_o, rays_d, rgb_gt).
        val_dataset: Validation dataset.
        config: Dictionary containing hyperparameters.
    """
    # 1. Initialization
    optimizer = optim.Adam(model.parameters(), lr=config['lr'])
    scheduler = optim.lr_scheduler.ExponentialLR(optimizer, gamma=config['decay_rate'])
    
    # Pre-compute rays if necessary (simplified here as DataLoader handles batching)
    # In original NeRF [S5, unofficial], rays are pre-computed into a flat array and shuffled.
    train_loader = DataLoader(train_dataset, batch_size=config['batch_size'], shuffle=True)
    
    n_iters = config['n_iters']
    log_every = config['log_every']
    save_every = config['save_every']
    
    model.train()
    
    for step in range(n_iters):
        # 2. Ray Sampling
        # Get a batch of rays and target colors from the dataset
        # Batch shape: [B, 3] for origins, [B, 3] for directions, [B, 3] for targets
        rays_o, rays_d, target_rgb = next(iter(train_loader))
        
        # 3. Forward Pass
        optimizer.zero_grad()
        
        # Render the batch of rays
        # render_rays returns predicted RGB, and optionally coarse RGB, depth, etc.
        # Implementation depends on specific NeRF variant (e.g., coarse/fine)
        output = model.render_rays(rays_o, rays_d)
        pred_rgb = output['rgb']
        
        # 4. Loss Computation
        # MSE Loss
        mse_loss = nn.functional.mse_loss(pred_rgb, target_rgb)
        
        # If coarse-to-fine, add coarse loss
        if 'rgb_coarse' in output:
            coarse_loss = nn.functional.mse_loss(output['rgb_coarse'], target_rgb)
            total_loss = mse_loss + coarse_loss
        else:
            total_loss = mse_loss
            
        # 5. Backpropagation and Update
        total_loss.backward()
        optimizer.step()
        scheduler.step()
        
        # 6. Logging and Checkpointing
        if step % log_every == 0:
            psnr = -10.0 * torch.log10(mse_loss).item()
            print(f"Step {step}: Loss={total_loss.item():.6f}, PSNR={psnr:.2f}")
            
        if step % save_every == 0:
            torch.save({
                'step': step,
                'model_state_dict': model.state_dict(),
                'optimizer_state_dict': optimizer.state_dict(),
            }, f'checkpoint_{step}.pth')
            
    return model
```

**Parameters with Typical Values:**
*   $N_{rand}$ (Batch Size): 1024 or 2048 rays per iteration [S5, S11, unofficial].
*   Learning Rate ($lr$): $5 \times 10^{-4}$ for Adam [S11, unofficial].
*   Decay Rate: 0.1 [S6, S18, unofficial].
*   Decay Steps: $args.lrate\_decay \times 1000$ [S5, S18, unofficial].
*   Total Iterations ($N_{iters}$): 100,000 [S11, unofficial].
*   Chunk Size: Limits rays processed per forward pass to manage VRAM [S5, S14, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by chunking rays [S5, S14, unofficial].
*   **Gradient Freeing:** Higher-order derivatives (e.g., for logging gradients) must be computed before `loss.backward()` [S3, unofficial].
*   **Resuming Training:** When loading a checkpoint, the scheduler's last epoch must be set to `start_epoch - 1` to maintain correct decay [S9, unofficial].
*   **Empty Checkpoints:** If a loaded checkpoint has step 0, it should be treated as invalid [S15, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a neural radiance field during optimization. It sets itself apart from evaluation by using stochastic perturbations, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the static and dynamic hyperparameters, optimizer settings, and schedule strategies required to stabilize and accelerate the differentiable volume rendering optimization loop. It solves the problem of divergent or slow convergence in non-convex radiance field objectives by managing learning rates, batch sizes, sampling densities, and regularization weights. What sets this method apart from its siblings is that it provides the *initialization and scheduling* parameters (e.g., base learning rates, decay schedules, initial alpha values) rather than the execution logic ([Training Loop Setup](1522-training-loop-setup.md)) or the specific loss formulations ([Regularization and Loss Functions](8586-regularization-and-loss-functions.md)).

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Camera Pose and Parameter Optimization refers to the family of methods that jointly optimize the parameters of a Neural Radiance Field (NeRF) and the camera poses (extrinsics) and/or intrinsics (e.g., focal length) from a set of input images. It solves the problem of reconstructing 3D scenes when camera parameters are unknown, inaccurate, or when the system requires online pose estimation for navigation. Unlike baseline NeRF, which relies on pre-estimated poses from Structure-from-Motion (SfM) pipelines like COLMAP, these methods treat camera parameters as learnable variables updated via gradient descent alongside the NeRF weights. This family encompasses static scene reconstruction with unknown poses, hybrid 2D/3D registration, and online robot navigation filters.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Regularization and Loss Functions define the objective criteria used to optimize Neural Radiance Fields (NeRFs) beyond simple photometric reconstruction. They solve the under-constrained nature of volume rendering, particularly in sparse-view settings, by preventing artifacts such as "floaters," enforcing geometric smoothness, and aligning semantic content. This family is distinguished from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the *what* (the loss terms) rather than the *how* (the integration method), and from [Training Configuration](2332-training-configuration.md) by providing the mathematical definitions of the optimization targets.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Progressive Inpainting-and-Erasing (PERF) is a training strategy for single-view NeRF that avoids geometry conflicts between visible and invisible regions by computing a mask of conflicted regions and eliminating them from NeRF training. It sets itself apart from standard single-view NeRF methods by integrating a collaborative RGBD inpainting step with a progressive erasing mechanism to handle the ambiguity of unseen geometry, building on the [NeRF Training Loop](970-nerf-training-loop.md) by dynamically expanding the reference view set.

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
Scene Representation and Architecture defines the neural network structures used to approximate the continuous radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ in differentiable volume rendering. It solves the problem of representing 3D geometry and appearance as a differentiable function that can be optimized via gradient descent. It sets itself apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the network topology and input encoding rather than the sampling strategy, and from [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md) by typically assuming a single global representation rather than decomposing the scene.

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
This family of methods accelerates NeRF training and rendering by reducing the computational cost of evaluating the neural radiance field. It addresses the bottleneck of querying high-dimensional MLPs for every sample point along every ray. Unlike standard uniform or adaptive sampling over the entire scene bounding box, these methods restrict evaluation to sparse structures (visual hulls) or use lightweight approximations (caches) to predict radiance, thereby reducing the number of expensive neural network queries.

### [Deceptive-NeRF/3DGS Pipeline](11860-deceptive-nerf-3dgs-pipeline.md)
Deceptive-NeRF/3DGS is a sparse-view reconstruction method that enhances NeRF or 3D Gaussian Splatting (3DGS) by continuously densifying input observations with synthetic pseudo-observations generated by a diffusion model during training. It distinguishes itself from sibling methods by introducing an uncertainty map derived from view-consistency warping to guide the diffusion process, ensuring that generated pseudo-observations adhere to epipolar constraints and 3D geometry, rather than relying on ensemble learning or variational inference for uncertainty estimation.

### [MELON](11909-melon.md)
MELON is a method for 3D inverse rendering that simultaneously reconstructs a neural radiance field and estimates camera poses from a set of unposed images. It solves the problem of local minima in pose estimation by using an encoder to amortize inference and a "modulo loss" that maps the latent pose space to a quotient set, allowing gradient descent to converge despite rotational symmetries. Unlike adversarial methods such as GNeRF, MELON does not require a discriminator, enabling ab-initio reconstruction with few images.

### [Urban Radiance Fields](11973-urban-radiance-fields.md)
Urban Radiance Fields is a method for 3D reconstruction and view synthesis in large-scale urban environments using sparse multi-view images supplemented by LiDAR data. It addresses the challenges of unbounded outdoor scenes, limited camera pose variety, and large-scale geometry by incorporating specific depth, sight, and sky segmentation losses into the standard NeRF training loop. Unlike standard NeRF which relies solely on photometric consistency, this method enforces geometric constraints via LiDAR and semantic constraints via sky segmentation to stabilize training in open environments.

### [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md)

### [DFF (Decomposing NeRF for Editing Via Feature Field Distillation)](12068-dff-decomposing-nerf-for-editing-via-feature-field-distillat.md)
DFF is a method that enables 3D semantic segmentation and editable novel-view synthesis by distilling features from a 2D teacher network (such as LSeg or DINO) into a neural radiance field, allowing the decomposition of scenes into semantic regions. It solves the problem that original NeRF feature spaces are entangled with low-level factors like color or spatial adjacency rather than semantic similarity, making naive decomposition robust. DFF is distinct from sibling methods like [Aug-NeRF](10294-aug-nerf.md) or [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md) by explicitly training a dedicated feature field via distillation from a pre-trained 2D encoder, rather than relying on intrinsic network features or external geometric constraints alone.

### [Blended-NeRF](12273-blended-nerf.md)
Blended-NeRF is a framework for editing a specific region of interest (ROI) in an existing Neural Radiance Field (NeRF) scene based on text prompts and a user-defined 3D ROI box. It solves the problem of local 3D editing by initializing a generator NeRF with the original scene's weights and blending the synthesized content with the original scene using volumetric blending. Unlike methods that synthesize new objects from scratch (e.g., Dream Fields) or require semantic decomposition/2D masks for localization, Blended-NeRF operates on any region using a 3D box without learning a new feature space.

### [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md)
This family of methods integrates Monocular Depth Estimation (MDE) priors into Neural Radiance Field (NeRF) training to resolve the shape-radiance ambiguity and improve geometric accuracy, particularly in sparse-view or few-shot settings. Unlike standard NeRF which relies solely on photometric consistency, these methods use MDE to guide volume sampling bounds, regularize depth distributions via Earth Mover’s Distance, or adapt the MDE network itself to the scene's absolute geometry through bidirectional knowledge transfer. They differ from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by explicitly injecting external geometric constraints derived from 2D depth priors into the 3D optimization loop.

### [NeRF-In Inpainting Optimization](12372-nerf-in-inpainting-optimization.md)
NeRF-In is a framework for removing unwanted objects from a 3D scene represented by a pre-trained Neural Radiance Field (NeRF) without requiring category-specific training data or additional training procedures. It solves the problem of intuitive 3D editing by allowing a user to specify a region for removal via a 2D mask on a single view, then optimizing the NeRF parameters to inpaint that region across all views using RGB-D priors. Unlike methods that require retraining from scratch or category-specific conditional fields, NeRF-In updates an existing model by jointly minimizing color and depth guiding losses derived from 2D image inpainting and depth completion.

### [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md)
This method jointly optimizes a time-dependent Neural Radiance Field (NeRF) and continuous camera motion from monocular video, representing scene geometry as local Signed Distance Fields (SDFs) and appearance at each time step. Unlike prior approaches that train a single global NeRF mapped via noisy initial transformations, this method uses local time-dependent representations constrained by consistency losses to avoid local minima and accurately reconstruct full scene geometry.

### [Adaptive View Selection for Inpainting](12405-adaptive-view-selection-for-inpainting.md)
This method addresses the inconsistency of single-view 2D inpainting in 3D scenes by integrating a 2D appearance prior with multi-view geometric consistency. It differs from standard NeRF training [NeRF Training Loop](970-nerf-training-loop.md) by dynamically switching the supervision signal for occluded regions between a 2D inpainter (LaMa) and other training views based on local texture richness. It builds on [NeRF Training Loop](970-nerf-training-loop.md) and [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md) by adding a gradient-based view selection mechanism to resolve conflicts between 2D priors and 3D geometry.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, multiple jittered rays are sampled uniformly at random through the spatial footprint of the pixel, and the mean of the rendered values is used as the predicted color in the loss function [S21]. This baseline relies on 'oracle' scale information that does not exist for most datasets and performs worse than mip-NeRF while being significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S22, unofficial]. This two-stage approach is consistently effective where training from scratch often fails [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### RapNeRF
A NeRF model tailored for view extrapolation, contrasting with standard NeRFs that excel at interpolation [S24]. Training begins with a baseline NeRF to recover a rough mesh, from which a Ray Atlas (RA) is created by mapping training ray directions onto its vertices [S24]. Random Ray Casting (RRC) is employed: for a training ray hitting a surface point, a backward ray is cast from that point toward a new origin with uniformly sampled angular perturbations [S24]. RRC and RA augment training rays with a predetermined probability to improve extrapolation [S24].

### NELF
A neural volumetric rendering framework modeling scene appearance using light transport vectors, enabling realistic relighting and view synthesis of human portraits from only five input images [S24]. A UNet-style CNN extracts per-view features, and an MLP regresses volume density and transport vectors [S24]. Environment maps are estimated to disentangle lighting, and the model is trained on synthetic data and adapted to real images via a domain adaptation module [S24].

### JaxNeRF
A re-implementation of the original NeRF in JAX, used as a benchmark comparison for early works seeking to improve on training and rendering speed [S24]. It is slightly faster and more suited for distributed computing than the original TensorFlow implementation of NeRF [S24].

### SNeS
A method improving geometry by learning probable symmetries for partly symmetric and partly hidden in-scene objects through soft symmetry constraints on geometry and material properties [S24].

### NeRF with Per-Frame Latent Codes
A method using per-frame learned latent codes and adding facial expression as a 76-dimensional coefficient of a morphable model constructed from Face2Face [S24].

### DreamFusion
A text-to-3D generation method that optimizes a NeRF to render images with high scores from a text-to-image diffusion model, requiring no 3D training data [S25]. Introduced in 2022 [S25].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot N_{samples} \cdot C_{MLP})$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), $N_{samples}$ is the number of samples per ray, and $C_{MLP}$ is the cost of a forward pass through the neural network [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot N_{samples} \cdot C_{MLP})$ for the forward pass, plus $O(N_{iters})$ if storing the full computational graph for backpropagation (though AD handles this efficiently) [44-radiance-field-optimization.md]. VRAM usage is a primary constraint, often requiring chunking of ray batches [S5, S14, unofficial].
*   **Measured Performance:** In standard implementations (e.g., Blender dataset), training takes approximately 100,000 iterations [S11, unofficial]. With a batch size of 1024 and 128 samples per ray, this can take several hours on a modern GPU [S17, unofficial].
*   **Scaling:** Performance scales with batch size, but larger batches hinder data loading performance and prevent full GPU occupancy, while smaller batches cap iterations per second [S15, unofficial]. A batch size of $2^{10}$ (1024) is often chosen as a trade-off [S15, unofficial].
*   **Strengths:** Simple to implement, differentiable end-to-end, produces high-quality novel views for interpolation [S24].
*   **Limitations:** Slow training due to repeated MLP evaluations, struggles with view extrapolation [S24], and requires significant VRAM for large scenes or high resolutions [S5, S14, unofficial].
*   **When to Use:** Standard novel view synthesis from multi-view images with known poses.
*   **When Not to Use:** When view extrapolation is required (use RapNeRF [S24]), when computational resources are extremely limited (use acceleration methods [11405-acceleration-and-sparse-structures.md]), or when camera poses are unknown (use Camera Pose Optimization [7611-camera-pose-and-parameter-optimization.md]).

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" in 2020 [S5, unofficial][S25]. It builds on the concept of coordinate-based neural networks for implicit scene representation [S3, unofficial]. The method solves the problem of synthesizing novel views from a sparse set of 2D images by optimizing a continuous volumetric function via gradient descent [S5, unofficial]. It has become the foundational training procedure for subsequent NeRF variants and 3D deep learning methods [S24, S25].

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
[S25] Differentiable Rendering - 可微渲染：从2D 学习3D, https://www.ultralytics.com/zh/glossary/differentiable-rendering, fetched 2026-10-30
