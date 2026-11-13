# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview

The NeRF Training Loop is the iterative optimization procedure that trains a coordinate-based neural network (MLP or SIREN) to represent a 3D scene as a continuous volumetric radiance field [S3, S5, S14, unofficial]. It solves the problem of reconstructing novel views of a scene from a set of input images by minimizing the difference between rendered and ground-truth pixel colors [S5, S6, unofficial]. It sets itself apart from evaluation rendering by using stochastic ray sampling and gradient-based weight updates rather than deterministic full-image rendering [S2, S9, unofficial]. It builds on [Optimization Loops and Training Procedures](5622-optimization-loops-and-training-procedures.md) by implementing the specific forward pass and loss computation for neural radiance fields.

## How it works

**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics) and intrinsics (focal length, principal point, resolution) [S2, S4, S5, unofficial].
*   **Outputs:** A trained neural network model (state dictionary) containing weights for density $\sigma$ and color $\mathbf{c}$, and optionally a learning rate scheduler state [S11, S15, unofficial].

**Data Structures:**
*   **Rays:** A batch of rays is represented as a tensor of shape $[B, 3, 3]$ or $[B, 6]$ depending on implementation, where $B$ is the batch size (number of rays) [S5, S14, unofficial]. The channels typically correspond to:
    1.  Ray Origin $(x, y, z)$
    2.  Ray Direction $(x, y, z)$
    3.  Target Ground Truth Color $(R, G, B)$ [S5, S14, unofficial].
    *   In some implementations, viewing directions are appended if view-dependent effects are enabled [S6, unofficial].
