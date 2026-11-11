# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth training views by updating the parameters of a continuous volumetric radiance field [S5, S6, S14, unofficial]. It solves the problem of learning a differentiable scene representation from multi-view images by repeatedly sampling rays, rendering predictions via volume integration, and backpropagating gradients through the rendering equation [S5, S6, unofficial]. It sets itself apart from evaluation by employing stochastic ray sampling, adaptive perturbation, and chunked processing to manage memory, whereas evaluation disables these for deterministic, full-image rendering [S2, S4, S9, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics) and intrinsics (focal length, principal point, resolution $H \times W$). Ground-truth RGB values $\mathbf{C}_{gt}$.
*   **Outputs:** Optimized neural network weights $\Theta$ representing the radiance field $F_\Theta(\mathbf{x}, \mathbf{d}) = (\sigma, \mathbf{c})$, where $\sigma$ is volume density and $\mathbf{c}$ is view-dependent color.

**Data Structures:**
*   **Rays:** A batch of rays is represented as a tensor of shape $[B, 3, 3]$ or $[B, 6]$ (depending on implementation), where $B$ is the batch size (number of rays). The first 3 dimensions typically store the ray origin $(o_x, o_y, o_z)$ and the next 3 store the ray direction $(d_x, d_y, d_z)$ [S5, S14, unofficial]. In some implementations, the target RGB is appended, forming a shape of $[B, 3, 3]$ where axis 1 is channel type (0: origin, 1: direction, 2: target) and axis 2 is the component [S14, unofficial].
*   **Pre-computed Rays:** To accelerate data loading, rays and targets are often pre-computed into a flat array `rays_rgb` of shape $[(N-1) \cdot H \cdot W, 3, 3]$ (or similar flattened structure) containing all possible ray-target pairs from the training set, shuffled at the start of each epoch [S5, S14, unofficial].
*   **Optimizer State:** Standard Adam optimizer state (momentum buffers) for network parameters [S5, S14, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S9, S11, unofficial].
    *   Initialize the neural network model (MLP) with random weights [S5, S15, unofficial]. For progressive methods, initialize grown sub-networks' alpha linear weights to match the parent to avoid instability [S1].
    *   Initialize the optimizer (typically Adam) with a base learning rate $lr$ (e.g., $5 \times 10^{-4}$) [S11, S14, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Pre-compute all ray origins, directions, and target RGB values from training images into a flat array `rays_rgb` [S5, S14, unofficial]. Shuffle this array [S11, unofficial].

2.  **Iteration Loop:** For iteration $i$ from 1 to $N_{iters}$ (typically 100,000 to 1,000,000) [S11, S14, unofficial]:
    a.  **Ray Sampling:**
        *   Select a batch of $N_{rand}$ rays (e.g., 1024 or 2048) from the pre-computed `rays_rgb` array [S5, S14, unofficial].
        *   If batching is not used, randomly select a training image, generate all rays for that image, optionally apply a central crop (precrop) for the first $P_{crop}$ iterations, and sample $N_{rand}$ unique pixel coordinates [S14, unofficial].
        *   Extract ray origins $\mathbf{o}$, directions $\mathbf{d}$, and target colors $\mathbf{C}_{gt}$ for the selected rays [S5, S14, unofficial].
        *   Apply perturbations to sampling positions if enabled (jitter amount $\epsilon$) to break symmetry and improve reconstruction [S2, unofficial].
    b.  **Forward Pass (Rendering):**
        *   Sample $K$ points along each ray within the near/far bounds (e.g., near=2.0, far=6.0 for Blender scenes) [S13, unofficial]. Points are sampled uniformly or stratified [S5, unofficial].
        *   Query the neural network $F_\Theta$ at each point $\mathbf{x}_k$ and direction $\mathbf{d}$ to obtain density $\sigma_k$ and color $\mathbf{c}_k$ [S5, S6, unofficial].
        *   Compute transmittance $T_k$ and weights $w_k$ using the emission-absorption model:
            $$ T_k = \exp\left(-\sum_{j=1}^{k-1} \sigma_j \delta_j\right) $$
            $$ w_k = T_k (1 - \exp(-\sigma_k \delta_k)) $$
            where $\delta_k$ is the distance between adjacent samples [S5, S6, unofficial].
        *   Render the predicted color $\hat{\mathbf{C}}(\mathbf{r})$ for each ray:
            $$ \hat{\mathbf{C}}(\mathbf{r}) = \sum_{k=1}^K w_k \mathbf{c}_k $$
        *   If a coarse-to-fine strategy is used, render a coarse image first, then sample additional points based on coarse weights (importance sampling) and render a fine image [S5, S11, unofficial].
    c.  **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) loss between the predicted color $\hat{\mathbf{C}}$ and the ground truth $\mathbf{C}_{gt}$:
            $$ \mathcal{L}_{rgb} = \frac{1}{B} \sum_{i=1}^B \|\hat{\mathbf{C}}_i - \mathbf{C}_{gt,i}\|^2 $$
            [S14, S17, unofficial].
        *   If coarse outputs are available, add the coarse loss $\mathcal{L}_{coarse}$ to the total loss [S11, S18, unofficial].
        *   Add any regularization terms (e.g., sparsity, Eikonal, Total Variation) if applicable [S7, unofficial][S8].
    d.  **Backpropagation and Update:**
        *   Zero the gradients of the optimizer [S18, unofficial].
        *   Compute gradients: $\nabla_\Theta \mathcal{L}$ [S18, unofficial].
        *   Step the optimizer: $\Theta \leftarrow \Theta - \eta \nabla_\Theta \mathcal{L}$ [S18, unofficial].
        *   Update the learning rate according to the scheduler [S18, unofficial].
    e.  **Logging and Checkpointing:**
        *   Log loss and PSNR every $I_{print}$ steps [S2, S14, unofficial].
        *   Render validation images every $I_{img}$ steps [S2, S11, unofficial].
        *   Save model checkpoints every $I_{weights}$ steps [S2, S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader

# Assume NeRFModel, render_rays, and dataset utilities are defined elsewhere
# NeRFModel: MLP taking (x, d) -> (sigma, color)
# render_rays: Differentiable volume renderer

def train_nerf(model, optimizer, scheduler, train_loader, val_loader, 
               num_iterations, batch_size, render_kwargs, log_interval, 
               checkpoint_interval, val_interval):
    """
    Main training loop for NeRF.
    
    Args:
        model: Neural Radiance Field MLP.
        optimizer: Optimizer (e.g., Adam).
        scheduler: Learning rate scheduler.
        train_loader: DataLoader yielding batches of (rays_o, rays_d, target_rgb).
        val_loader: DataLoader for validation.
        num_iterations: Total number of training steps.
        batch_size: Number of rays per batch (N_rand).
        render_kwargs: Dict containing near, far, N_samples, perturb, etc.
        log_interval: Steps between logging.
        checkpoint_interval: Steps between saving checkpoints.
        val_interval: Steps between validation rendering.
    """
    model.train()
    criterion = nn.MSELoss()
    
    for step in range(1, num_iterations + 1):
        # 1. Sample Batch
        # In a real implementation, rays are sampled from pre-computed arrays
        # or generated on-the-fly from the dataset.
        # Here we assume train_loader yields (rays_o, rays_d, target_rgb)
        # of shape [batch_size, 3] for origins, directions, and targets.
        try:
            rays_o, rays_d, target_rgb = next(data_iterator)
        except StopIteration:
            data_iterator = iter(train_loader)
            rays_o, rays_d, target_rgb = next(data_iterator)
            
        # Move to device
        rays_o = rays_o.to(device)
        rays_d = rays_d.to(device)
        target_rgb = target_rgb.to(device)
        
        # 2. Forward Pass (Rendering)
        # render_rays handles ray marching, network querying, and alpha compositing
        # It returns predicted RGB, and potentially depth, accumulation, and raw outputs
        with torch.no_grad(): # Optional: if render_rays is deterministic and we want to save memory, 
                              # but usually we need gradients through it. 
                              # However, some implementations do forward pass inside the grad context.
            # Standard practice:
            pass 
            
        # Zero gradients
        optimizer.zero_grad()
        
        # Forward pass through renderer
        # render_rays takes ray origins, directions, and model
        # Returns: rgb_map (predicted color), and optionally extras (coarse rgb, etc.)
        rgb_map, extras = render_rays(model, rays_o, rays_d, **render_kwargs)
        
        # 3. Loss Computation
        # Primary photometric loss
        loss = criterion(rgb_map, target_rgb)
        
        # Add coarse loss if available (for coarse-to-fine training)
        if 'rgb0' in extras:
            loss_coarse = criterion(extras['rgb0'], target_rgb)
            loss = loss + loss_coarse
            
        # Add regularization if defined in model or kwargs
        if 'regularization_loss' in extras:
            loss = loss + extras['regularization_loss']
            
        # 4. Backpropagation
        loss.backward()
        
        # 5. Optimization Step
        optimizer.step()
        scheduler.step()
        
        # 6. Logging and Checkpointing
        if step % log_interval == 0:
            psnr = -10.0 * torch.log10(loss).item()
            print(f"Step {step}: Loss={loss.item():.6f}, PSNR={psnr:.2f}")
            
        if step % val_interval == 0:
            evaluate(model, val_loader, step)
            
        if step % checkpoint_interval == 0:
            save_checkpoint(model, optimizer, scheduler, step, 'checkpoint.pth')

def evaluate(model, val_loader, step):
    """Render validation images."""
    model.eval()
    with torch.no_grad():
        # Sample a few validation rays or full images
        # Render and compute metrics
        pass
    model.train()
```

**Parameters with Typical Values:**
*   $N_{rand}$ (Batch Size): 1024 or 2048 rays per iteration [S5, S11, S14, unofficial].
*   $N_{iters}$: 100,000 to 1,000,000 iterations [S11, S14, unofficial].
*   Learning Rate ($lr$): $5 \times 10^{-4}$ for Adam [S11, unofficial].
*   Decay: Exponential decay with rate 0.1 every $args.lrate\_decay \times 1000$ steps [S5, S18, unofficial].
*   Near/Far Bounds: Near=2.0, Far=6.0 for synthetic Blender scenes [S13, unofficial]. For LLFF, computed from bounding depths [S2, unofficial].
*   Samples per Ray ($K$): 128 (coarse) + 64 (fine) [S5, unofficial].
*   Positional Encoding Frequency: 10 for position, 4 for direction [S11, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering full images or large batches can exceed GPU memory. Mitigated by chunking rays (processing $N_{chunk}$ rays at a time) [S5, S14, unofficial].
*   **Gradient Freeing:** Higher-order derivatives (e.g., for logging gradients) must be computed *before* `loss.backward()` to avoid freeing the autograd graph [S3, unofficial].
*   **Instability in Growth:** Directly growing a randomly initialized network causes density to drop to zero. Initialize new sub-networks' weights to match the parent [S1].
*   **Empty Checkpoints:** Loading a checkpoint with step 0 is considered an error [S15, unofficial].
*   **Learning Rate Drop:** Stop training if learning rate drops below $10^{-7}$ [S16, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a neural radiance field (NeRF) during optimization. It solves the problem of computing gradients for scene parameters by sampling points along camera rays, querying a neural network for volume density and color, and integrating them via alpha compositing. It sets itself apart from evaluation by using stochastic perturbations, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results.

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

### [DreamFusion](12087-dreamfusion.md)
DreamFusion is a text-to-3D generation method that optimizes a Neural Radiance Field (NeRF) to render images with high scores from a 2D text-to-image diffusion model, requiring no 3D training data. It differs from standard NeRF training by using a pretrained diffusion model as a perceptual loss function instead of ground-truth pixel comparisons, enabling the synthesis of novel 3D objects from text prompts.

### [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md)
This family of methods integrates Monocular Depth Estimation (MDE) priors into Neural Radiance Field (NeRF) training to resolve the shape-radiance ambiguity and improve geometric accuracy, particularly in sparse-view or few-shot settings. Unlike standard NeRF which relies solely on photometric consistency, these methods use MDE to guide volume sampling bounds, regularize depth distributions via Earth Mover’s Distance, or adapt the MDE network itself to the scene's absolute geometry through bidirectional knowledge transfer. They differ from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by explicitly injecting external geometric constraints derived from 2D depth priors into the 3D optimization loop.

### [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md)
This method jointly optimizes a time-dependent Neural Radiance Field (NeRF) and continuous camera motion from monocular video, representing scene geometry as local Signed Distance Fields (SDFs) and appearance at each time step. Unlike prior approaches that train a single global NeRF mapped via noisy initial transformations, this method uses local time-dependent representations constrained by consistency losses to avoid local minima and accurately reconstruct full scene geometry.

### [Adaptive View Selection for Inpainting](12405-adaptive-view-selection-for-inpainting.md)
This method addresses the inconsistency of single-view 2D inpainting in 3D scenes by integrating a 2D appearance prior with multi-view geometric consistency. It differs from standard NeRF training [NeRF Training Loop](970-nerf-training-loop.md) by dynamically switching the supervision signal for occluded regions between a 2D inpainter (LaMa) and other training views based on local texture richness. It builds on [NeRF Training Loop](970-nerf-training-loop.md) and [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md) by adding a gradient-based view selection mechanism to resolve conflicts between 2D priors and 3D geometry.

### [RehearsalNeRF](12477-rehearsalnerf.md)
RehearsalNeRF is a method for learning disentangled neural fields under severe illumination changes by leveraging scenes captured under stable lighting (rehearsal stages) to enforce geometric consistency between different lighting conditions. It jointly optimizes five neural fields—static objects, dynamic objects, and dynamically changing illuminations—to decouple dynamic lighting from moving/static objects, addressing the "illumination-radiance ambiguity" where illumination radiance is entangled with object radiance. Unlike methods that assume stable light sources, RehearsalNeRF explicitly handles varying illuminations and dynamic objects, enabling applications such as controlling lighting while stopping motion.

### [NeuMan](12480-neuman.md)
NeuMan is a method for novel view synthesis of a human subject interacting with a static scene, driven by a single video input. It jointly optimizes a background scene NeRF and a pose-driven human NeRF, allowing for the generation of novel views with novel human poses. It distinguishes itself from standard NeRF training by explicitly modeling the human as a deformable entity in a canonical space using SMPL parameters and an error-correction network, while simultaneously separating the human from the background to enable clean compositing.

### [AR-NeRF](12493-ar-nerf.md)
AR-NeRF is an extension of Neural Radiance Fields (NeRF) that employs an aperture camera model instead of a pinhole camera model to represent defocus effects and viewpoint changes in a unified manner within a common ray-tracing framework. It enables the fully unsupervised learning of depth and defocus effects from single natural images by jointly leveraging defocus and viewpoint cues without explicit supervision or pretrained models. Unlike standard NeRF, which assumes a pinhole camera, AR-NeRF represents each pixel using a collection of rays converging at a focus plane, allowing the model to disentangle defocus-aware and defocus-independent representations through aperture randomized training.

### [NeRF Editing and Inpainting Procedures](12494-nerf-editing-and-inpainting-procedures.md)
This family encompasses optimization procedures that modify a pre-trained Neural Radiance Field (NeRF) to achieve specific editing goals, such as object removal, semantic segmentation, or content insertion, without retraining the entire scene from scratch. Unlike standard NeRF training which assumes multi-view consistency of the original data, these methods introduce external guidance (e.g., 2D inpainting, semantic features, or text prompts) and optimize the radiance field parameters $\Theta$ to align with this guidance while preserving unedited regions. They differ from sibling methods like [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md) by focusing on post-hoc modification of an existing representation rather than incremental scene expansion.

### [Camera Preconditioning](12545-camera-preconditioning.md)
Camera Preconditioning (CamP) is a technique to normalize and decorrelate camera parameters during Neural Radiance Field (NeRF) optimization by transforming the optimization problem from direct camera parameters $\boldsymbol{\phi}_i$ to latent preconditioned parameters $\tilde{\boldsymbol{\phi}}_i$. It addresses ill-conditioning caused by mixed units (e.g., translation in world units vs. focal length in pixels) and spurious correlations between parameters, which naive optimization or simple per-parameter scaling fails to resolve. Unlike [Adam](12051-camera-pose-and-extrinsic-intrinsic-optimization.md) which normalizes update magnitudes but ignores parameter sensitivity differences, CamP explicitly scales parameters based on their projected impact on the image plane.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. The full-resolution supersampling baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### RapNeRF
A NeRF model tailored for view extrapolation, contrasting with standard NeRFs that excel at interpolation [S24]. Training begins with a baseline NeRF to recover a rough mesh [S24]. A Ray Atlas (RA) is created by extracting a rough 3D mesh from a pretrained NeRF and mapping training ray directions onto its vertices [S24]. Random Ray Casting (RRC) is employed: for a training ray hitting a surface point $v = o + tz d$, a backward ray is cast from $v$ toward a new origin $o'$ with uniformly sampled angular perturbations [S24]. RRC and RA augment training rays with a predetermined probability [S24].

### NELF
A neural volumetric rendering framework modeling scene appearance using light transport vectors, enabling realistic relighting and view synthesis of human portraits from only five input images [S24]. A UNet-style CNN extracts per-view features [S24]. An MLP regresses volume density and transport vectors [S24]. Environment maps are estimated to disentangle lighting [S24]. It is trained on synthetic data and adapted to real images via a domain adaptation module [S24].

### JaxNeRF
A re-implementation of the original NeRF in JAX, used as a benchmark comparison for early works seeking to improve on training and rendering speed [S24]. It is slightly faster and more suited for distributed computing than the original TensorFlow implementation of NeRF [S24].

### SNeS
A method improving geometry by learning probable symmetries for partly symmetric and partly hidden in-scene objects through soft symmetry constraints on geometry and material properties [S24].

### NeRF with Per-Frame Latent Codes
A method using per-frame learned latent codes and adding facial expression as a 76-dimensional coefficient of a morphable model constructed from Face2Face [210] [S24].

### DreamAvatar
A diffusion-based 3D avatar model generation method with text guidance, using an SMPL model as a shape prior and text-guided 2D image generation via diffusion to create training data in a DreamFusion-like 3D generation pipeline combining NeRF and diffusion [S24].

### FlipNeRF Training Pipeline
Starts with baseline mip-NeRF representation [S25]. Adds flipped reflection rays to the training loop [S25]. Applies LUE to both original and flipped rays [S25]. Applies LBFC with JSD on bottleneck features [S25]. Enforces bottleneck feature consistency between original and flipped ray pairs [S25]. Integrates LOri. from Ref-NeRF to refine normal estimation [S25]. The full model achieves PSNR 19.55, SSIM 0.767, LPIPS 0.180, Average Error 0.101, and NLL 2.56 on Realistic Synthetic 360◦ [S25].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot K)$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), and $K$ is the number of samples per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot K)$ for the forward pass, plus $O(\text{Model Size})$ for network weights [S5, unofficial]. VRAM usage is mitigated by chunking rays [S5, S14, unofficial].
*   **Measured Performance:** Training on 128x128 images for 250 epochs can take significant time, with PSNR increasing from ~10.6 to ~18.9 over 20 epochs in some implementations [S4, unofficial][S12]. For larger scenes, 100,000+ iterations are typical [S11, unofficial].
*   **Scaling:** Performance scales with batch size, but larger batches hinder data loading performance and prevent full GPU occupancy, while smaller batches cap iterations per second [S15, unofficial]. A batch size of $2^{10}$ (1024) is a common tradeoff [S15, unofficial].
*   **Strengths:** Simple to implement, differentiable end-to-end, produces high-quality novel views for interpolation [S5, unofficial][S24].
*   **Limitations:** Slow convergence, requires many views for good results, struggles with view extrapolation [S24], prone to "floaters" in empty space [S1].
*   **When to Use:** When high-quality novel view synthesis is needed for scenes with sufficient multi-view coverage and static geometry.
*   **When Not to Use:** For real-time rendering (without acceleration), for scenes with significant motion or dynamic lighting (without extensions), or for view extrapolation tasks [S24].

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial]. It builds on the concept of coordinate-based neural networks for implicit scene representation [S3, unofficial]. The method uses stochastic gradient descent with Adam optimizer to minimize the MSE between rendered and ground-truth colors [S5, S14, unofficial]. It has become a foundational technique in 3D computer vision, inspiring numerous variants for acceleration, dynamic scenes, and different applications [S24].

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
