# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview

The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth observations by updating the parameters of a continuous volumetric representation [S5, S6, S14, unofficial]. It solves the problem of learning a scene's geometry and appearance by repeatedly sampling rays, computing differentiable volume renderings, and propagating gradients through the rendering equation [S4, S9, unofficial]. It sets itself apart from evaluation rendering by using stochastic sampling with perturbation and adaptive step sizes to facilitate gradient-based optimization, whereas evaluation uses deterministic rendering for reproducible metrics [S2, S9, unofficial].

## How it works

**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics) and intrinsics, defining a set of rays $\mathbf{r}_i$ originating from pixel coordinates. Ground-truth RGB colors $\mathbf{C}_{gt}$ for these rays.
*   **Outputs:** Optimized neural network weights $\Theta$ (and potentially camera poses or other auxiliary parameters) that define the radiance field $F_\Theta$.

**Data Structures:**
*   **Rays Array:** Pre-computed or on-the-fly generated array of ray bundles. A common layout is a 3D tensor `rays_rgb` of shape $[(N-1) \times H \times W, 3, 3]$ where axis 0 is the flattened pixel index, axis 1 represents the channel type (0: ray origin, 1: ray direction, 2: target RGB), and axis 2 contains the XYZ/RGB components [S5, S14, unofficial]. Data type is typically `float32` [S5, S14, unofficial].
*   **Model State:** Contains the neural network parameters (e.g., MLP weights for density $\sigma$ and color $\mathbf{c}$) and optimizer states (momentum buffers, etc.) [S11, unofficial].
*   **Checkpoint:** A dictionary structure storing `'model'` (state dict), `'optimizer'` (state dict), and `'epoch'` or `'step'` (int) [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the neural network model (e.g., NeRF MLP) [S5, S11, unofficial]. If using a coarse-fine architecture, initialize both coarse and fine networks [S11, unofficial].
    *   Initialize the optimizer (typically Adam) with a learning rate $lr$ [S5, S11, S17, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Precompute ray bundles from training images if using batching [S5, S14, unofficial].

2.  **Iteration Loop (for $i = 1$ to $N_{iters}$):**
    *   **Ray Sampling:**
        *   Select a batch of $N_{rand}$ rays [S5, S14, unofficial].
        *   If using pre-computed batching, slice the `rays_rgb` array and shuffle when exhausted [S5, S14, unofficial].
        *   If not batching, randomly select pixel coordinates from training images, optionally applying a central crop for the first $P_{crop}$ iterations [S14, unofficial].
        *   Construct ray origins $\mathbf{o}$ and directions $\mathbf{d}$ from the selected pixels and camera parameters [S4, S7, unofficial].
        *   Retrieve ground-truth target colors $\mathbf{C}_{gt}$ for these rays [S4, S14, unofficial].
    *   **Forward Pass (Rendering):**
        *   Sample points along each ray within the near/far bounds [S13, unofficial].
        *   Query the neural network for density $\sigma$ and color $\mathbf{c}$ at these points [S5, S6, unofficial].
        *   Compute the rendered color $\hat{\mathbf{C}}$ using volume rendering (alpha compositing) [S5, S6, unofficial].
        *   If using a coarse-fine scheme, render coarse predictions $\hat{\mathbf{C}}_{coarse}$ and fine predictions $\hat{\mathbf{C}}_{fine}$ [S11, unofficial].
    *   **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) loss between predicted and ground-truth colors:
            $$ \mathcal{L}_{rgb} = \frac{1}{N_{rand}} \sum_{j=1}^{N_{rand}} \| \hat{\mathbf{C}}_j - \mathbf{C}_{gt,j} \|^2 $$
            [S14, S17, unofficial].
        *   If coarse-fine training is used, add the coarse loss: $\mathcal{L} = \mathcal{L}_{rgb, fine} + \mathcal{L}_{rgb, coarse}$ [S11, unofficial].
        *   Add any regularization terms (e.g., sparsity, Eikonal) if applicable [S7, unofficial][S8].
    *   **Backpropagation and Update:**
        *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
        *   Compute gradients: `loss.backward()` [S18, unofficial].
        *   Update model parameters: `optimizer.step()` [S18, unofficial].
        *   Update learning rate according to the scheduler [S9, S18, unofficial].
    *   **Logging and Checkpointing:**
        *   Log loss and PSNR metrics every $I_{print}$ steps [S2, S14, unofficial].
        *   Render validation images every $I_{img}$ steps [S2, S11, unofficial].
        *   Save model checkpoint every $I_{weights}$ steps [S2, S11, unofficial].

**Formulas:**
*   **MSE Loss:**
    $$ \text{MSE}(\mathbf{x}, \mathbf{y}) = \text{mean}((\mathbf{x} - \mathbf{y})^2) $$
    [S14, unofficial].
*   **PSNR from MSE:**
    $$ \text{PSNR} = -10 \log_{10}(\text{MSE}) $$
    [S14, unofficial].
*   **Exponential Decay Learning Rate:**
    $$ lr_{new} = lr_{initial} \times \text{decay\_rate}^{(\text{global\_step} / \text{decay\_steps})} $$
    [S18, unofficial]. Here, $\text{decay\_steps} = \text{args.lrate\_decay} \times 1000$ [S5, S18, unofficial].

**Parameters:**
*   $N_{rand}$ (Batch Size): Number of rays sampled per iteration. Typical values: 1024 or 2048 [S5, S14, unofficial].
*   $lr$ (Learning Rate): Typical value: $5 \times 10^{-4}$ [S11, unofficial].
*   $N_{iters}$ (Iterations): Total number of gradient steps. Default: 100,000 [S11, unofficial].
*   $\text{decay\_rate}$: Factor for LR decay. Default: 0.1 [S6, S18, unofficial].
*   $\text{decay\_steps}$: Steps between decay. Calculated as $\text{args.lrate\_decay} \times 1000$ [S5, S18, unofficial].
*   $\text{chunk}$: Maximum rays processed per forward pass to limit VRAM [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by splitting ray batches into chunks via `chunk=args.chunk` in the render call [S5, unofficial].
*   **Gradient Computation Order:** Higher-order derivatives (e.g., for logging gradients or Laplacians) must be computed *before* `loss.backward()` because the backward pass frees the autograd graph [S3, unofficial].
*   **Resuming Training:** When loading a checkpoint, the learning rate scheduler's last epoch must be set to `start_epoch - 1` to ensure correct scheduling upon resumption [S9, unofficial].
*   **Empty Checkpoints:** If a checkpoint is loaded but the state step is 0, it is considered empty and should trigger an error [S15, unofficial].
*   **Instability in Growth:** Direct growth of randomly initialized sub-networks can cause density to drop to zero. Weights of grown sub-networks should be initialized to match the parent network [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def train_nerf_loop(model, train_dataset, optimizer, scheduler, 
                    n_iters=100000, batch_size=1024, 
                    log_every=100, render_every=5000, save_every=10000):
    """
    Main training loop for NeRF.
    
    Args:
        model: Neural Radiance Field model (MLP).
        train_dataset: Dataset containing images, poses, and intrinsics.
        optimizer: Optimizer (e.g., Adam).
        scheduler: Learning rate scheduler.
        n_iters: Total number of training iterations.
        batch_size: Number of rays per batch (N_rand).
        log_every: Frequency of logging metrics.
        render_every: Frequency of rendering validation images.
        save_every: Frequency of saving checkpoints.
    """
    model.train()
    criterion = nn.MSELoss()
    
    # Precompute rays if using batching (optional optimization)
    # rays_rgb = precompute_rays(train_dataset) # Shape: [N_pixels, 3, 3]
    
    for step in range(1, n_iters + 1):
        # 1. Ray Sampling
        # Randomly select batch_size rays from the training set
        # Implementation depends on dataset structure. 
        # Example: sample pixel coordinates, get rays_o, rays_d, and target_rgb
        rays_o, rays_d, target_rgb = sample_rays(train_dataset, batch_size)
        
        # 2. Forward Pass
        # Render the rays using the current model
        # render_rays returns predicted RGB, and potentially coarse RGB if coarse-fine
        pred_rgb, extras = render_rays(model, rays_o, rays_d)
        
        # 3. Loss Computation
        # Primary photometric loss
        loss = criterion(pred_rgb, target_rgb)
        
        # Add coarse loss if using coarse-fine training
        if 'rgb_coarse' in extras:
            loss += criterion(extras['rgb_coarse'], target_rgb)
            
        # Add regularization losses if defined (e.g., sparsity, Eikonal)
        # loss += regularization_terms(extras, model)
        
        # 4. Backpropagation
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # 5. Learning Rate Update
        scheduler.step()
        
        # 6. Logging and Checkpointing
        if step % log_every == 0:
            mse_val = loss.item()
            psnr_val = -10.0 * torch.log10(torch.tensor(mse_val))
            print(f"Step {step}: Loss={mse_val:.6f}, PSNR={psnr_val:.2f}")
            
        if step % render_every == 0:
            render_validation_images(model, train_dataset)
            
        if step % save_every == 0:
            save_checkpoint(model, optimizer, scheduler, step)

def sample_rays(dataset, batch_size):
    """
    Samples a batch of rays from the dataset.
    Returns:
        rays_o: Tensor of shape [batch_size, 3]
        rays_d: Tensor of shape [batch_size, 3]
        target_rgb: Tensor of shape [batch_size, 3]
    """
    # Placeholder for dataset-specific ray sampling logic
    # This involves picking random image indices, random pixel coordinates,
    # and constructing rays from camera poses and intrinsics.
    raise NotImplementedError("Implement ray sampling based on dataset structure")

def render_rays(model, rays_o, rays_d):
    """
    Renders rays through the NeRF model.
    Returns:
        pred_rgb: Tensor of shape [batch_size, 3]
        extras: Dictionary containing intermediate outputs (e.g., rgb_coarse)
    """
    # Placeholder for volume rendering logic
    # This involves sampling points along rays, querying the MLP,
    # and performing alpha compositing.
    raise NotImplementedError("Implement volume rendering logic")

def save_checkpoint(model, optimizer, scheduler, step):
    """
    Saves the model, optimizer, and scheduler state.
    """
    checkpoint = {
        'step': step,
        'model_state_dict': model.state_dict(),
        'optimizer_state_dict': optimizer.state_dict(),
        'scheduler_state_dict': scheduler.state_dict()
    }
    torch.save(checkpoint, f"checkpoint_step_{step}.pth")
```

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images and depth maps from a Neural Radiance Field (NeRF) during optimization. It solves the problem of computing gradients for the neural network parameters by sampling points along camera rays, querying volume density and color, and integrating them via alpha compositing. It sets itself apart from evaluation rendering by using stochastic sampling with perturbation and adaptive step sizes to facilitate gradient-based optimization, whereas evaluation uses deterministic rendering for reproducible metrics.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the static and dynamic hyperparameters, optimization schedules, and environment settings required to converge a differentiable volume rendering model. It solves the problem of balancing convergence speed, reconstruction fidelity, and computational stability by specifying optimizer states, learning rate decay strategies, batch sampling rates, and regularization weights. It sets itself apart from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the gradient descent mechanics rather than the forward rendering pass, and from [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md) by defining the temporal evolution of the optimization state rather than the initial spatial representation.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Regularization and Loss Functions define the objective landscape for optimizing Neural Radiance Fields (NeRFs), extending the standard photometric reconstruction loss to address specific failure modes such as overfitting in sparse views, geometric ambiguity ("floaters"), semantic incoherence, and style mismatches. This family sets itself apart from general optimization procedures by providing specialized mathematical formulations that constrain the density $\sigma$ and color $\mathbf{c}$ fields, or the rendered outputs, to adhere to prior knowledge (geometric, semantic, or statistical) rather than relying solely on pixel-wise alignment with ground truth images.

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

### [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)
This family defines the neural network structures and input-output mappings that approximate the continuous radiance field function $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to volume density $\sigma$ and color $\mathbf{c}$ [Parent]. It differs from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the internal topology of the network and the representation of the scene data, rather than the numerical integration of the rendering equation. It sets itself apart from [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md) by addressing the global mapping capacity and feature encoding, whereas partitioning methods address scalability through domain decomposition.

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
This family of methods accelerates NeRF training and rendering by reducing the computational cost of evaluating the neural radiance field. It addresses the bottleneck of querying high-dimensional MLPs for every sample point along every ray. Unlike standard uniform or adaptive sampling over the entire scene bounding box, these methods restrict evaluation to sparse structures (visual hulls) or use lightweight approximations (caches) to predict radiance, thereby reducing the number of expensive neural network queries.

### [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md)
This family of methods addresses the problem of optimizing camera parameters (extrinsics and intrinsics) simultaneously with Neural Radiance Field (NeRF) weights, eliminating the reliance on pre-computed Structure-from-Motion (SfM) pipelines like COLMAP. By treating camera poses as learnable parameters, these methods enable 3D reconstruction from sparse or uncalibrated image sets. What sets this family apart from siblings like [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md) is the explicit optimization of discrete camera poses per image rather than continuous motion functions, and from [Camera Preconditioning](12545-camera-preconditioning.md) by focusing on the joint gradient-based update of poses and scene geometry rather than preconditioning the optimizer for faster convergence.

### [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md)
This family of methods integrates pre-trained Monocular Depth Estimation (MDE) networks into the NeRF training loop to resolve the shape-radiance ambiguity and scale uncertainty inherent in sparse-view or single-view reconstruction. Unlike standard NeRF which relies solely on photometric consistency, these methods use MDE outputs as geometric priors to guide volume sampling, regularize density distributions, or directly adapt the MDE network to the scene's absolute geometry. They are distinguished from sibling methods like [DäRF](12331-d-rf.md) by their specific strategies for handling the relative nature of monocular depth, ranging from sparse prior fine-tuning to distribution-based Earth Mover’s Distance supervision [11911-depth-guided-nerf-training-via-earth-mover-s-distance.md].

### [NeRF Editing and Inpainting Procedures](12494-nerf-editing-and-inpainting-procedures.md)
This family of methods modifies a pre-trained Neural Radiance Field (NeRF) to remove unwanted objects, inpaint occluded regions, or alter semantic content without retraining the entire scene from scratch. They solve the problem of localized 3D editing, addressing view-dependent artifacts and geometric inconsistencies that arise when modifying specific regions. Unlike global retraining, these methods decouple editing logic from global parameter updates, using techniques such as feature field distillation, diffusion-guided pseudo-observations, or adaptive view selection to ensure multi-view consistency.

### [FlipNeRF Training Pipeline](12501-flipnerf-training-pipeline.md)
FlipNeRF is a training procedure for few-shot novel view synthesis that enhances the mip-NeRF representation by introducing flipped reflection rays into the training loop. It distinguishes itself from standard NeRF pipelines by enforcing consistency between original and geometrically flipped rays through specific uncertainty-aware and bottleneck feature losses, rather than relying solely on color reconstruction.

### [NeRF Ensemble with Conjoint Pruning](12608-nerf-ensemble-with-conjoint-pruning.md)
NeRF Ensemble with Conjoint Pruning is a training strategy that improves novel view synthesis quality by combining two Neural Radiance Field (NeRF) models of different grid resolutions, specifically using the Direct Voxel Grid Optimization (DVGO) architecture. It addresses the high computational requirements and optimization difficulties of fine-grained voxel grids by achieving superior performance compared to single models while maintaining a similar parameter count and memory footprint. Unlike vanilla ensembling, which may yield suboptimal results, this method employs a specific three-stage process involving independent training, fine-tuning, and conjoint pruning to prevent performance degradation.

### [MLI-NeRF](12709-mli-nerf.md)
MLI-NeRF is a two-stage optimization procedure that learns an intrinsic-aware Neural Radiance Field (NeRF) capable of novel view synthesis, relighting, and intrinsic decomposition (separating reflectance and shading) without relying on statistical priors or ground-truth intrinsic labels. It differs from standard NeRF variants by explicitly modeling multiple light sources to generate physics-based pseudo-labels, and it differs from prior intrinsic NeRF methods (like [IntrinsicNeRF](10253-nerv-training.md)) by leveraging 3D geometric constraints and multi-light visibility rather than chromaticity or semantic statistical constraints. The method solves the ill-posed nature of inverse rendering by decoupling the geometry learning (Stage 1) from the intrinsic component separation (Stage 2), using pseudo-labels derived from the Stage 1 model to guide the decomposition.

### [Diffusion-guided Optimization](13230-diffusion-guided-optimization.md)
Diffusion-guided optimization methods enhance Neural Radiance Field (NeRF) training by integrating pretrained 2D or 3D diffusion models as priors to regularize geometry, fill missing views, or generate novel 3D content from text. Unlike standard NeRF training which relies solely on photometric reconstruction losses from observed images, these methods leverage the generative knowledge of diffusion models to constrain the optimization landscape. They are distinguished from sibling methods like [KNN-NeRF](8672-knn-nerf.md) or [Cross-view Self-enhancement](13434-cross-view-self-enhancement.md) by their explicit use of score distillation or latent diffusion losses rather than nearest-neighbor statistics or self-supervised consistency alone.

### [Application-Specific Methods](13294-application-specific-methods.md)
Application-Specific Methods adapt the general NeRF training loop to solve domain-specific inverse problems by modifying the neural field architecture, loss functions, and optimization strategy. Unlike standard NeRF, which optimizes a single MLP for photometric reconstruction, these pipelines decompose fields to isolate physical properties, enforce domain priors (geometric, semantic, or physical), and employ iterative multi-stage optimization to handle complex constraints.

### [Cross-view Self-enhancement](13434-cross-view-self-enhancement.md)
Cross-view Self-enhancement (CSE) is a training strategy for semantic NeRFs that replaces view-inconsistent 2D semantic labels (e.g., from CLIP) with 3D-consistent pseudo-maps rendered from the model itself. It addresses the ambiguity where 2D vision-language models fail to maintain consistent object identities across diverse perspectives, which degrades multi-view consistent rendering. Unlike methods that rely solely on external supervision, CSE leverages the inherent 3D consistency of NeRFs to iteratively refine and synthesize semantic supervision, ensuring that the semantic field learns coherent structures across all views.

### [NeRF-Art](13438-nerf-art.md)
NeRF-Art is a text-guided method for stylizing pre-trained Neural Radiance Fields (NeRFs) by simultaneously altering both appearance and geometry without requiring mesh guidance. It distinguishes itself from prior approaches by employing a hybrid global-local contrastive learning strategy based on CLIP, which ensures strong style adherence and cross-view consistency, addressing the insufficiency of direct CLIP similarity constraints.

### [Neural Implicit Surfaces with PBR](13481-neural-implicit-surfaces-with-pbr.md)
This method integrates a Signed Distance Function (SDF) with a Physically-Based Rendering (PBR) material model to achieve accurate scene representation, explicitly separating specular and diffuse reflections. It differs from standard NeRF by deriving normals from the SDF gradient rather than direct optimization, and from basic SDF methods by employing a precomputed BRDF LUT for specular calculation. It builds on [NeRF Training Loop](970-nerf-training-loop.md) but replaces the generic radiance field with a geometry-aware implicit surface representation.

### [Block-NeRF](13537-block-nerf.md)
Block-NeRF is a method for city-scale NeRF-based reconstruction from street-level images that handles transient appearance and moving objects by dividing neighborhoods into overlapping blocks. It scales NeRF to city-scale scenes by spatially decomposing the environment into manageable blocks, training independent NeRFs per block, and compositing them at render time. It sets itself apart from standard NeRF by enabling the reconstruction of large outdoor environments (e.g., San Francisco's Mission Bay) that exceed the memory and computational limits of a single global model, while specifically addressing the challenge of transient objects (pedestrians, cars) common in street-level imagery.

### [NeRF in the Wild](13541-nerf-in-the-wild.md)
NeRF in the Wild (NeRF-W) extends standard neural radiance fields to handle unconstrained photo collections by modeling per-image appearance variations and transient objects. It addresses the limitations of original NeRF, which assumes static geometry and consistent lighting, by introducing latent embeddings for each input image.

### [ContraNeRF](13659-contranerf.md)
ContraNeRF is a generalizable neural radiance field method designed to improve synthetic-to-real novel view synthesis by introducing geometry-aware contrastive learning to learn multi-view consistent features with geometric constraints, and cross-view attention to enhance geometry perception. It sets itself apart from vanilla NeRF by eliminating per-scene optimization, allowing it to generalize to unseen scenes, and distinguishing itself from other generalizable methods by explicitly addressing the sharp but inaccurate volume density artifacts common in synthetic-to-real transfer through contrastive geometry constraints.

### [RoGUENeRF](13731-roguenerf.md)
RoGUENeRF is a post-processing enhancer for trained Neural Radiance Fields (NeRF) that improves visual quality and high-frequency texture fidelity while maintaining view-consistency and robustness to inaccurate camera calibration. It distinguishes itself from sibling methods like [NeRFLiX](12501-flipnerf-training-pipeline.md) by requiring significantly less training data (operating effectively on small real-world datasets like LLFF without degradation) and offering faster inference speeds by avoiding test-time augmentation (TTA). It builds on [NeRF Training Loop](970-nerf-training-loop.md) by taking rendered RGB and depth maps as input rather than optimizing the NeRF weights directly.

### [PlanarNeRF](13791-planarnerf.md)
PlanarNeRF is an online learning framework for detecting dense 3D planar primitives from monocular RGB and depth sequences. It combines plane segmentation rendering, an efficient plane fitting module, and a global memory bank to learn plane primitives in an online fashion using neural field representation, appearance, and geometry knowledge. It distinguishes itself from standard NeRF training loops by explicitly modeling planar instances via a dedicated rendering branch and maintaining a global memory bank for consistent plane tracking across views.

## Operating characteristics

*   **Time Complexity:** The training loop is iterative, with complexity proportional to $N_{iters} \times N_{rand} \times N_{samples}$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), and $N_{samples}$ is the number of samples per ray [S5, S14, unofficial].
*   **Memory Complexity:** Memory usage is dominated by the storage of intermediate states for backpropagation, which can be $O(kn)$ where $n$ is pixels and $k$ is samples, leading to high VRAM requirements [44-radiance-field-optimization.md]. Chunking rays helps mitigate this [S5, unofficial].
*   **Measured Performance:** In standard implementations (e.g., Blender dataset), training can take several hours to days depending on resolution and iteration count (e.g., 100,000 iterations) [S11, unofficial]. PSNR typically increases over epochs, e.g., from ~10 dB to ~19 dB over 20 epochs in some Keras implementations [S12].
*   **Scaling:** Performance scales with batch size, but larger batches may hinder data loading performance and prevent full GPU occupancy, while smaller batches cap iterations per second [S15, unofficial]. A batch size of $2^{10}$ (1024) is often chosen as a trade-off [S15, unofficial].
*   **Strengths:** Stochastic optimization allows for efficient training on large datasets by sampling subsets of rays [S5, unofficial]. The differentiable nature allows for end-to-end learning of geometry and appearance.
*   **Limitations:** High computational cost due to repeated MLP evaluations and volume integration. Sensitive to hyperparameters like learning rate and batch size. Can suffer from overfitting or "floaters" without regularization [S1].
*   **When to Use:** Suitable for high-quality novel view synthesis from multi-view images where computational resources are available.
*   **When Not to Use:** Not suitable for real-time applications or very large-scale scenes without acceleration techniques (e.g., [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)).

## Background

The NeRF Training Loop originates from the seminal paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" by Mildenhall et al. (2020) [S5, S11, unofficial]. It was developed to address the limitations of traditional 3D reconstruction methods by using a neural network to implicitly represent a 3D scene as a continuous volumetric function. The training procedure involves optimizing this function by minimizing the difference between rendered and observed images. Subsequent works have extended this loop with various optimizations, such as coarse-fine sampling [S11, unofficial], positional encoding [S20], and different acceleration structures [S7, S15, unofficial].

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
[S25] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
