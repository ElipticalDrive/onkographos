# Training Loop with Adaptive Regularization & LR Decay

Also called: NeRF Training Loop with Exponential LR Decay, Ray Batching Training, SimpleNeRF Training Configuration & Loss Schedule, Training Loop with Exponential LR Decay

## Overview
This method defines a specific training protocol for optimizing sparse-input Neural Radiance Fields (NeRFs) by combining an Adam optimizer with exponential learning rate decay, a two-phase loss schedule, and adaptive regularization weights [S2][S4, unofficial]. It solves the problem of training instability and overfitting in sparse view scenarios by progressively introducing complex regularization terms and decaying the learning rate to fine-tune the neural field [S2][S4, unofficial]. It differs from sibling schedules by explicitly coupling the learning rate decay with the activation of specific loss components (e.g., switching from basic color/depth losses to advanced consistency losses) and dynamically scaling regularization weights relative to the learning rate [S2][S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training images, camera poses, scene bounds, depth priors (optional), and hyperparameters ($lr_{initial}$, $decay\_rate$, $decay\_steps$, $batch\_size$, $nSamples$, regularization weights).
*   **Outputs:** Optimized NeRF model parameters (MLP weights, embeddings), training metrics (PSNR, loss).

**Data Structures:**
*   **Ray Batches:** A tensor of shape $(N_{rand}, 5)$ or $(N_{rand}, 6)$ containing ray origins $(o_x, o_y, o_z)$ and directions $(d_x, d_y, d_z)$, optionally including depth priors and confidence scores [S3, S4, unofficial].
*   **Optimizer State:** Adam optimizer state for model parameters [S1, unofficial][S2].
*   **Model:** A neural field (MLP) that maps 3D coordinates and view directions to color and density [S2][S4, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training images, poses, and bounds [S3, unofficial].
    *   Initialize the NeRF model [S3, unofficial].
    *   Initialize the Adam optimizer with initial learning rate $lr_{initial}$ and weight decay [S1, unofficial][S2].
    *   Set $global\_step = 0$ [S3, unofficial].

2.  **Ray Sampling:**
    *   Randomly sample $N_{rand}$ ray indices [S3, S4, unofficial].
    *   Fetch corresponding ray data (origin, direction, target RGB, depth priors, confidences) [S3, S4, unofficial].
    *   Shuffle ray indices periodically to ensure uniform sampling [S1, S3, unofficial].

3.  **Forward Pass (Rendering):**
    *   Render the sampled rays using the neural field model [S4, unofficial].
    *   Compute the predicted RGB map ($rgb\_map$), alphas ($alphas\_map$), depth ($depth\_map$), and auxiliary outputs (e.g., raw density, uncertainty) [S4, unofficial].
    *   The rendering process involves volume integration along each ray [S3, unofficial].

4.  **Loss Computation:**
    *   Compute the primary reconstruction loss (Mean Squared Error):
        $$ loss_{img} = \frac{1}{N} \sum_{i=1}^{N} (rgb_i - target_i)^2 $$
        where $N$ is the number of rays in the batch [S3, S4, unofficial].
    *   **Phase 1 (Iterations $0$ to $10,000$):**
        *   Compute only $L_{color}$ (based on $loss_{img}$) and $L_{sd}$ (stereo depth loss, if depth priors are available) [S2].
        *   Total Loss: $L = L_{color} + L_{sd}$ [S2].
    *   **Phase 2 (Iterations $> 10,000$):**
        *   Compute additional regularization losses: $L_{ap}$ (appearance consistency), $L_{av}$ (view consistency), and $L_{cfc}$ (cross-frame consistency) [S2].
        *   Adjust loss weights such that their magnitudes are of a similar order [S2].
        *   Total Loss: $L = L_{color} + L_{sd} + L_{ap} + L_{av} + L_{cfc}$ [S2].
    *   **Adaptive Regularization (Continuous):**
        *   If `Ortho_reg_weight > 0`:
            $$ loss_{reg} = tensorf.vector\_comp\_diffs() $$
            $$ L_{ortho} = Ortho\_reg\_weight \times loss_{reg} $$
            Add $L_{ortho}$ to total loss [S4, unofficial].
        *   If `L1_reg_weight > 0`:
            $$ loss_{reg\_L1} = tensorf.density\_L1() $$
            $$ L_{L1} = L1\_reg\_weight \times loss_{reg\_L1} $$
            Add $L_{L1}$ to total loss [S4, unofficial].
            *   *Note:* `L1_reg_weight` switches from an initial value to a rest value after the first alpha mask update iteration [S4, unofficial].
        *   If `TV_weight_density > 0` or `TV_weight_app > 0`:
            *   Decay TV weights by `lr_factor` every iteration [S4, unofficial].
            *   Compute Total Variation loss for density and appearance fields [S4, unofficial].
            *   Add weighted TV losses to total loss [S4, unofficial].

5.  **Backward Pass and Optimization:**
    *   Zero gradients: `optimizer.zero_grad()` [S4, unofficial].
    *   Compute gradients: `total_loss.backward()` [S4, unofficial].
    *   Update model parameters: `optimizer.step()` [S3, S4, unofficial].

6.  **Learning Rate and Weight Decay Update:**
    *   Update the global step: $global\_step \leftarrow global\_step + 1$ [S3, unofficial].
    *   Calculate the new learning rate using exponential decay:
        $$ lr_{new} = lr_{initial} \times (decay\_rate)^{\frac{global\_step}{decay\_steps}} $$
        where $decay\_rate$ is typically $0.1$ and $decay\_steps$ is often $args.lrate\_decay \times 1000$ [S3, unofficial].
    *   Update the learning rate for all parameter groups in the optimizer [S3, S4, unofficial].
    *   *Note:* Some implementations apply the decay factor multiplicatively to the current LR: $lr_{new} = lr_{old} \times lr\_factor$ [S4, unofficial]. Ensure consistency with the source formula.

7.  **Logging and Checkpointing:**
    *   Compute PSNR for tracking:
        $$ PSNR = -10 \cdot \log_{10}(loss_{img}) $$
        [S1, S3, unofficial].
    *   Log metrics and save checkpoints periodically [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_adaptive_reg(
    model, optimizer, train_dataset, val_dataset,
    lr_initial=5e-4, decay_rate=0.1, decay_steps=100000,
    batch_size=2048, n_samples=48,
    ortho_reg_weight=0.0, l1_reg_weight=0.0,
    tv_weight_density=0.0, tv_weight_app=0.0,
    lr_factor=0.1,
    phase_switch_step=10000
):
    """
    Training loop with adaptive regularization and exponential LR decay.
    """
    global_step = 0
    current_lr = lr_initial
    
    # Initialize TV weights if applicable
    current_tv_weight_density = tv_weight_density
    current_tv_weight_app = tv_weight_app
    
    while global_step < decay_steps: # Or until convergence
        # 1. Sample Rays
        ray_indices = sample_random_rays(train_dataset, batch_size)
        rays, targets, depth_priors, confidences = fetch_ray_data(train_dataset, ray_indices)
        
        # 2. Forward Pass
        rgb_map, alphas_map, depth_map, extras = model.render(
            rays, n_samples=n_samples, white_bg=True, ndc_ray=False
        )
        
        # 3. Compute Loss
        # Primary MSE Loss
        loss_img = torch.mean((rgb_map - targets.rgb) ** 2)
        total_loss = loss_img
        
        # Phase 1: Basic Losses
        if global_step < phase_switch_step:
            # L_color is already included as loss_img
            # L_sd (Stereo Depth) if depth priors available
            if depth_priors is not None:
                loss_sd = compute_stereo_depth_loss(depth_map, depth_priors, confidences)
                total_loss += loss_sd
        else:
            # Phase 2: Advanced Losses
            # L_ap, L_av, L_cfc (Placeholder for specific implementations)
            loss_ap = compute_appearance_consistency_loss(extras)
            loss_av = compute_view_consistency_loss(extras)
            loss_cfc = compute_cross_frame_consistency_loss(extras)
            # Weights adjusted to balance magnitudes [S2]
            total_loss += loss_ap + loss_av + loss_cfc
            
        # Adaptive Regularization
        if ortho_reg_weight > 0:
            loss_reg_ortho = model.vector_comp_diffs()
            total_loss += ortho_reg_weight * loss_reg_ortho
            
        if l1_reg_weight > 0:
            # Switch l1_reg_weight after first alpha mask update [S4, unofficial]
            loss_reg_l1 = model.density_L1()
            total_loss += l1_reg_weight * loss_reg_l1
            
        if current_tv_weight_density > 0:
            loss_tv_density = model.TV_loss_density(tvreg)
            total_loss += current_tv_weight_density * loss_tv_density
            
        if current_tv_weight_app > 0:
            loss_tv_app = model.TV_loss_app(tvreg)
            total_loss += current_tv_weight_app * loss_tv_app
            
        # 4. Backward Pass
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
        
        # 5. Update Learning Rate and Regularization Weights
        global_step += 1
        
        # Exponential LR Decay
        # Formula: new_lrate = lrate * (decay_rate ** (global_step / decay_steps)) [S3, unofficial]
        # Or multiplicative: lr *= lr_factor [S4, unofficial]
        # Using the explicit exponential formula from [S3, unofficial] as primary, [S4, unofficial] as alternative
        new_lr = lr_initial * (decay_rate ** (global_step / decay_steps))
        
        for param_group in optimizer.param_groups:
            param_group['lr'] = new_lr
        current_lr = new_lr
            
        # Decay TV weights multiplicatively [S4, unofficial]
        current_tv_weight_density *= lr_factor
        current_tv_weight_app *= lr_factor
        
        # 6. Logging
        psnr = -10.0 * np.log10(loss_img.item())
        log_metrics(global_step, psnr, total_loss.item())
        
        # Checkpointing
        if global_step % 10000 == 0:
            save_checkpoint(model, optimizer, global_step)
            
        # Validation
        if global_step % 5000 == 0:
            validate(model, val_dataset)
```

**Parameters with Typical Values:**
*   $lr_{initial}$: $5 \times 10^{-4}$ [S2].
*   $decay\_rate$: $0.1$ [S3, unofficial].
*   $decay\_steps$: $args.lrate\_decay \times 1000$ (e.g., 100,000 steps) [S3, unofficial].
*   $batch\_size$ ($N_{rand}$): 2048 (common in NeRF implementations, though not explicitly stated in notes, [S4, unofficial] uses `args.batch_size`) [S4, unofficial].
*   $nSamples$: 48 (default in many NeRFs, [S4, unofficial] uses `nSamples`) [S4, unofficial].
*   $L_{ap}, L_{v}, L_{p}$ weights: 10, 4, 3 respectively [S2].
*   $k$: 5, $e_{\tau}$: 0.1 [S2].
*   $\lambda_1 = 1$, $\lambda_2 = \lambda_3 = \lambda_4 = \lambda_5 = 0.1$ [S2].

**Edge Cases and Pitfalls:**
*   **TV Weight Vanishing:** If `lr_factor < 1`, TV regularization weights decay multiplicatively every iteration, potentially vanishing rapidly [S4, unofficial].
*   **L1 Weight Switching:** `L1_reg_weight` switches values only after the first alpha mask update, which may cause inconsistent regularization early in training [S4, unofficial].
*   **Validation Set Size:** Cap validation set at 102,400 rays (1024*100) to manage memory; sample linearly if larger [S1, unofficial].
*   **Precision:** Ensure loss weights are scaled so their magnitudes are similar to prevent one term from dominating [S2].

## Operating characteristics
*   **Time Complexity:** $O(N_{rays} \cdot N_{samples})$ per iteration for rendering [S3, S4, unofficial]. Total training time is approximately 22 hours per scene on a single NVIDIA RTX A4000 16GB GPU for 100,000 iterations [S2].
*   **Memory Complexity:** $O(N_{rays} \cdot N_{samples})$ for intermediate rendering buffers [S4, unofficial].
*   **Strengths:** Effective for sparse input NeRFs; reduces floaters and radiance ambiguity through adaptive regularization [S2]. Simple to implement with standard PyTorch/TensorFlow [S2][S3, unofficial].
*   **Limitations:** Computationally expensive due to volume rendering and multiple loss terms [S2]. Sensitive to hyperparameter tuning (loss weights, decay rates) [S2][S4, unofficial].
*   **When to Use:** Sparse view synthesis, scenes with significant floaters or ambiguity [S2].
*   **When Not to Use:** Dense view settings where simpler schedules suffice; real-time applications requiring faster convergence [S2].

## Background
This training configuration is primarily associated with **SimpleNeRF** [S2], which regularizes sparse input NeRFs using simpler solutions (e.g., fewer PE frequencies, view-independent radiance) as augmentations. It differs from **FreeNeRF** by using reprojection error to filter incorrect depth estimates and including $L_{cfc}$, which FreeNeRF lacks [S2]. The exponential decay schedule is a standard technique in NeRF optimization, often combined with warm-up phases [G]. The specific two-phase loss schedule and adaptive regularization weights are detailed in the SimpleNeRF implementation [S2] and related collaborative NeRF works [S4, unofficial].

## Sources
[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S2] SimpleNeRF: Regularizing Sparse Input Neural Radiance Fields with Simpler Solutions, https://doi.org/10.1145/3610548.3618188, fetched 2026-10-26
[S3] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S4] Collaborative Neural Radiance Fields for Novel View Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
