# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth observations by updating the parameters of a continuous volumetric radiance field [S5, S6, S14, unofficial]. It solves the inverse problem of recovering scene geometry and appearance from 2D images by leveraging the differentiability of the volume rendering equation [44-radiance-field-optimization.md]. It sets itself apart from evaluation rendering by using stochastic ray sampling and perturbation to facilitate gradient-based optimization, whereas evaluation uses deterministic rendering for reproducible metrics [S2, S9, unofficial].

## How it works
The training loop minimizes the Mean Squared Error (MSE) between the predicted pixel colors $\hat{C}$ and the ground-truth colors $C_{gt}$ by iteratively sampling rays, rendering them via the differentiable forward pass, and updating network weights via gradient descent [S5, S6, S14, unofficial].

### Inputs and Outputs
*   **Inputs:** A dataset of $N$ images with known camera poses (extrinsics $C2W$ and intrinsics $K$), image dimensions $(H, W)$, and near/far bounds $(near, far)$ for ray marching [S2, S5, S14, unofficial].
*   **Outputs:** Optimized neural network weights $\Theta$ representing the radiance field $F_\Theta(\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ [Parent].

### Data Structures
*   **Rays Array:** Pre-computed array of shape $[(N-1)HW, 3, 3]$ containing flattened pixel indices, ray origins, ray directions, and target RGB colors [S5, S14, unofficial].
    *   Axis 0: Flattened pixel index.
    *   Axis 1: Channel type ($0$: origin, $1$: direction, $2$: target RGB).
    *   Axis 2: XYZ/RGB components.
    *   Type: `float32` [S5, S14, unofficial].
*   **Checkpoint:** Dictionary containing model state (`coarse`, `fine` or single `model`), optimizer state, and step/epoch counters [S10, S11, unofficial].

### Numbered Steps

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the NeRF model(s) (coarse and/or fine MLPs) and background model if applicable [S11, S15, unofficial].
    *   Initialize the optimizer (typically Adam) and learning rate scheduler (Exponential Decay or Cosine Annealing) [S5, S11, S17, unofficial].
    *   If resuming, load checkpoint and set scheduler last epoch to `start_epoch - 1` [S9, unofficial].

2.  **Ray Sampling (Per Iteration):**
    *   Select a batch of $N_{rand}$ rays from the pre-computed rays array or randomly sample pixel coordinates from training images [S5, S14, unofficial].
    *   If `use_batching` is True, slice the pre-computed array; otherwise, generate rays on-the-fly [S14, unofficial].
    *   If `precrop_iters` is active, sample from a central crop of the image [S14, unofficial].
    *   Construct the ray bundle tensor of shape $[3, B, 3]$ (origin, direction, target) [S14, unofficial].

3.  **Forward Pass (Rendering):**
    *   Sample $K$ points along each ray within $[near, far]$ using uniform or stratified sampling [S5, S9, unofficial].
    *   Apply positional encoding to 3D positions and viewing directions [S20].
    *   Query the NeRF MLP(s) to obtain density $\sigma$ and color $\mathbf{c}$ for each sample point [S5, S6, unofficial].
    *   Compute transmittance $T_i$ and weights $w_i$ using the emission-absorption model [44-radiance-field-optimization.md].
    *   Composite the final color $\hat{C}(\mathbf{r})$ and depth [S5, S6, unofficial].
    *   *Note:* If a coarse network is used, render coarse predictions first, then sample additional points based on coarse weights for fine rendering [S5, S11, unofficial].

4.  **Loss Computation:**
    *   Compute the primary photometric loss:
        $$ \mathcal{L}_{rgb} = \frac{1}{B} \sum_{i=1}^B \| \hat{C}_i - C_{gt, i} \|^2 $$
        [S14, S17, unofficial].
    *   If coarse predictions are available, add coarse loss: $\mathcal{L}_{coarse} = \text{MSE}(\hat{C}_{coarse}, C_{gt})$ [S11, S18, unofficial].
    *   Add regularization terms if defined (e.g., sparsity, TV, Eikonal) [S7, unofficial][S8].
    *   Total Loss: $\mathcal{L} = \mathcal{L}_{rgb} + \lambda_{coarse}\mathcal{L}_{coarse} + \mathcal{L}_{reg}$ [S11, S18, unofficial].

5.  **Backpropagation and Update:**
    *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
    *   Compute gradients: `loss.backward()` [S18, unofficial].
    *   Update weights: `optimizer.step()` [S18, unofficial].
    *   Update learning rate according to the scheduler [S9, S18, unofficial].

6.  **Logging and Checkpointing:**
    *   Log loss and PSNR ($\text{PSNR} = -10 \log_{10}(\text{MSE})$) at intervals [S14, unofficial].
    *   Render validation images at intervals [S9, S11, unofficial].
    *   Save checkpoint at intervals [S9, S11, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def train_nerf_loop(model, optimizer, scheduler, dataloader, 
                    num_epochs, render_interval, checkpoint_interval, 
                    device='cuda'):
    """
    Standard NeRF training loop.
    
    Args:
        model: Neural Radiance Field model (MLP).
        optimizer: Optimizer (e.g., Adam).
        scheduler: Learning rate scheduler.
        dataloader: Iterator yielding (images, rays, poses) batches.
        num_epochs: Total training epochs.
        render_interval: Epochs between validation renders.
        checkpoint_interval: Epochs between checkpoint saves.
        device: Computation device.
    """
    mse_loss = nn.MSELoss()
    model.train()
    
    for epoch in range(num_epochs):
        epoch_loss = 0.0
        num_batches = 0
        
        for batch in dataloader:
            # 1. Data Loading
            # batch contains: images (H, W, 3), rays (N, 6) [origins, dirs], 
            # targets (N, 3), poses (N, 4, 4)
            images = batch['images'].to(device)
            rays = batch['rays'].to(device)
            targets = batch['targets'].to(device)
            
            # 2. Forward Pass (Ray Marching & Integration)
            # render_rays handles sampling, MLP query, and alpha compositing
            # Returns: rgb (N, 3), depth (N,), extras (dict with coarse rgb if applicable)
            outputs = render_rays(model, rays)
            predicted_rgb = outputs['rgb']
            
            # 3. Loss Computation
            # Primary photometric loss
            loss = mse_loss(predicted_rgb, targets)
            
            # Optional: Coarse loss if two-stage training is used
            if 'rgb_coarse' in outputs:
                loss += mse_loss(outputs['rgb_coarse'], targets)
                
            # Optional: Regularization (e.g., Eikonal, Sparsity)
            # loss += regularization_terms(model, outputs)

            # 4. Backpropagation
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            # 5. Logging
            epoch_loss += loss.item()
            num_batches += 1
            
            # Optional: Learning rate scheduling (if step-based)
            # scheduler.step() 

        # End of Epoch
        avg_loss = epoch_loss / num_batches
        psnr = -10 * torch.log10(torch.tensor(avg_loss)).item()
        print(f"Epoch {epoch}: Loss={avg_loss:.6f}, PSNR={psnr:.2f}")
        
        # 6. Validation and Checkpointing
        if epoch % render_interval == 0:
            render_validation_images(model, dataloader.val_dataset, device)
            
        if epoch % checkpoint_interval == 0:
            save_checkpoint({
                'epoch': epoch,
                'model_state_dict': model.state_dict(),
                'optimizer_state_dict': optimizer.state_dict(),
                'loss': avg_loss,
            }, checkpoint_path)
            
        # Scheduler step (if epoch-based)
        scheduler.step()

def render_rays(model, rays):
    """
    Differentiable forward pass for a batch of rays.
    rays: Tensor of shape (N, 6) where cols 0-2 are origins, 3-5 are directions.
    """
    origins = rays[:, :3]
    directions = rays[:, 3:6]
    
    # Normalize directions
    directions = directions / torch.norm(directions, dim=-1, keepdim=True)
    
    # Sample points along rays (Uniform or Stratified)
    # t_vals: (N, K)
    t_vals = torch.linspace(0., 1., 128, device=origins.device).unsqueeze(0)
    # Add perturbation for training stability [S5, S9, unofficial]
    t_rand = torch.rand_like(t_vals)
    t_vals = torch.sort(torch.cat([t_vals, t_rand], dim=-1), dim=-1).values
    
    # Map t_vals to depth range [near, far]
    near = 2.0
    far = 6.0
    z_vals = near * (1. - t_vals) + far * t_vals
    
    # Compute 3D points
    pts = origins.unsqueeze(1) + directions.unsqueeze(1) * z_vals.unsqueeze(2)
    
    # Query NeRF
    # model returns density (sigma) and color (c)
    # sigma: (N, K), c: (N, K, 3)
    sigma, c = model(pts, directions)
    
    # Compute weights using alpha compositing
    # delta_z = z_vals[:, 1:] - z_vals[:, :-1]
    # alpha = 1 - torch.exp(-sigma * delta_z)
    # T = torch.cumprod(1 - alpha + 1e-10, dim=1, exclusive=True)
    # weights = alpha * T
    # rgb = torch.sum(weights.unsqueeze(-1) * c, dim=1)
    
    # Simplified composite for pseudocode completeness
    # Note: In practice, this involves careful handling of boundaries and transmittance
    weights = compute_weights(sigma, z_vals)
    rgb = torch.sum(weights.unsqueeze(-1) * c, dim=1)
    
    return {'rgb': rgb}

def compute_weights(sigma, z_vals):
    """Compute rendering weights from density and sample depths."""
    # delta_z = torch.diff(z_vals, dim=1, prepend=torch.zeros_like(z_vals[:, :1]))
    # This is a placeholder for the actual volume rendering integration logic
    # which requires handling the last sample and boundary conditions.
    # For implementation, refer to the volume rendering equation in [44-radiance-field-optimization.md].
    raise NotImplementedError("Implement volume rendering integration based on standard NeRF code [S5, unofficial]")
```

### Parameters
*   **Learning Rate (`lr`):** Typically $5 \times 10^{-4}$ for Adam [S11, unofficial].
*   **Batch Size (`N_rand`):** Number of rays per iteration. Typical values: 1024 or 2048 [S5, S11, unofficial].
*   **Iterations (`n_iters`):** Total optimization steps. Default: 100,000 [S11, unofficial].
*   **Positional Encoding Frequencies:** `pos_freq` (position) default 10, `dir_freq` (direction) default 4 [S11, unofficial].
*   **Decay:** Exponential decay with `decay_rate=0.1` every `decay_steps` (e.g., 1000 steps) [S5, S18, unofficial].

### Edge Cases and Pitfalls
*   **VRAM Overflow:** Rendering large batches can exceed GPU memory. Mitigate by chunking rays during the forward pass (`chunk` argument) [S5, S14, unofficial].
*   **Gradient Freeing:** Higher-order derivatives (for logging gradients/Laplacians) must be computed *before* `loss.backward()` to avoid freeing the autograd graph [S3, unofficial].
*   **Resuming Training:** When loading checkpoints, ensure the learning rate scheduler's state is reset to `start_epoch - 1` to prevent incorrect LR schedules [S9, unofficial].
*   **Instability in Growth:** In dynamic growth methods (e.g., Recursive-NeRF), initializing new sub-network weights to match the parent prevents density collapse [S1].
*   **Empty Checkpoints:** Loading a checkpoint with step 0 should be treated as an error or fresh start [S15, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images and depth maps from a Neural Radiance Field (NeRF) during optimization. It sets itself apart from evaluation rendering by using stochastic sampling with perturbation and adaptive step sizes to facilitate gradient-based optimization, whereas evaluation uses deterministic rendering for reproducible metrics.

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
This family of methods modifies a pre-trained Neural Radiance Field (NeRF) to remove, insert, or alter objects and textures without retraining the entire scene from scratch. It solves the problem of applying semantic or geometric transformations to specific regions while maintaining multi-view consistency. It differs from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on post-training modification or specialized optimization loops for editing, rather than initial scene reconstruction.

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

### [Progressive Camera Placement for NeRF](13844-progressive-camera-placement-for-nerf.md)
This method is a greedy optimization algorithm that selects camera positions to maximize a reconstruction quality metric $E(C, B_{32})$, which combines observation frequency and angular uniformity, to improve NeRF reconstruction quality with fewer images. Unlike uncertainty-based methods (e.g., ActiveNeRF), this method does not require additional image acquisition to estimate reconstruction quality, making it cheaper to compute. It builds on [NeRF Training Loop](970-nerf-training-loop.md) by dynamically selecting the next best view based on the current state of the occupancy grid and camera set.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, multiple jittered rays are sampled uniformly at random through the spatial footprint of the pixel, rendered with NeRF, and averaged to predict the color in the loss function [S21]. This approach relies on 'oracle' scale information that does not exist for most datasets and performs worse than mip-NeRF while being significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S22, unofficial]. This two-stage approach is consistently effective where training from scratch often fails [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### JaxNeRF
A re-implementation of the original NeRF in JAX, used as a benchmark comparison for early works seeking to improve on training and rendering speed [S24]. It is slightly faster and more suited for distributed computing than the original TensorFlow implementation of NeRF [S24].

### Gradient Detachment in Training
A training strategy where the object field component is detached from the backbone to prevent gradient backpropagation to the rendering part, ensuring independent optimization [S25, unofficial]. With detachment, gradients do not backpropagate to the NeRF backbone; without it, gradients from the object field component backpropagate to the backbone, causing mutual influence and a significant drop in rendering quality [S25, unofficial]. The optimal hyper-parameter for empty 3D point losses ($\Delta d$) is found to be 0.05 meters [S25, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot K)$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), and $K$ is the number of samples per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot K)$ for storing intermediate rendering states during the forward pass, plus $O(\Theta)$ for model parameters [44-radiance-field-optimization.md]. Chunking reduces peak VRAM usage [S5, unofficial].
*   **Performance:** Standard NeRF training on Blender datasets (800x800) takes approximately 100,000 iterations [S11, unofficial]. On LLFF datasets, training may take 250 epochs on 128x128 images [S4, unofficial].
*   **Strengths:** Simple to implement, differentiable end-to-end, produces high-quality novel views [S5, S14, unofficial].
*   **Limitations:** Slow convergence due to uniform sampling, high memory usage for large scenes, prone to "floaters" and overfitting in sparse views [S1, S9394-sparse-view-and-initialization-methods.md].
*   **When to Use:** When high-quality view synthesis from dense multi-view images is required and computational resources are sufficient [S5, S11, unofficial].
*   **When Not to Use:** For real-time applications, large-scale scenes without acceleration structures, or sparse-view scenarios without additional regularization [S1, S11405-acceleration-and-sparse-structures.md].

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial]. It established the standard protocol for optimizing continuous volumetric representations using stochastic gradient descent on photometric loss. The method builds on earlier work in neural implicit representations and differentiable rendering [S24]. It has become the foundational training procedure for numerous NeRF variants and extensions [S1][S2, S4, S5, S6, S7, S9, S10, S11, S13, S14, S15, S17, S18, S19, unofficial][S20, S21][S22, unofficial][S23, S24][S25, unofficial].

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
