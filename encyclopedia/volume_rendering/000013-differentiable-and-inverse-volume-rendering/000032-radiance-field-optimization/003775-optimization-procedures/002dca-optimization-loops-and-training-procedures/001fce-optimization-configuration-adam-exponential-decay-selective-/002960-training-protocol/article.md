# Training Protocol

Also called: Standard Optimization Procedure

## Overview
The Training Protocol defines the standard optimization procedure for neural networks in volume rendering, encompassing optimizer settings, learning rate scheduling, and data augmentation strategies [S1]. It addresses the problem of stabilizing gradient descent and preventing convergence failure during the reconstruction of implicit fields or radiance fields from noisy or sparse inputs [S1, S2]. It differs from general optimization loops by explicitly integrating specific regularization priors (such as density priors) and staged training phases (such as pilot networks) to handle early-stage feature integration issues [S2, S3].

## How it works
The method configures the training loop for neural networks representing volumetric data. It combines a specific optimizer configuration with time-dependent scheduling of learning rates, data augmentations, and regularization weights.

**Inputs and Outputs**
*   **Inputs:** Neural network parameters (weights and biases), training data (point clouds, voxel grids, or radiance field samples), and hyperparameters (learning rate, decay schedule, augmentation parameters).
*   **Outputs:** Optimized neural network weights representing the target volume or radiance field.

**Data Structures**
*   **Optimizer State:** Maintains first and second moment estimates for Adam [S1].
*   **Training Schedule:** A step counter or epoch counter used to trigger conditional logic for learning rate decay, background switching, and prior application [S2, S3].

**Parameters**
*   **Optimizer:** Adam [S1, S3].
    *   $\beta_1 = 0.9$ [S1].
    *   $\beta_2 = 0.999$ [S1].
    *   Initial Learning Rate ($\eta$) = $0.0001$ [S1].
*   **Decay Schedule:** Halve the learning rate every 100 epochs [S1].
*   **Augmentation:**
    *   Gaussian Noise Standard Deviation ($\sigma$) = $0.5$ (assuming unit cube grid cells) [S1].
    *   Transformations: Random scaling and translation [S1].
*   **Regularization Weights (Density Prior):**
    *   Density regularization weight ($\lambda_1$) = $2 \cdot 10^{-5}$ [S3].
    *   Color regularization weight = $8 \cdot 10^{-4}$ [S3].

**Steps**
1.  **Initialize** the neural network weights. If using a pilot network, initialize the small pilot network (two hidden layers, 64 neurons each) [S2].
2.  **Iterate** through training steps (epochs or iterations).
3.  **Check Step Count** for conditional behaviors:
    *   **Steps 1–25:** Use constant color backgrounds (random shade of gray) [S3].
    *   **Steps > 25:** Switch to background augmentation (noise pattern or random Fourier feature chosen uniformly at random) [S3].
    *   **Steps 1–300 (Pilot):** If using the pilot network variant, render using the pilot network to aid coarse geometry finding [S2].
    *   **Steps > 300 (Pilot):** Discard the pilot network and switch to the main framework (e.g., VFR) [S2].
    *   **Steps < 100:** Do not apply density prior regularization [S3].
    *   **Steps $\ge$ 100:** Apply density prior regularization with weight $\lambda_1 = 2 \cdot 10^{-5}$ and color regularization with weight $8 \cdot 10^{-4}$ [S3].
4.  **Augment Input Data:**
    *   Apply random scaling and translation to input point clouds or grids [S1].
    *   Add Gaussian noise with $\sigma = 0.5$ to input point clouds if simulating real scan data [S1].
