# Training Loop

Also called: NeRF Training Pipeline, RtStereoHumanModel Training Loop, Training Loop & Data Fetching

## Overview
The Training Loop is the main optimization procedure for Neural Radiance Fields (NeRF), orchestrating data loading, ray sampling, model forward passes, loss calculation, and parameter updates [S1, unofficial]. It solves the problem of iteratively refining a differentiable radiance field representation to match observed input images [G]. It sets itself apart from sibling methods like [Training Ray Batching](2252-training-ray-batching.md) by managing the high-level control flow (epochs, validation, checkpointing) rather than just the geometry of ray selection, and from [Camera Pose Optimization](15669-camera-pose-optimization.md) by typically assuming fixed camera poses unless explicitly extended [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training dataset (images, poses, intrinsics), model architecture (NeRF MLP/Gaussian Splatting), optimizer configuration (learning rate, weight decay), and hyperparameters (batch size, epochs, near/far bounds) [G].
*   **Outputs:** Trained model weights, optimizer state, and optionally rendered validation/test images [G].

**Data Structures:**
*   **Batch:** A dictionary or object containing tensors for image data (`lmain`, `rmain` for stereo; or single view), camera parameters (intrinsics $K$, extrinsics $P$), and metadata (view indices, bounds) [G].
*   **Ray Bundle:** A flat array of rays, where each ray is defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, concatenated with the ground truth RGB color $\mathbf{c}$ at the sampled pixel [S4, unofficial].
*   **Metrics:** Accumulators for L1 loss, SSIM, and Chamfer distance [G].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets using `get_nerf_datasets` [S1, unofficial].
    *   Create the NeRF model and optimizer via `create_model(cfg)` or `create_nerf(args)` [S1, unofficial][S4, unofficial].
    *   Initialize the learning rate scheduler (e.g., LambdaLR or OneCycleLR) [S1, unofficial][S2, unofficial].
    *   Initialize gradient scaler if using mixed precision [S2, unofficial].

2.  **Data Fetching:**
    *   Fetch the next batch from `train_loader` [S1, unofficial].
    *   If `StopIteration` occurs, reset the iterator to `iter(train_loader)` and fetch again [S3, unofficial].
    *   Move all relevant tensors (images, camera matrices, masks) to the CUDA device [S1, unofficial][S2, unofficial].
    *   *Dataset-Specific Bounds:*
        *   **LLFF:** Near = $\min(\text{bounds}) \times 0.9$, Far = $\max(\text{bounds}) \times 1.0$ (or 0/1 if `no_ndc`) [S4, unofficial].
        *   **Blender:** Near = 2.0, Far = 6.0 [S4, unofficial].
        *   **DeepVoxels:** Near = $\text{hemi\_R} - 1.0$, Far = $\text{hemi\_R} + 1.0$ [S4, unofficial].

3.  **Ray Sampling:**
    *   Sample random pixels from the ground truth image [S1, unofficial].
    *   Generate a ray bundle from these pixels using camera intrinsics and extrinsics [S1, unofficial].
    *   If batching is enabled, stack rays and concatenate with RGB values, then reshape into a flat array [S4, unofficial].
    *   *Parameter:* `N_rand` (number of random rays per batch) [S4, unofficial].

4.  **Forward Pass:**
    *   Pass the ray bundle through the model with `is_train=True` [S2, unofficial].
    *   The model renders the scene, producing predicted RGB colors and optionally depth/alpha [G].
    *   *Gaussian Splatting Variant:* Render using `pts2render` with background color from config (e.g., `(0.0, 0.0, 0.0)`) [S2, unofficial].

5.  **Loss Calculation:**
    *   Compute the primary rendering loss (e.g., L1 or MSE) between predicted and ground truth RGB [S1, unofficial].
    *   Compute structural similarity (SSIM) loss [S2, unofficial].
    *   *Optional Regularization:*
        *   **Chamfer Loss:** If enabled, extract valid points from left/right XYZ, sample 10,000 points, and compute chamfer distance [S2, unofficial].
        *   **Total Loss Formula:** $\text{Loss} = 0.8 \times L1 + 0.2 \times (1 - \text{SSIM}) + 0.5 \times \text{Chamfer Loss}$ [S2, unofficial].
        *   **Adaptive/Two-Phase:** Use adaptive rendering loss weights or two-phase supervision to balance low-frequency structure and high-frequency details [S7].

6.  **Backpropagation and Optimization:**
    *   Scale the loss using `GradScaler` if mixed precision is active [S2, unofficial].
    *   Perform backward pass: `loss.backward()` [S1, unofficial].
    *   Unscale gradients and clip them to a global norm of 1.0 [S2, unofficial].
    *   Step the optimizer: `optimizer.step()` [S1, unofficial].
    *   Step the learning rate scheduler [S1, unofficial].
    *   Update the gradient scaler [S2, unofficial].
    *   Zero gradients: `optimizer.zero_grad()` [S1, unofficial].

7.  **Logging and Validation:**
    *   Accumulate loss metrics (L1, SSIM, Chamfer) [S2, unofficial].
    *   Print averages every 100 steps [S2, unofficial].
    *   If `total_steps % eval_freq == 0`:
        *   Set model to eval mode.
        *   Run evaluation (`run_eval()`) on validation set [S2, unofficial].
        *   Set model back to train mode.
        *   Freeze BatchNorm layers if applicable (e.g., in Raft-Stereo) [S2, unofficial].

8.  **Checkpointing and Rendering:**
    *   If `epoch % checkpoint_interval == 0`, save model and optimizer state [S1, unofficial].
    *   If `epoch % render_interval == 0` and `epoch > 0`, render test images using surround cameras [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def training_loop(model, train_loader, val_loader, optimizer, scheduler, cfg):
    """
    Main training loop for NeRF.
    """
    total_steps = 0
    scaler = GradScaler(enabled=cfg.mixed_precision)
    
    for epoch in range(cfg.start_epoch, cfg.num_epochs):
        for batch in train_loader:
            image = batch['image'].to(device='cuda')
            camera = batch['camera'].to(device='cuda')
            
            rays, gt_rgb = sample_rays(image, camera, cfg.N_rand)
            
            with autocast(enabled=cfg.mixed_precision):
                pred_rgb = model(rays, is_train=True)
                l1_loss = l1_loss_fn(pred_rgb, gt_rgb)
                ssim_loss = 1.0 - ssim_fn(pred_rgb, gt_rgb)
                
                chamfer_loss = 0.0
                if cfg.if_chamfer and total_steps > cfg.iter_from:
                    chamfer_loss = compute_chamfer_loss(batch)
                
                total_loss = 0.8 * l1_loss + 0.2 * ssim_loss + 0.5 * chamfer_loss

            scaler.scale(total_loss).backward()
            scaler.unscale_(optimizer)
            clip_grad_norm_(model.parameters(), max_norm=1.0)
            scaler.step(optimizer)
            scaler.update()
            optimizer.zero_grad()
            
            total_steps += 1
            
            if total_steps % 100 == 0:
                log_metrics(l1_loss, ssim_loss, chamfer_loss)

        scheduler.step()
        
        if total_steps % cfg.eval_freq == 0:
            model.eval()
            run_eval(val_loader)
            model.train()
            
        if epoch % cfg.checkpoint_interval == 0:
            save_checkpoint(model, optimizer, epoch)
            
        if epoch % cfg.render_interval == 0 and epoch > 0:
            render_test_images(model)

# Helper definitions
class GradScaler:
    def __init__(self, enabled): self.enabled = enabled
    def scale(self, loss): return loss
    def backward(self): pass
    def unscale_(self, optimizer): pass
    def step(self, optimizer): optimizer.step()
    def update(self): pass

def sample_rays(image, camera, n_rand):
    # Samples n_rand random pixels and constructs ray origins/directions
    return rays, gt_rgb

def l1_loss_fn(pred, target): return torch.mean(torch.abs(pred - target))
def ssim_fn(pred, target): return 1.0 # Placeholder for SSIM calculation
def compute_chamfer_loss(batch): return 0.0
def log_metrics(l1, ssim, chamfer): pass
def run_eval(val_loader): pass
def save_checkpoint(model, optimizer, epoch): pass
def render_test_images(model): pass
```

## Subtopics

### [MVDream Multi-View Diffusion Guidance](6345-mvdream-multi-view-diffusion-guidance.md)
MVDream is a guidance method that fine-tunes Stable Diffusion with 3D-aware components trained on Objaverse to improve multi-view consistency and alleviate the Janus problem in text-to-3D generation. It provides gradient guidance for 3D optimization by encoding RGB images and camera poses into a multi-view diffusion model, predicting noise, and stepping the DDIM scheduler. Unlike standard Stable Diffusion guidance which treats views independently, MVDream explicitly conditions on camera poses to enforce geometric coherence across multiple viewpoints.

### [Diffusion-Based Guidance for Radiance Field Optimization](8117-diffusion-based-guidance-for-radiance-field-optimization.md)
Diffusion-Based Guidance for Radiance Field Optimization uses the score function of a pretrained 2D diffusion model to provide gradient signals that guide the optimization of a differentiable radiance field parametrization $z = g(\theta)$ towards the mode of a conditional probability distribution $p(z_t | y)$. It solves the problem of generating or refining 3D scenes from sparse views or text prompts by leveraging the rich prior knowledge of 2D diffusion models, effectively removing non-photorealistic artifacts and enforcing geometric consistency. It sets itself apart from sibling methods like [VCCNF Training Loop](4736-vccnf-training-loop.md) by relying on a frozen, pre-trained diffusion prior rather than a learned multi-view consistency network, and from [MVDream Multi-View Diffusion Guidance](6345-mvdream-multi-view-diffusion-guidance.md) by typically using a single-view or text-conditioned 2D model without explicit multi-view joint training.

### [Specific Architecture Training Procedures](18781-specific-architecture-training-procedures.md)
Specific Architecture Training Procedures define the optimization loops and configuration strategies required to train distinct radiance field architectures, addressing their unique convergence challenges and data requirements. Unlike generic training loops, these methods often decouple network stages or employ specialized loss functions (e.g., latent code regression) to resolve instabilities inherent in joint optimization or conditional generation.

### [Training and Supervision](19347-training-and-supervision.md)
This article details the supervision strategy for optimizing a hybrid lighting representation that combines a volumetric lighting field with a parametric sky model for street scenes. It solves the problem of estimating physically plausible illumination from single or multi-view LDR images to enable realistic virtual object insertion. It is distinguished from sibling methods by its explicit separation of sky and volumetric lighting components, supervised via a pre-trained sky modeling network and adversarial feedback from object insertion realism rather than pure photometric consistency alone.

### Render Pipeline Initialization
*   **Parameters:** `valCamerasCount` (int, default 100), `testCamerasCount` (int, default 200), `grayLized` (bool, default true) [S5, unofficial].
*   **Layout:** Render class fields include `renderRes` (Image); `groundTruth`, `train_cameras`, `val_cameras`, `test_cameras` (Image*/Camera*); `valCamerasCount` (int, default 100); `testCamerasCount` (int, default 200); `grayLized` (bool, default true); `datasetSize` (int); `scene` (ObjectScene*); `ray` (_rays); `sceneOptimizer` (Optimizer); `currentHierarchy` (int); `trainingProcessIdx` (int, default 0); `sampleRaysIndexGPU`, `sampleRaysIndexHost` (int*); `idx` (int*); `config` (Configuration*) [S5, unofficial].

### Importance prediction with stop gradient
*   **Definition:** A design choice where the input features to the importance predictor are detached from the gradient computation to prevent trivial solutions and simplify loss scaling [S6, unofficial].
*   **Parameter:** Importance predictor loss weight is not required as a separate hyperparameter because stop gradient decouples it from the main loss, allowing Adam to scale them independently [S6, unofficial].
*   **Pitfall:** Without stop gradient, gradients could push the proposer and importance predictor toward a trivial solution where bad samples are predicted as unimportant to minimize loss [S6, unofficial].

### Adaptive rendering loss weight learning
*   **Definition:** A mechanism that adaptively learns rendering loss weights to align frequency relationships between PE and pixel supervision [S7].
*   **Pitfall:** May be unstable in the early training phase, causing global structures to resemble the baseline [S7].
*   **Number:** Improves all metrics compared to Baseline (PSNR 19.73 on DTU 3-view) [S7].
*   **Comparison:** Improves local details (e.g., hands become cleaner) compared to Baseline and Model-A, but requires two-phase supervision for stable global structure learning [S7].

### Two-phase rendering supervision
*   **Definition:** A rendering supervision strategy that makes NeRF optimization focus more on learning low-frequency information [S7].
*   **Number:** Improves PSNR from 19.54 (Baseline) to 19.81 (Model-A) on DTU 3-view [S7].
*   **Comparison:** Better at rendering global structures (e.g., face) compared to Baseline, but lacks local detail refinement [S7].

### Evaluation and Metric Computation Pipeline
*   **Step:** Train the model with a train/test split by passing the `--eval` flag to `train.py`. This withholds a test set for evaluation [S8, unofficial].
*   **Step:** Generate renderings for the trained model using `render.py`, specifying the model path [S8, unofficial].
*   **Step:** Compute error metrics on the generated renderings using `metrics.py`, specifying the model path [S8, unofficial].

### Prior Gradient Caching
*   **Definition:** An optimization acceleration technique where back-propagated prior gradients ($\nabla_x \lambda_{diff} L_{diff}$) are cached and reused for multiple Adam steps on scene codes, while rendering gradients ($\nabla_x \lambda_{rend} L_{rend}$) are recomputed in each step [S9].
*   **Step:** In the outer loop, compute prior gradients $g_{xdiff}$ once per scene batch [S9].
*   **Step:** In the inner loop, compute new rendering gradients $g_{xrend}$ for the current ray batch [S9].
*   **Step:** Add the cached prior gradient $g_{xdiff}$ to the new rendering gradient $g_{xrend}$ to form the update gradient $g_x$ [S9].
*   **Step:** Update scene codes using $g_x$, allowing fewer diffusion passes than rendering passes [S9].

### City-on-Web Training and LOD Refinement
*   **Step:** LOD generation phase: Freeze training of submodels for 5,000 iterations, then refine them and shared global deferred MLP jointly for an additional 10,000 iterations [S10].
*   **Parameter:** Training loss weights: $\lambda_1 = 1.0, \lambda_2 = 1.0, \lambda_3 = 0.01, \lambda_4 = 0.05, \lambda_5 = 0.001$. Sample count for sparsity loss: $2^{14}$ samples [S10].

### Soft 3D Segmentation from 2D User Edits
*   **Step:** Train $MLP_s$ alongside the last layer of the color MLP using the RGB loss (Eq. 4), without ground truth segmentation masks [S11].
*   **Formula:** $c_x = \alpha_x c'_x + (1 - \alpha_x) c_x$ where $c'_x$ is the new color from the retrained last layer, $c_x$ is the original color, and $\alpha_x$ is the probability of the feature being part of the edited area [S11].
*   **Layout:** Soft segmentation MLP ($MLP_s$): Two-layer MLP. Input: 32-dimensional multiresolution hash grid feature vector $f$. Hidden layer: 64 neurons. Output: 1-dimensional probability $\alpha$ [S11].
*   **Number:** Convergence time for soft segmentation: typically less than 5 seconds [S11].

### Camera Predictor Training
*   **Definition:** A self-supervised method to train a CNN-based encoder to predict camera parameters from rendered images, used for style inversion tasks [S12, unofficial].
*   **Step:** Randomly sample a camera pose and latent codes to render the output image [S12, unofficial].
*   **Step:** Use a CNN-based encoder (backbone initialized with pretrained ResNet18) to read the image and predict input camera parameters based on a smoothed L1 loss function [S12, unofficial].

### Trajectory Smoothness Regularization
*   **Definition:** A regularization technique applied to the camera trajectory to prevent high-frequency predictions and improve trajectory estimation by leveraging the sequential nature of camera movement [S13].
*   **Definition:** A regularization term added to the loss function to enforce smooth and continuous camera trajectory estimation across adjacent Rolling Shutter (RS) frames, addressing the issue of high-frequency predictions in vanilla RS-NeRF [S13].
*   **Step:** For each pair of adjacent trajectory vectors $\vec{d}_k$ and $\vec{d}_{k+1}$, compute the mid-point trajectory vector $\vec{d}_{mid}$ [S13].
*   **Formula:** $\mathcal {L}_{s}=||\overrightarrow {\mathbf {n}}_{mid} - \operatorname {mean}(\overrightarrow {\mathbf {n}}_k + \overrightarrow {\mathbf {n}}_{k+1})||_2.$ where $\overrightarrow {\mathbf {n}}_k$ and $\overrightarrow {\mathbf {n}}_{k+1}$ are unit direction vectors of estimated camera trajectories for adjacent RS frames $I^R_k$ and $I^R_{k+1}$, and $\overrightarrow {\mathbf {n}}_{mid}$ is the unit direction vector for the camera trajectory during the exposure interval between these two frames [S13].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of training steps and batch size. Each step involves a forward pass through the MLP/Gaussian splatter and a backward pass [G].
*   **Memory Complexity:** Proportional to the number of model parameters and the batch size (storing gradients and optimizer states) [G].
*   **Measured Performance:** Varies by implementation. For example, IBRNet baseline training takes 0.33 sec/iteration with 33.3 sec/frame rendering [S12, unofficial].
*   **Scaling:** Scales with batch size and number of rays sampled per step (`N_rand`). Larger batches improve gradient stability but increase memory usage [G].
*   **Strengths:** Flexible, supports various loss functions (L1, SSIM, Chamfer), and integrates with standard PyTorch/TensorFlow optimization utilities [G].
*   **Limitations:** Can be slow for high-resolution scenes due to ray sampling and rendering overhead. Prone to local minima without careful initialization or regularization (e.g., importance sampling, adaptive loss weights) [G].
*   **When to use:** Standard NeRF training for novel view synthesis [G].
*   **When not to use:** When real-time training is required (consider [MVDream](6345-mvdream-multi-view-diffusion-guidance.md) or [Diffusion-Based Guidance](8117-diffusion-based-guidance-for-radiance-field-optimization.md) for faster convergence via priors) [G].

## Background
The NeRF training loop was introduced in the seminal paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" by Mildenhall et al. (2020) [S4, unofficial]. It established the standard pipeline of volumetric rendering integrated with gradient-based optimization [G]. Subsequent works have extended this loop with various optimizations, such as importance sampling [S6, unofficial], adaptive loss weighting [S7], and diffusion-based guidance [S1, unofficial].

## Sources
[S1] Shreyas120/l3d_assignment3, https://github.com/Shreyas120/l3d_assignment3, fetched 2026-10-26 (unofficial)
[S2] YaourtB/GPS_plus, https://github.com/YaourtB/GPS_plus, fetched 2026-10-27 (unofficial)
[S3] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S4] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S5] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S6] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S7] Few-shot NeRF by Adaptive Rendering Loss Regularization, https://core.ac.uk/download/708671746.pdf, fetched 2026-10-30
[S8] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
[S9] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
[S10] City-on-Web: Real-Time Neural Rendering of Large-Scale Scenes on the Web, https://doi.org/10.1007/978-3-031-72970-6_22, fetched 2026-10-26
[S11] IReNe: Instant Recoloring of Neural Radiance Fields, https://openaccess.thecvf.com/content/CVPR2024/papers/Mazzucchelli_IReNe_Instant_Recoloring_of_Neural_Radiance_Fields_CVPR_2024_paper.pdf, fetched 2026-10-29
[S12] StyleNeRF: A Style-based 3D-Aware Generator for High-resolution Image Synthesis, https://doi.org/10.48550/arxiv.2110.08985, fetched 2026-10-26 (unofficial)
[S13] RS-NeRF: Neural Radiance Fields from Rolling Shutter Images, https://arxiv.org/abs/2407.10267v1, fetched 2026-10-26
