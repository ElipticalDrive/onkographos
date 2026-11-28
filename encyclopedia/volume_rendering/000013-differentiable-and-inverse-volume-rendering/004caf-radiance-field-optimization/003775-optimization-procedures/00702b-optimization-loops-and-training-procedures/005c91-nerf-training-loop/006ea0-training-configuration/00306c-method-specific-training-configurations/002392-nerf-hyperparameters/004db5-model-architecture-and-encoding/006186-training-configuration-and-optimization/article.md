# Training Configuration and Optimization

Also called: NeRF Training Hyperparameters, Optimization Settings

## Overview
This article defines the hyperparameter configurations, optimization schedules, and loss function variants required to train Neural Radiance Field (NeRF) models effectively. It addresses the problem of converging implicit neural representations from sparse or unconstrained 2D image collections by specifying learning rates, sampling strategies, and architectural regularizations. What sets this family apart from the underlying [Model Architecture and Encoding](13550-model-architecture-and-encoding.md) is the focus on the *training dynamics*—such as learning rate schedules, loss weighting, and specific regularization techniques—rather than the network topology or positional encoding functions themselves.

## Shared principles
The training process for NeRF variants generally follows a stochastic gradient descent loop using the Adam optimizer. The core mechanism involves minimizing a photometric reconstruction loss between rendered pixels and ground-truth images. Key shared principles include:
*   **Ray Sampling:** Rays are sampled through the scene, and points are queried along these rays using stratified or importance sampling to estimate the volume rendering integral .
*   **Optimization:** The Adam optimizer is the standard choice, often with a learning rate decay schedule .
*   **Loss Function:** The primary loss is typically Mean Squared Error (MSE) or L1 loss between rendered and target colors .

## Subtopics

### [Depth-Guided NeRF Training Configuration](11916-depth-guided-nerf-training-configuration.md)
This configuration extends standard NeRF training by incorporating explicit depth supervision to improve geometric accuracy, particularly in sparse settings. It distinguishes itself by using an Earth Mover’s Distance (EMD) based depth loss and architectural regularization (dropout) to handle depth uncertainty. It is chosen when ground-truth depth maps are available or when high-fidelity geometry is required over pure photometric fidelity. See [Depth-Guided NeRF Training Configuration](11916-depth-guided-nerf-training-configuration.md).

### [CLIPNeRF Training Configuration](12073-clipnerf-training-configuration.md)
This configuration enables text-guided stylization of NeRFs by replacing photometric loss with semantic constraints in the CLIP embedding space. It distinguishes itself by minimizing the negative cosine similarity between rendered image patches and text prompts, allowing for style transfer without paired 3D data. It is chosen for artistic stylization or semantic editing tasks. See [CLIPNeRF Training Configuration](12073-clipnerf-training-configuration.md).

### [NeRF-W Hyperparameter Configuration](14635-nerf-w-hyperparameter-configuration.md)
This configuration addresses the reconstruction of unconstrained, "in-the-wild" photo collections containing transient objects (pedestrians, vehicles) and varying lighting. It distinguishes itself by decomposing the scene into static geometry and transient appearance components using a multi-branch MLP, preventing transient artifacts from corrupting the static structure. It is chosen for large-scale, uncontrolled scene capture. See [NeRF-W Hyperparameter Configuration](14635-nerf-w-hyperparameter-configuration.md).

## Background
The original NeRF paper  established the baseline training configuration, using Adam with a learning rate of $5 \times 10^{-4}$, 500,000 total steps, and a learning rate decay to $5 \times 10^{-5}$ after 80% of training. Subsequent works have adapted these configurations to handle specific challenges: depth guidance  for accuracy, CLIP  for semantic control, and NeRF-W  for in-the-wild robustness.

