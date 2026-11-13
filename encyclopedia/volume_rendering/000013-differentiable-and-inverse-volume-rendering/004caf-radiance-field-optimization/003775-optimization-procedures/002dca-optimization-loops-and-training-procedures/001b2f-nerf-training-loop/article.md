# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth observations by updating the parameters of a Neural Radiance Field (NeRF) [S5, S6, S14, unofficial]. It solves the inverse problem of learning a continuous volumetric scene representation from a set of posed 2D images [S5, S14, unofficial]. It sets itself apart from evaluation rendering by using stochastic ray sampling and gradient-based weight updates rather than deterministic full-image synthesis [S2, S9, unofficial]. It builds on [Optimization Loops and Training Procedures](5622-optimization-loops-and-training-procedures.md) by implementing the specific data loading, ray batching, and loss computation mechanics required for coordinate-based MLPs [S5, S14, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics $C_i$ and intrinsics $K_i$) and dimensions $H \times W$. A NeRF model (typically two MLPs: one for density $\sigma$ and one for color $\mathbf{c}$, optionally with positional encoding) [S5, S14, unofficial]. An optimizer (typically Adam) [S5, unofficial][S12][S13, unofficial].
*   **Outputs:** Updated NeRF weights $\theta$ after $T$ iterations.

**Data Structures:**
*   **Rays:** A ray is defined by origin $\mathbf{o}$ and direction $\mathbf{d}$. In pre-computed batching implementations, rays are stored in a flat array `rays_rgb` of shape $[(N-1)HW, 3, 3]$ (float32), where axis 1 contains $[\mathbf{o}, \mathbf{d}, \mathbf{c}_{gt}]$ and axis 2 contains the XYZ/RGB components [S5, S14, unofficial].
*   **Batch:** A tensor of shape $[3, B, 3]$ containing the current batch of $B$ rays, split into origins/directions and target colors [S14, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the NeRF model parameters (e.g., weights to small random values or zeros) [S15, unofficial].
    *   Initialize the optimizer (e.g., Adam with learning rate $\eta$) [S5, unofficial][S12][S13, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Pre-compute all possible rays from training images into `rays_rgb` if using batching [S5, S11, S14, unofficial].

2.  **Iteration Loop (for $i = 1$ to $N_{iters}$):**
    *   **Ray Sampling:**
        *   If batching is enabled: Slice the next $B$ rays from `rays_rgb` starting at index $i_{batch}$ [S5, S14, unofficial].
        *   If not batching: Randomly select a training image, generate all rays for that image, optionally apply a central crop (precrop) for early iterations, and randomly sample $B$ unique pixel coordinates [S14, unofficial].
        *   Extract ray origins $\mathbf{o}$, directions $\mathbf{d}$, and ground-truth colors $\mathbf{c}_{gt}$ for the batch [S7, S14, unofficial].
    *   **Forward Pass (Rendering):**
        *   Sample $K$ points along each ray within the near/far bounds [S13, S14, unofficial].
        *   Query the NeRF model for density $\sigma$ and color $\mathbf{c}$ at these points [S5, S6, unofficial].
        *   Compute the rendered color $\hat{\mathbf{c}}$ using volume rendering (alpha compositing) [S5, S6, unofficial].
        *   If using a coarse-to-fine strategy (two-stage), render with a coarse network, resample points based on weights, and render with a fine network [S5, S11, unofficial].
    *   **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between predicted and ground-truth colors:
            $$ \mathcal{L}_{MSE} = \frac{1}{B} \sum_{j=1}^B \| \hat{\mathbf{c}}_j - \mathbf{c}_{gt,j} \|^2 $$
            [S14, S17, unofficial].
        *   If coarse-to-fine, add the coarse loss: $\mathcal{L} = \mathcal{L}_{MSE, coarse} + \mathcal{L}_{MSE, fine}$ [S11, unofficial].
        *   Add regularization losses if applicable (e.g., sparsity, Eikonal, Total Variation) [S7, unofficial][S8][S16, unofficial].
    *   **Backpropagation:**
        *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
        *   Compute gradients: `loss.backward()` [S18, unofficial].
        *   Update weights: `optimizer.step()` [S18, unofficial].
    *   **Learning Rate Update:**
        *   Update the learning rate according to the scheduler (e.g., exponential decay: $\eta_{new} = \eta_{old} \cdot \text{decay\_rate}^{(i / \text{decay\_steps})}$) [S5, S18, unofficial].
    *   **Logging and Checkpointing:**
        *   Log loss and PSNR ($\text{PSNR} = -10 \log_{10}(\mathcal{L}_{MSE})$) every $L$ steps [S14, unofficial].
        *   Render validation images every $V$ steps [S11, unofficial].
        *   Save model checkpoint every $C$ steps [S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def train_nerf_loop(model, optimizer, scheduler, dataloader, num_iters, device):
    """
    Standard NeRF training loop.
    
    Args:
        model: NeRF neural network (MLP).
        optimizer: Optimizer (e.g., Adam).
        scheduler: Learning rate scheduler.
        dataloader: Iterator yielding batches of (rays_o, rays_d, target_rgb).
        num_iters: Total number of training iterations.
        device: CUDA or CPU device.
    """
    criterion = nn.MSELoss()
    
    for step in range(num_iters):
        # 1. Data Loading
        # dataloader yields (rays_o, rays_d, target_rgb) for the current batch
        rays_o, rays_d, target_rgb = next(dataloader)
        rays_o = rays_o.to(device)
        rays_d = rays_d.to(device)
        target_rgb = target_rgb.to(device)
        
        # 2. Forward Pass
        # model.render_rays returns predicted RGB for the batch
        # Note: Implementation of render_rays depends on [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
        pred_rgb = model.render_rays(rays_o, rays_d)
        
        # 3. Loss Computation
        loss = criterion(pred_rgb, target_rgb)
        
        # 4. Backpropagation
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # 5. Learning Rate Update
        scheduler.step()
        
        # 6. Logging (Optional)
        if step % 100 == 0:
            psnr = -10.0 * torch.log10(loss)
            print(f"Step {step}, Loss: {loss.item():.6f}, PSNR: {psnr.item():.2f}")
```

**Parameters with Typical Values:**
*   **Learning Rate ($\eta$):** $5 \times 10^{-4}$ [S11, unofficial], $0.01$ [S17, unofficial], or defined in config [S8][S9, unofficial].
*   **Batch Size ($B$ or $N_{rand}$):** 1024 [S5, S11, unofficial], 2048 [S5, unofficial], 4096 [S17, unofficial].
*   **Number of Iterations ($N_{iters}$):** 100,000 [S11, unofficial], 1,000,000 [S14, unofficial].
*   **Decay Rate:** 0.1 [S6, S18, unofficial].
*   **Decay Steps:** $\text{args.lrate\_decay} \times 1000$ [S5, S18, unofficial].
*   **Positional Encoding Frequency:** 10 for position, 4 for direction [S11, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Large batches or high-resolution rays can exceed GPU memory. Mitigated by chunking rays during the forward pass (processing $K$ rays at a time) [S5, S14, unofficial].
*   **Gradient Freeing:** Higher-order derivatives (e.g., for logging gradients) must be computed *before* `loss.backward()` to avoid freeing the autograd graph [S3, unofficial].
*   **Resuming Training:** When loading a checkpoint, the scheduler's last epoch must be set to `start_epoch - 1` to ensure correct scheduling [S9, unofficial].
*   **Empty Checkpoints:** If a checkpoint is loaded but the step counter is 0, it is treated as an error/empty state [S15, unofficial].

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
This family of methods accelerates NeRF training and rendering by reducing the volume of space queried by the neural network, thereby lowering memory usage and computational cost. It addresses the bottleneck where standard NeRFs sample uniformly across a large bounding box, evaluating the MLP in empty space or on transient objects. It sets itself apart from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by modifying *where* samples are taken (spatial constraints) rather than *how* they are integrated, and from [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) by using static or pre-computed geometric priors (hulls, blocks) rather than dynamic error metrics during training.

### [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md)
This family of methods addresses the problem of optimizing camera parameters (extrinsics and intrinsics) simultaneously with Neural Radiance Field (NeRF) weights, eliminating the reliance on pre-computed Structure-from-Motion (SfM) pipelines like COLMAP. By treating camera poses as learnable parameters, these methods enable 3D reconstruction from sparse or uncalibrated image sets. What sets this family apart from siblings like [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md) is the explicit optimization of discrete camera poses per image rather than continuous motion functions, and from [Camera Preconditioning](12545-camera-preconditioning.md) by focusing on the joint gradient-based update of poses and scene geometry rather than preconditioning the optimizer for faster convergence.

### [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md)
This family of methods integrates pre-trained Monocular Depth Estimation (MDE) networks into the NeRF training loop to resolve the shape-radiance ambiguity and scale uncertainty inherent in sparse-view or single-view reconstruction. Unlike standard NeRF which relies solely on photometric consistency, these methods use MDE outputs as geometric priors to guide volume sampling, regularize density distributions, or directly adapt the MDE network to the scene's absolute geometry. They are distinguished from sibling methods like [DäRF](12331-d-rf.md) by their specific strategies for handling the relative nature of monocular depth, ranging from sparse prior fine-tuning to distribution-based Earth Mover’s Distance supervision [11911-depth-guided-nerf-training-via-earth-mover-s-distance.md].

### [NeRF Editing and Inpainting Procedures](12494-nerf-editing-and-inpainting-procedures.md)
This family covers methods for modifying pre-trained Neural Radiance Fields (NeRFs) to remove objects, insert new content, or alter appearance without retraining the entire scene from scratch. It solves the problem of maintaining multi-view consistency and geometric plausibility when applying 2D priors (such as inpainting or diffusion models) to 3D representations. Unlike standard training loops that optimize for reconstruction of input views, these procedures optimize for a target state defined by user intent (e.g., masks, text prompts, or reference images), often leveraging external 2D generative models or semantic fields as guidance.

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

### [RePaint-NeRF](14365-repaint-nerf.md)
RePaint-NeRF is a framework for editing 3D content in Neural Radiance Fields (NeRF) using text prompts by leveraging a pre-trained diffusion model to guide changes in designated 3D content, ensuring view consistency and scene integrity. It solves the problem of modifying specific objects within a complex 3D scene without altering the surrounding environment or introducing artifacts like "black holes" where previously occluded geometry is exposed. It sets itself apart from standard NeRF optimization by decoupling the process into a semantic mask extraction stage and a diffusion-guided editing stage, using CLIP features for segmentation and Stable Diffusion for generation.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. This approach relies on 'oracle' scale information that does not exist for most datasets [S21]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### JaxNeRF
A re-implementation of the original NeRF in JAX, used as a benchmark comparison for early works seeking to improve on training and rendering speed [S24]. It is slightly faster and more suited for distributed computing than the original TensorFlow implementation of NeRF [S24].

### Gradient Detachment in Training
A training strategy where the object field component is detached from the backbone to prevent gradient backpropagation to the rendering part, ensuring independent optimization [S25, unofficial]. With detachment, gradients do not backpropagate to the NeRF backbone; without it, gradients from the object field component backpropagate to the NeRF backbone, causing mutual influence [S25, unofficial]. Detachment achieves better scene rendering and decomposition quality [S25, unofficial]. The hyper-parameter $\Delta d$ for empty 3D point losses is typically 0.05 meters [S25, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot B \cdot K)$, where $N_{iters}$ is the number of iterations, $B$ is the batch size (number of rays), and $K$ is the number of samples per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(B \cdot K)$ for storing intermediate rendering states during the forward pass, plus $O(\text{model\_size})$ for network weights [S5, S14, unofficial]. VRAM usage is mitigated by chunking rays [S5, unofficial].
*   **Measured Performance:** Training on 128x128 images for 250 epochs [S4, unofficial]. On a dataset of 199 images, 5 epochs with batch size 1024 results in ~39,000 iterations [S17, unofficial].
*   **Scaling:** Larger batch sizes (e.g., $2^{14}$) can hinder data loading performance and prevent full GPU occupancy, while smaller batch sizes (e.g., $2^{10}$) cap iterations per second [S15, unofficial].
*   **Strengths:** Stochastic sampling allows optimization on large datasets without loading all rays into memory simultaneously [S5, S14, unofficial].
*   **Limitations:** Blurry predictions if voxel grid resolution is coarse or advanced ray sampling is not used [S17, unofficial]. Slow convergence compared to some accelerated methods [S24].

## Background
The NeRF training loop was introduced in the seminal paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" by Mildenhall et al. (2020) [S5, unofficial][S24]. It established the standard procedure of optimizing a coordinate-based MLP using stochastic gradient descent on photometric loss [S5, S14, unofficial]. Early implementations used TensorFlow [S5, unofficial], later ported to PyTorch [S6, S13, unofficial] and JAX [S24]. The method has become the foundational training paradigm for subsequent NeRF variants and related neural rendering techniques [S1, S24].

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
