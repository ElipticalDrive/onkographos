# Training Protocol

Also called: DCCVT Optimization Setup, DCCVT Training Configuration

## Overview
The Training Protocol defines the standard optimization procedure for neural networks in volume rendering, encompassing optimizer settings, learning rate scheduling, and data augmentation strategies [S2]. It addresses the problem of stabilizing gradient descent and preventing convergence failure during the reconstruction of implicit fields or radiance fields from noisy or sparse inputs [S3]. It differs from general optimization loops by explicitly integrating staged training phases (such as pilot networks) to handle early-stage feature integration issues [S3].

## How it works
The method configures the training loop for neural networks representing volumetric data. It combines a specific optimizer configuration with time-dependent scheduling of learning rates, data augmentations, and regularization weights [S2].

**Inputs and Outputs**
*   **Inputs:** Neural network parameters (weights and biases), training data (point clouds, voxel grids, or radiance field samples), and hyperparameters (learning rate, decay schedule, augmentation parameters) [S2].
*   **Outputs:** Optimized neural network weights representing the target volume or radiance field [S2].

**Data Structures**
*   **Optimizer State:** Maintains first and second moment estimates for the Adam optimizer [S2].
*   **Training Schedule:** A step counter or epoch counter used to trigger conditional logic for pilot network switching and background selection [S3].

**Parameters**
*   **Optimizer:** Adam [S2].
    *   $\beta_1 = 0.9$, $\beta_2 = 0.999$ [S2].
    *   Initial Learning Rate ($\eta$) = $0.0001$ [S2].
*   **Decay Schedule:** Halve the learning rate every 100 epochs [S2].
*   **Augmentation:**
    *   Transformations: Random scaling and translation [S2].
    *   Gaussian Noise Standard Deviation ($\sigma$) = $0.5$ (assuming unit cube grid cells) [S2].
*   **Pilot Network:** Two hidden layers, each with 64 neurons [S3].
*   **Training Duration:** 400 epochs (SDF/voxel inputs) or 250 epochs (point cloud inputs) [S2]. For VFR with pilot network, total 20,000 steps [S3].
*   **Alternative Configuration (DCCVT):** Learning rate $5 \times 10^{-4}$, betas $\beta = (0.8, 0.99)$, 1000 iterations [S1].

**Steps**
1.  **Initialize** the neural network weights. If using a pilot network, initialize it with two hidden layers and 64 neurons each [S3].
2.  **Iterate** through training steps (epochs or iterations).
3.  **Check Step Count** for conditional behaviors:
    *   **Steps 1–300 (Pilot):** Use the pilot network to aid coarse geometry finding [S3].
    *   **Steps > 300 (Pilot):** Discard the pilot network and switch to the main framework (e.g., VFR) [S3].
    *   **Steps 1–25:** Use constant gray backgrounds [S3].
    *   **Steps > 25:** Switch to background augmentation (noise pattern or random Fourier feature) [S3].
    *   **Steps < 100:** Do not apply density or color prior regularization [S3].
    *   **Steps $\ge$ 100:** Apply density prior regularization with weight $\lambda_1 = 2 \cdot 10^{-5}$ and color regularization with weight $\lambda_2 = 8 \cdot 10^{-4}$ [S3].