*   **Pre-computed Rays:** To accelerate training, rays are often pre-computed and stored in a flat array of shape $[(N-1) \cdot H \cdot W, 3, 3]$ (float32) [S5, S14, unofficial]. This array is shuffled at the start of each epoch to ensure random sampling [S11, unofficial].
*   **Checkpoint:** A dictionary containing:
    *   `model`: State dict of the neural network [S11, unofficial].
    *   `optim`: State dict of the optimizer [S11, unofficial].
    *   `step` or `epoch`: Current iteration count [S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the neural network (e.g., NeRF MLP) with specific architecture parameters (layers, hidden units) [S4, S15, unofficial].
    *   Initialize the optimizer (typically Adam or AdamW) with a learning rate $lr$ (default $5 \times 10^{-4}$ or $0.01$) [S5, S11, S17, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   If resuming, load checkpoint and set scheduler last epoch to `current_epoch - 1` [S9, unofficial].

2.  **Ray Sampling:**
    *   For each training iteration, sample a batch of $N_{rand}$ rays (default 1024 or 2048) [S5, S14, unofficial].
    *   Rays are selected by randomly choosing pixel coordinates from the training images [S4, S7, unofficial].
    *   Convert pixel coordinates to 3D ray origins and directions using camera intrinsics [S4, S5, unofficial].
    *   Retrieve the ground truth RGB color for each pixel [S4, S5, unofficial].
    *   *Edge Case:* If using batching with a pre-computed array, increment the batch index and shuffle the array when exhausted [S5, S14, unofficial].
    *   *Edge Case:* For the first $precrop\_iters$ iterations, sample only from a central crop of the image (fraction $precrop\_frac$) to stabilize early training [S2, S14, unofficial].

3.  **Forward Pass (Rendering):**
    *   For each ray in the batch, sample $K$ points along the ray segment between near and far bounds [S5, S13, unofficial].
        *   Bounds are dataset-specific: e.g., Blender uses fixed $[2.0, 6.0]$, LLFF uses computed bounding depths [S2, S13, unofficial].
        *   Sampling can be uniform or stratified (perturbed) to reduce aliasing [S2, S5, unofficial].
    *   Query the neural network at each point $(\mathbf{x}, \mathbf{d})$ to obtain density $\sigma$ and color $\mathbf{c}$ [S5, S6, unofficial].
        *   Inputs are often encoded using positional encoding $\gamma(p)$ [S20].
    *   Compute transmittance $T_i$ and weights $w_i$ using the emission-absorption model [S5, S6, unofficial]:
        $$
        T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)
        $$
        $$
        w_i = T_i (1 - \exp(-\sigma_i \delta_i))
        $$
    *   Composite the final color $\hat{C}$:
        $$
        \hat{C} = \sum_{i=1}^K w_i \mathbf{c}_i
        $$
    *   *Coarse-to-Fine:* If using a two-stage model, render a coarse prediction, sample additional points based on coarse weights, and render a fine prediction [S5, S11, unofficial].

4.  **Loss Computation:**
    *   Compute the Mean Squared Error (MSE) between the predicted color $\hat{C}$ and the ground truth color $C_{gt}$ [S3, S14, unofficial]:
        $$
        \mathcal{L}_{MSE} = \frac{1}{B} \sum_{b=1}^B \|\hat{C}_b - C_{gt,b}\|^2
        $$
    *   *Coarse Loss:* If a coarse network output $\hat{C}_{coarse}$ is available, add its MSE loss to the total loss [S11, S18, unofficial]:
        $$
        \mathcal{L}_{total} = \mathcal{L}_{coarse} + \mathcal{L}_{fine}
        $$
    *   *Regularization:* Add optional terms such as sparsity loss, Total Variation (TV) loss, or Eikonal loss [S7, unofficial][S8].

5.  **Backpropagation and Update:**
    *   Zero the gradients of the optimizer [S18, unofficial].
    *   Compute gradients via backpropagation: `loss.backward()` [S3, S18, unofficial].
    *   *Pitfall:* Higher-order derivative logging (e.g., gradient/laplacian for visualization) must occur *before* `loss.backward()` as the autograd graph is freed [S3, unofficial].
    *   Step the optimizer: `optimizer.step()` [S3, S18, unofficial].
    *   Update the learning rate according to the scheduler [S9, S18, unofficial].
        *   Exponential Decay: $lr_{new} = lr_{old} \times decay\_rate^{(step / decay\_steps)}$ [S5, S18, unofficial].

6.  **Logging and Checkpointing:**
    *   Log loss and PSNR ($PSNR = -10 \log_{10}(MSE)$) at regular intervals (e.g., every 100 steps) [S3, S11, unofficial].
    *   Render validation images at regular intervals [S9, S11, unofficial].
    *   Save model checkpoint at regular intervals (e.g., every 10,000 steps) [S2, S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def train_nerf_loop(model, optimizer, scheduler, dataloader, num_epochs, 
                    batch_size=1024, device='cuda'):
    """
    Standard NeRF training loop.
    
    Args:
        model: Neural Radiance Field model (MLP).
        optimizer: Optimizer (e.g., Adam).
        scheduler: Learning rate scheduler.
        dataloader: Iterator yielding (rays, targets) batches.
        num_epochs: Number of training epochs.
        batch_size: Number of rays per batch.
        device: Computation device.
    """
    mse_loss = nn.MSELoss()
    
    for epoch in range(num_epochs):
        model.train()
        for batch_idx, (rays, targets) in enumerate(dataloader):
            # 1. Move data to device
            rays = rays.to(device)      # Shape: [B, 6] or [B, 3, 3]
            targets = targets.to(device) # Shape: [B, 3]
            
            # 2. Forward Pass
            # Note: The specific rendering logic (ray marching, volume integration)
            # is encapsulated within the model's forward method or a helper.
            # For a standard NeRF, this involves sampling points, querying the MLP,
            # and compositing colors.
            predictions = model(rays)   # Returns predicted RGB colors
            
            # 3. Loss Computation
            loss = mse_loss(predictions, targets)
            
            # Optional: Add coarse loss if two-stage training
            # if 'rgb_coarse' in predictions:
            #     loss += mse_loss(predictions['rgb_coarse'], targets)

            # 4. Backpropagation
            optimizer.zero_grad()
            loss.backward()
            
            # 5. Optimization Step
            optimizer.step()
            
            # 6. Learning Rate Update
            scheduler.step()
            
            # 7. Logging (every N steps)
            if batch_idx % 100 == 0:
                psnr = -10.0 * torch.log10(loss)
                print(f"Epoch {epoch}, Batch {batch_idx}, Loss: {loss.item():.6f}, PSNR: {psnr.item():.2f}")

        # 8. Checkpointing (every N epochs)
        if epoch % 10 == 0:
            torch.save({
                'epoch': epoch,
                'model_state_dict': model.state_dict(),
                'optimizer_state_dict': optimizer.state_dict(),
                'loss': loss.item(),
            }, f'checkpoint_epoch_{epoch}.pth')

    return model
```

**Parameters:**
*   `lr`: Learning rate. Default: $5 \times 10^{-4}$ [S11, unofficial] or $0.01$ [S17, unofficial].
*   `batch_size` ($N_{rand}$): Number of rays sampled per iteration. Default: 1024 [S11, unofficial], 2048 [S5, unofficial], or 4096 [S17, unofficial].
*   `num_epochs` / `n_iters`: Total training duration. Default: 100,000 iterations [S11, unofficial] or 10 epochs [S17, unofficial].
*   `decay_rate`: Factor for exponential decay. Default: 0.1 [S6, S18, unofficial].
*   `decay_steps`: Steps between decays. Default: `args.lrate_decay * 1000` [S5, S18, unofficial].
*   `chunk`: Maximum rays processed per forward pass to limit VRAM. Default: varies [S5, unofficial].
*   `perturb`: Jitter amount for stratified sampling. Default: 0.0 (no jitter) or 1.0 [S2, unofficial].
*   `multires`: Log2 of max frequency for positional encoding of 3D location. Default: 10 [S2, unofficial].
*   `multires_views`: Log2 of max frequency for positional encoding of viewing direction. Default: 4 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Large batch sizes or high-resolution images can exceed GPU memory. Mitigated by chunking rays during the forward pass [S5, unofficial].
*   **Instability in Growing Networks:** When dynamically growing sub-networks (e.g., in Recursive-NeRF), initializing alpha weights to match the parent prevents density collapse [S1].
*   **Checkpoint Resumption:** When loading a checkpoint, the scheduler's last epoch must be adjusted to `start_epoch - 1` to prevent learning rate jumps [S9, unofficial].
*   **Empty Checkpoints:** If a checkpoint is loaded but the step counter is 0, it is treated as invalid [S15, unofficial].
*   **Gradient Freeing:** Higher-order derivatives (for logging/visualization) must be computed before `loss.backward()` [S3, unofficial].

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

### [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)
Scene Representation and Architecture defines the neural network structures that map 6D input coordinates (3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$) to radiance field properties (density $\sigma$ and color $\mathbf{c}$) [Parent]. While the canonical NeRF uses a simple two-MLP structure, this family encompasses modifications to the network topology, input representations, and output heads to address specific challenges such as single-view synthesis, dynamic scenes, high-resolution rendering, and semantic editing. These architectures differ from their siblings by introducing specialized components like diffusion priors, multi-scale grids, or physical material decompositions to enhance reconstruction quality, efficiency, or controllability.

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
This family of methods accelerates NeRF training and rendering by reducing the volume of space queried by the neural network, addressing the computational bottleneck of sampling empty space or redundant regions in large scenes. Unlike [Zip-NeRF Training Procedure](9538-zip-nerf-training-procedure.md) which uses caching to reduce gradient variance, or [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) which varies density along rays, these methods explicitly restrict the spatial domain of queries using geometric hulls, spatial partitioning, or block decomposition.

### [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md)
This family of methods addresses the problem of optimizing camera parameters (extrinsics and intrinsics) simultaneously with Neural Radiance Field (NeRF) weights, eliminating the reliance on pre-computed Structure-from-Motion (SfM) pipelines like COLMAP. By treating camera poses as learnable parameters, these methods enable 3D reconstruction from sparse or uncalibrated image sets. What sets this family apart from siblings like [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md) is the explicit optimization of discrete camera poses per image rather than continuous motion functions, and from [Camera Preconditioning](12545-camera-preconditioning.md) by focusing on the joint gradient-based update of poses and scene geometry rather than preconditioning the optimizer for faster convergence.

### [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md)
This family of methods integrates pre-trained Monocular Depth Estimation (MDE) networks into the NeRF training loop to resolve the shape-radiance ambiguity and scale uncertainty inherent in sparse-view or single-view reconstruction. Unlike standard NeRF which relies solely on photometric consistency, these methods use MDE outputs as geometric priors to guide volume sampling, regularize density distributions, or directly adapt the MDE network to the scene's absolute geometry. They are distinguished from sibling methods like [DäRF](12331-d-rf.md) by their specific strategies for handling the relative nature of monocular depth, ranging from sparse prior fine-tuning to distribution-based Earth Mover’s Distance supervision [11911-depth-guided-nerf-training-via-earth-mover-s-distance.md].

### [NeRF Editing and Inpainting Procedures](12494-nerf-editing-and-inpainting-procedures.md)
NeRF Editing and Inpainting Procedures modify a pre-trained Neural Radiance Field (NeRF) to remove, add, or alter objects and textures while maintaining multi-view consistency. They solve the problem of 3D editing consistency, which is difficult because standard NeRFs entangle geometry and appearance, and 2D editing tools lack 3D coherence. These methods are distinguished from standard training by their focus on modifying an existing representation rather than learning it from scratch, often leveraging 2D priors (diffusion, inpainting) or semantic features to guide the optimization or rendering of the radiance field.

### [FlipNeRF Training Pipeline](12501-flipnerf-training-pipeline.md)
FlipNeRF is a training procedure for few-shot novel view synthesis that enhances the mip-NeRF representation by introducing flipped reflection rays into the training loop. It distinguishes itself from standard NeRF pipelines by enforcing consistency between original and geometrically flipped rays through specific uncertainty-aware and bottleneck feature losses, rather than relying solely on color reconstruction.

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

### [NeRF in the Wild](13541-nerf-in-the-wild.md)
NeRF in the Wild (NeRF-W) extends standard neural radiance fields to handle unconstrained photo collections by modeling per-image appearance variations and transient objects. It addresses the limitations of original NeRF, which assumes static geometry and consistent lighting, by introducing latent embeddings for each input image.

### [ContraNeRF](13659-contranerf.md)
ContraNeRF is a generalizable neural radiance field method designed to improve synthetic-to-real novel view synthesis by introducing geometry-aware contrastive learning to learn multi-view consistent features with geometric constraints, and cross-view attention to enhance geometry perception. It sets itself apart from vanilla NeRF by eliminating per-scene optimization, allowing it to generalize to unseen scenes, and distinguishing itself from other generalizable methods by explicitly addressing the sharp but inaccurate volume density artifacts common in synthetic-to-real transfer through contrastive geometry constraints.

### [PlanarNeRF](13791-planarnerf.md)
PlanarNeRF is an online learning framework for detecting dense 3D planar primitives from monocular RGB and depth sequences. It combines plane segmentation rendering, an efficient plane fitting module, and a global memory bank to learn plane primitives in an online fashion using neural field representation, appearance, and geometry knowledge. It distinguishes itself from standard NeRF training loops by explicitly modeling planar instances via a dedicated rendering branch and maintaining a global memory bank for consistent plane tracking across views.

### [Progressive Camera Placement for NeRF](13844-progressive-camera-placement-for-nerf.md)
This method is a greedy optimization algorithm that selects camera positions to maximize a reconstruction quality metric $E(C, B_{32})$, which combines observation frequency and angular uniformity, to improve NeRF reconstruction quality with fewer images. Unlike uncertainty-based methods (e.g., ActiveNeRF), this method does not require additional image acquisition to estimate reconstruction quality, making it cheaper to compute. It builds on [NeRF Training Loop](970-nerf-training-loop.md) by dynamically selecting the next best view based on the current state of the occupancy grid and camera set.

### [Neural Refractive Radiance Field (NeRRF)](13882-neural-refractive-radiance-field-nerrf.md)
Neural Refractive Radiance Field (NeRRF) is an inverse rendering method for transparent scenes that reconstructs surface geometry and a radiance field from multi-view images without prior knowledge of the object's Index of Refraction (IOR). It solves the problem of incorrect radiance field learning and visual artifacts (such as holes and color errors) that occur when standard NeRF or SDF-based methods like NeuS attempt to model refraction with a fixed or unknown IOR. NeRRF distinguishes itself by automatically searching for the optimal IOR during training and correcting the calculation of refractive effects, specifically enabling accurate rendering of foreground darkening caused by refraction at object edges.

### [NeRF-Neural ODE](13904-nerf-neural-ode.md)
NeRF-Neural ODE replaces the discrete quadrature summation of standard NeRF with a continuous Neural Ordinary Differential Equation (Neural ODE) solver to integrate radiance and transmittance along rays. It differs from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by treating the ray integration as a continuous dynamical system solved via adaptive ODE solvers rather than fixed-step numerical integration.

### [OV-NeRF](13995-ov-nerf.md)
OV-NeRF is a method for accurate NeRF-based open-vocabulary 3D semantic understanding that leverages pre-trained vision and language foundation models (CLIP and SAM) to enhance semantic field learning through single-view and cross-view strategies. It addresses the limitations of prior methods (e.g., LERF, 3DOVS) that directly use noisy and view-inconsistent CLIP relevancy maps for training. OV-NeRF introduces Region Semantic Ranking (RSR) regularization to rectify coarse boundaries in single views using SAM mask proposals, and Cross-view Self-enhancement (CSE) to enforce view-consistent supervision using refined 3D semantics from the trained field itself.

### [LOLNeRF](14071-lolnerf.md)
LOLNeRF is a method for single-shot view synthesis of human faces that learns a generative 3D model from single-view images without relying on explicit geometric information like depth or point clouds. It differs from adversarial approaches like $\pi$-GAN by using Generative Latent Optimization (GLO) to learn a per-image table of latent codes, allowing it to reconstruct specific training images with high fidelity while avoiding the resolution limits imposed by GAN discriminators.

### [Robustness and Generalization](14094-robustness-and-generalization.md)
This family encompasses methods that improve the generalization of Neural Radiance Fields (NeRFs) to unseen views and enhance visual fidelity by addressing inconsistencies in geometry and texture. Unlike standard training which minimizes photometric error on training data, these methods introduce mechanisms to prevent overfitting, handle calibration errors, or refine features post-training. [Aug-NeRF](10294-aug-nerf.md) uses adversarial perturbations during training to regularize the model, [NeRF Ensemble with Conjoint Pruning](12608-nerf-ensemble-with-conjoint-pruning.md) combines multiple models to reduce variance, and [RoGUENeRF](13731-roguenerf.md) applies geometric reprojection and refinement to restore high-frequency details in novel views.

### [ShadowNeuS](14132-shadowneus.md)
ShadowNeuS reconstructs a neural Signed Distance Field (SDF) of a scene from single-view images under multiple lighting conditions by supervising shadow rays (rays between the light source and the scene) rather than just camera rays. The core problem solved is that NeRF does not model rays from the scene to the light source, missing shadow information which contains geometry cues via light visibility (shadows). Unlike NeRF which uses multi-view cameras as ray sensors, this method uses a single-view camera and shadow rays to provide supervision, making it applicable to real capture setups with limited views.

### [FusedRF](14268-fusedrf.md)
FusedRF is a method to fuse multiple teacher Radiance Fields (RFs) into a single, compact student RF representation using distillation, allowing for composited scene rendering with the memory and time complexity of a single RF. Unlike naive composition which requires parallel tracing into multiple RFs (increasing memory and render time linearly with scene count), FusedRF fuses them into one RF, maintaining constant memory and render time. Unlike retraining from scratch using rendered RGB views, FusedRF uses 3D distillation from existing RFs, reducing training time by 3x.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, multiple jittered rays are sampled uniformly at random through the spatial footprint of the pixel, and the mean of the rendered values is used as the predicted color in the loss function [S21]. This baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. Performance-wise, the 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### JaxNeRF
A re-implementation of the original NeRF in JAX, used as a benchmark comparison for early works seeking to improve on training and rendering speed [S24]. It is slightly faster and more suited for distributed computing than the original TensorFlow implementation of NeRF [S24].

### Gradient Detachment in Training
A training strategy where the object field component is detached from the backbone to prevent gradient backpropagation to the rendering part, ensuring independent optimization [S25, unofficial]. With detachment, gradients do not backpropagate to the NeRF backbone; without it, gradients from the object field component backpropagate to the backbone, causing mutual influence and a significant drop in rendering quality [S25, unofficial]. The hyper-parameter $\Delta d$ for empty 3D point losses is optimally found to be 0.05 meters [S25, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot K)$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), and $K$ is the number of samples per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot K)$ for storing intermediate rendering states during the forward pass, plus $O(P)$ for model parameters [S5, unofficial]. VRAM usage is a primary constraint, often mitigated by chunking rays [S5, unofficial].
*   **Measured Performance:** In standard Blender dataset experiments, training typically runs for 100,000 iterations [S11, unofficial]. With a batch size of 1024 and 128 samples per ray, training on a single GPU can take several hours [S11, S17, unofficial].
*   **Scaling:** Performance scales with batch size up to VRAM limits; larger batches improve throughput but require more memory [S15, unofficial]. The code notes that batch sizes of $2^{10}$ (1024) are a trade-off between data loading performance and GPU occupancy [S15, unofficial].
*   **Strengths:** Stochastic sampling allows for efficient optimization on large datasets without loading all rays into memory simultaneously [S5, unofficial]. The use of MSE loss and Adam optimizer provides stable convergence for continuous regression [S17, unofficial].
*   **Limitations:** Convergence can be slow due to the low-frequency bias of MLPs [S1]. Training quality is limited by the coarse voxel grid resolution in some implementations, resulting in blurry images [S17, unofficial].
*   **When to use:** Standard NeRF training is used when multi-view images are available and the goal is high-fidelity novel view synthesis [S5, unofficial].
*   **When not to use:** When computational resources are extremely limited and real-time rendering is required, or when only sparse point clouds are available (use KNN-NeRF instead) [S1].

## Background

The NeRF Training Loop was introduced by Mildenhall et al. in the paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial]. It established the standard approach of optimizing a coordinate-based MLP to represent a 3D scene by minimizing the photometric error between rendered and observed images. The method relies on differentiable volume rendering to backpropagate gradients from the image space to the network weights [44-radiance-field-optimization.md]. Since its introduction, it has become the foundational training procedure for numerous NeRF variants and extensions [S1, S24].

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
