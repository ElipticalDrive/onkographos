# Specific Architecture Training Procedures


## Overview
Specific Architecture Training Procedures define the optimization loops and configuration strategies required to train distinct radiance field architectures, addressing their unique convergence challenges and data requirements. Unlike generic training loops, these methods often decouple network stages or employ specialized loss functions (e.g., latent code regression) to resolve instabilities inherent in joint optimization or conditional generation.

## Shared principles
The training procedures for these architectures share the following mechanisms:
- **Iterative Optimization:** Both child methods utilize stochastic gradient descent (or Adam) over epochs or fixed step counts, minimizing a composite loss function $\mathcal{L}$ that typically combines reconstruction error with architecture-specific regularization or matching terms.
- **State Management:** Both methods require explicit management of model checkpoints and loss histories, saving parameters (e.g., `.param` files) and metrics (e.g., `.npy` arrays) at regular intervals to allow for monitoring and resumption.
- **Device Placement:** Inputs (voxel data, images, indices) and model parameters are explicitly moved to the target device (e.g., GPU) before forward passes.
- **Scheduler Application:** Learning rate scheduling is applied periodically (e.g., per epoch) to stabilize convergence, using standard schedulers like `StepLR`.

## Subtopics

### [VCCNF Training Loop](4736-vccnf-training-loop.md)
The VCCNF Training Loop is a supervised optimization procedure for a Voxelized Continuous Coordinate Normalizing Flow (VCCNF) [4736-vccnf-training-loop.md]. It learns to map Gaussian noise to shape latent codes conditioned on voxel data by minimizing the Mean Squared Error (MSE) between predicted and target latent codes. It is chosen when training conditional generative models for 3D shapes where voxel data is available. It differs from diffusion-based guidance by directly optimizing a normalizing flow in a latent space rather than guiding a radiance field with a diffusion prior.

### [Two-Stage Training for Learnable Proposer](8685-two-stage-training-for-learnable-proposer.md)
Two-Stage Training for Learnable Proposer divides network optimization into two distinct stages to resolve the chicken-and-egg problem inherent in learning to sample for radiance fields [8685-two-stage-training-for-learnable-proposer.md]. Training the proposer requires a good fine network to provide a useful training signal, but the fine network can only improve if sampled at informative locations. This method sets itself apart from joint training approaches by decoupling proposer initialization from radiance field fine-tuning to ensure stable convergence. It is chosen when using learnable sampling strategies that depend on coarse network features.

### IBRNet Baseline Training Loop
The IBRNet Baseline Training Loop involves learning a generalized model for 250,000 solver steps on 15 training DTU scenes, followed by fine-tuning the pre-trained model on seven training views of a specific test scene for another 60,000 steps [S1, unofficial]. This two-phase approach allows the model to learn generalizable features before adapting to specific scenes. The training time is approximately 0.33 seconds per iteration, with a rendering time of 33.3 seconds per frame [S1, unofficial].

## Background
The VCCNF and Two-Stage Training procedures are described in unofficial notes associated with the source [S1, unofficial]. The IBRNet training configuration is detailed in the source [S1, unofficial], which compares training times and steps against other methods like NeRF and IDR.

## Sources
[S1] Fast Training of Neural Lumigraph Representations using Meta Learning, https://doi.org/10.48550/arxiv.2106.14942, fetched 2026-10-26 (unofficial)