4.  **Augment Input Data:** Apply random scaling and translation to input point clouds or grids [S2]. Add Gaussian noise with $\sigma = 0.5$ to input point clouds when simulating real scan data [S2].
5.  **Forward Pass:** Compute the rendered output or field values using the current network (or pilot network if in early steps) [S3].
6.  **Compute Loss:** Calculate the loss using the same functions as UNDC for SDF/voxel inputs [S2]. Include regularization terms if step $\ge$ 100 [S3].
7.  **Backward Pass:** Compute gradients with respect to network parameters [S2].
8.  **Update Parameters:** Apply the Adam optimizer update step with the current learning rate [S2].
9.  **Update Learning Rate:** If the current epoch is a multiple of 100, halve the learning rate [S2].
10. **Repeat** until convergence or maximum epochs (400 for SDF/voxel, 250 for point cloud, or 20,000 total steps for VFR with pilot) [S2, S3].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_protocol(network, pilot_network, data_loader, total_steps=20000, 
                   lr_init=0.0001, decay_interval=100, 
                   bg_switch_step=25, prior_start_step=100, 
                   pilot_steps=300, lambda_density=2e-5, lambda_color=8e-4):
    # Initialize optimizer state for Adam [S2]
    optimizer = OptimizerAdam(network.parameters(), lr=lr_init, betas=(0.9, 0.999))
    current_lr = lr_init
    epoch = 0

    for step in range(total_steps):
        # 1. Learning Rate Decay (Epoch-based) [S2]
        if step > 0 and step % decay_interval == 0:
            epoch += 1
            current_lr /= 2.0
            for param_group in optimizer.param_groups:
                param_group['lr'] = current_lr

        # 2. Handle Pilot Network [S3]
        if step < pilot_steps and pilot_network is not None:
            model = pilot_network
        else:
            model = network

        # 3. Data Augmentation [S2]
        batch = next(data_loader)
        inputs = batch['inputs']
        targets = batch['targets']
        
        # Random scaling and translation [S2]
        scale_factor = 1.0 + (random.uniform(-0.1, 0.1))
        translation = random.uniform(-0.05, 0.05)
        inputs = inputs * scale_factor + translation
        
        # Gaussian noise for point clouds [S2]
        if inputs.ndim == 2:  # Point cloud check [G]
            noise = random.gauss(0, 0.5)
            inputs = inputs + noise

        # 4. Background Selection [S3]
        if step < bg_switch_step:
            background = 0.5  # Constant gray [G]
        else:
            background = random.uniform(-1.0, 1.0)  # Augmented background [G]

        # 5. Forward Pass [S3]
        outputs = model(inputs, background)
        
        # 6. Loss Computation [S2]
        loss = compute_loss(outputs, targets)  # Uses UNDC loss functions [S2]
        
        # 7. Apply Regularization Priors [S3]
        if step >= prior_start_step:
            density_reg = compute_density_prior(outputs)  # [S3]
            color_reg = compute_color_prior(outputs)      # [S3]
            loss = loss + lambda_density * density_reg + lambda_color * color_reg
            
        # 8. Backward Pass and Update [S2]
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()  # Applies Adam update rule [S2]
```

**Edge Cases and Pitfalls**
*   **Early Convergence Failure:** Without a pilot network or careful initialization, the model may fail to find correct geometry due to feature integration of all samples in early training [S3].
*   **Optimization Instability:** Background augmentation (noise/Fourier features) can cause instability in the first few steps; hence, constant gray backgrounds are used initially [S3].
*   **Prior Timing:** Applying density priors too early may prevent the general scene from being well-represented; they should only begin after step 100 [S3].
*   **Precision:** Gaussian noise is applied assuming unit cube grid cells; scaling must match the input grid resolution [S2].

## Subtopics

### [Learned Initializations for Neural Fields](6509-learned-initializations-for-neural-fields.md)
This method addresses the inefficiency of optimizing coordinate-based neural networks from random weights for each new signal by applying meta-learning to derive optimal initial weight parameters. It sets itself apart from standard training protocols by decoupling the initialization phase from the specific instance optimization, allowing the network to start closer to the solution manifold of a target signal class (e.g., faces, 3D shapes). This results in faster convergence and improved generalization when only partial observations are available.

### [Network Training Settings](7193-network-training-settings.md)
This article defines the standard hyperparameter configuration for neural networks in radiance-field optimization, specifically focusing on activation functions, dropout regularization, and latent space dimensionality validation. It addresses the problem of stabilizing training and preventing overfitting in differentiable rendering pipelines. It is distinguished from general optimization loops by its focus on architectural regularization (dropout) and non-linearities (PReLU) rather than gradient descent dynamics. It builds on [Training Protocol](4528-training-protocol.md).

### Pilot Network for Early Training
A small neural network used during the early training stage to aid the model in finding coarse geometry, preventing convergence failure caused by feature integration of all samples [S3]. It uses two hidden layers with 64 neurons each and is active only for the first 300 training steps, after which it is discarded in favor of the main framework (e.g., VFR) [S3].

### [Training Protocol with Augmented Backgrounds and Density Prior](7043-training-protocol-with-augmented-backgrounds-and-density-pri.md)
A variant that introduces staged background augmentation and density priors. For the first 25 steps, constant gray backgrounds are used to prevent instability; thereafter, random backgrounds (noise/Fourier features) are applied. Density prior regularization ($\lambda_1 = 2 \cdot 10^{-5}$) and color regularization ($8 \cdot 10^{-4}$) are applied only after step 100 to refine the scene after initial representation.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of training steps and batch size. The pilot network variant adds negligible overhead as it is used for only a small fraction of steps (300 out of 20,000) [S3].
*   **Memory Complexity:** Depends on the neural network architecture (e.g., 128 channels per hidden layer for UNDC [S2]) and the storage of optimizer states (Adam requires two additional tensors per parameter) [S2].
*   **Performance:** The protocol is designed for reconstruction tasks, including noisy real scans, where heavy augmentation and specific priors improve robustness [S2].
*   **Strengths:** Stabilizes training in early stages (via pilot network or constant backgrounds) and refines geometry later (via density priors) [S2, S3].
*   **Limitations:** Requires careful tuning of step thresholds (25, 100, 300) and regularization weights [S3]. Not given in the sources for general radiance field scaling beyond specific experiments.

## Background
The training protocol described originates from works on Neural Dual Contouring (UNDC) and Volume Feature Rendering (VFR) [S2, S3]. It was developed to address challenges in reconstructing implicit surfaces and radiance fields from point clouds and voxel data [S2]. The use of Adam with exponential decay and specific data augmentations is standard in deep learning for geometry processing [S2]. The pilot network concept was introduced to mitigate convergence failures in early training stages [S3].

## Sources
[S1] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S2] Neural Dual Contouring, https://arxiv.org/abs/2202.01999v3, fetched 2026-10-26
[S3] Volume Feature Rendering for Fast Neural Radiance Field Reconstruction, https://doi.org/10.52202/075280-2853, fetched 2026-10-26