5.  **Forward Pass:** Compute the rendered output or field values using the current network (or pilot network if in early steps) [S1, S2].
6.  **Compute Loss:** Calculate the loss using the same functions as UNDC (for SDF/voxel inputs) or standard radiance field losses [S1, S3]. Include regularization terms if step $\ge$ 100 [S3].
7.  **Backward Pass:** Compute gradients with respect to network parameters [S1].
8.  **Update Parameters:** Apply the Adam optimizer update step with the current learning rate [S1].
9.  **Update Learning Rate:** If the current epoch is a multiple of 100, halve the learning rate [S1].
10. **Repeat** until convergence or maximum epochs (e.g., 400 for SDF/voxel, 250 for point cloud, or 20,000 total steps for VFR with pilot) [S1, S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_protocol(network, pilot_network, data_loader, total_steps=20000, 
                   lr_init=0.0001, decay_interval=100, 
                   bg_switch_step=25, prior_start_step=100, 
                   pilot_steps=300, lambda_density=2e-5, lambda_color=8e-4):
    optimizer = Adam(network.parameters(), lr=lr_init, betas=(0.9, 0.999))
    current_lr = lr_init
    
    for step in range(total_steps):
        # 1. Learning Rate Decay (Epoch-based, assuming 1 step = 1 epoch for simplicity or mapped)
        # Note: Sources specify "every 100 epochs". If step is iteration, this logic needs epoch mapping.
        # Assuming step corresponds to epoch for S1, or adjusting for S2's 20k steps.
        if step > 0 and step % decay_interval == 0:
            current_lr /= 2.0
            for param_group in optimizer.param_groups:
                param_group['lr'] = current_lr

        # 2. Handle Pilot Network (S2)
        if step < pilot_steps and pilot_network is not None:
            # Use pilot network for coarse geometry
            model = pilot_network
        else:
            # Discard pilot, use main network
            model = network

        # 3. Data Augmentation and Background Selection (S1, S3)
        batch = next(data_loader)
        inputs = batch['inputs']
        targets = batch['targets']
        
        # Random scaling and translation (S1)
        inputs = augment_random_scale_translate(inputs)
        
        # Gaussian noise for point clouds (S1)
        if is_point_cloud(inputs):
            noise = torch.randn_like(inputs) * 0.5
            inputs = inputs + noise
            
        # Background selection (S3)
        if step < bg_switch_step:
            background = random_gray_color()
        else:
            background = random_augmentation_background() # noise pattern or Fourier feature

        # 4. Forward Pass
        outputs = model(inputs, background)
        
        # 5. Loss Computation
        loss = compute_loss(outputs, targets) # Uses UNDC loss functions for SDF (S1)
        
        # 6. Apply Regularization Priors (S3)
        if step >= prior_start_step:
            density_reg = compute_density_prior_regularization(outputs)
            color_reg = compute_color_regularization(outputs)
            loss = loss + lambda_density * density_reg + lambda_color * color_reg
            
        # 7. Backward Pass and Update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
```

**Edge Cases and Pitfalls**
*   **Early Convergence Failure:** Without a pilot network or careful initialization, the model may fail to find correct geometry due to feature integration of all samples in early training [S2].
*   **Optimization Instability:** Background augmentation (noise/Fourier features) can cause instability in the first few steps; hence, constant gray backgrounds are used initially [S3].
*   **Prior Timing:** Applying density priors too early may prevent the general scene from being well-represented; they should only begin after step 100 [S3].
*   **Precision:** Gaussian noise is applied assuming unit cube grid cells; scaling must match the input grid resolution [S1].

## Subtopics

### [Learned Initializations for Neural Fields](6509-learned-initializations-for-neural-fields.md)
This method addresses the inefficiency of optimizing coordinate-based neural networks from random weights for each new signal by applying meta-learning to discover optimal initial weight parameters. It enables faster convergence and serves as a strong prior for a specific class of signals, improving generalization under partial observations. It differs from standard random initialization by learning a shared starting point across a dataset, and from hard-coded inductive biases (like SDF inverted spheres) by relying on learned parameters rather than architectural constraints.

### [Network Training Settings](7193-network-training-settings.md)
This article defines the standard hyperparameter configuration for neural networks in radiance-field optimization, specifically focusing on activation functions, dropout regularization, and latent space dimensionality validation. It addresses the problem of stabilizing training and preventing overfitting in differentiable rendering pipelines. It is distinguished from general optimization loops by its focus on architectural regularization (dropout) and non-linearities (PReLU) rather than gradient descent dynamics. It builds on [Training Protocol](4528-training-protocol.md).

### Pilot Network for Early Training
A small neural network used during the early training stage to aid the model in finding coarse geometry, preventing convergence failure caused by feature integration of all samples [S2]. It uses two hidden layers with 64 neurons each and is active only for the first 300 training steps, after which it is discarded in favor of the main framework (e.g., VFR) [S2].

### Training Protocol with Augmented Backgrounds and Density Prior
A variant that introduces staged background augmentation and density priors. For the first 25 steps, constant gray backgrounds are used to prevent instability; thereafter, random backgrounds (noise/Fourier features) are applied [S3]. Density prior regularization ($\lambda_1 = 2 \cdot 10^{-5}$) and color regularization ($8 \cdot 10^{-4}$) are applied only after step 100 to refine the scene after initial representation [S3].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of training steps and batch size. The pilot network variant adds negligible overhead as it is used for only a small fraction of steps (300 out of 20,000) [S2].
*   **Memory Complexity:** Depends on the neural network architecture (e.g., 128 channels per hidden layer for UNDC [S1]) and the storage of optimizer states (Adam requires two additional tensors per parameter) [S1].
*   **Performance:** The protocol is designed for reconstruction tasks, including noisy real scans, where heavy augmentation and specific priors improve robustness [S1].
*   **Strengths:** Stabilizes training in early stages (via pilot network or constant backgrounds) and refines geometry later (via density priors) [S2, S3].
*   **Limitations:** Requires careful tuning of step thresholds (25, 100, 300) and regularization weights [S3]. Not given in the sources for general radiance field scaling beyond specific experiments.

## Background
The training protocol described originates from works on Neural Dual Contouring (UNDC) and Volume Feature Rendering (VFR) [S1, S2]. It was developed to address challenges in reconstructing implicit surfaces and radiance fields from point clouds and voxel data, particularly in the presence of noise and sparse data [S1, S2]. The use of Adam with exponential decay and specific data augmentations is standard in deep learning for geometry processing [S1]. The pilot network concept was introduced to mitigate convergence failures in early training stages [S2]. The density prior and background augmentation strategies were refined for semantic volume rendering and text-based transfer function design [S3].

## Sources
[S1] Neural Dual Contouring, https://arxiv.org/abs/2202.01999v3, fetched 2026-10-26
[S2] Volume Feature Rendering for Fast Neural Radiance Field Reconstruction, https://doi.org/10.52202/075280-2853, fetched 2026-10-26
[S3] Text-based transfer function design for semantic volume rendering, https://doi.org/10.1109/vis55277.2024.00047, fetched 2026-10-26
