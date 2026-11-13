# Acceleration and Sparse Structures

Also called: Spatial Partitioning, Visual Hull Culling, Block-based NeRF

## Overview
This family of methods accelerates NeRF training and rendering by reducing the volume of space queried by the neural network, thereby lowering memory usage and computational cost. It addresses the bottleneck where standard NeRFs sample uniformly across a large bounding box, evaluating the MLP in empty space or on transient objects. It sets itself apart from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by modifying *where* samples are taken (spatial constraints) rather than *how* they are integrated, and from [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) by using static or pre-computed geometric priors (hulls, blocks) rather than dynamic error metrics during training.

## Shared principles
All methods in this family restrict the domain of the radiance field query. Instead of evaluating the neural network $\mathcal{F}_\Theta$ at arbitrary points $\mathbf{x}$ within the scene bounding box $\mathcal{B}$, they define a subset $\mathcal{S} \subset \mathcal{B}$ such that $\mathbf{x} \in \mathcal{S}$ is required for evaluation. This reduces the number of forward passes through the MLP. The core mechanism is a culling step:
1.  Define a spatial constraint (e.g., visual hull, block grid, sparse cache).
2.  For each ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, compute the intersection of the ray with the constraint.
3.  Only evaluate $\mathcal{F}_\Theta(\mathbf{r}(t))$ for $t$ where $\mathbf{r}(t)$ lies within the constraint.

## Subtopics

### [Zip-NeRF Training Procedure](9538-zip-nerf-training-procedure.md)
A two-stage inverse rendering optimization that uses a sparse Zip-NeRF radiance cache and a fast cache to decouple material/environment optimization from scene geometry. It sets itself apart by using a "gradient trick" to handle the non-analytic expected value of the radiance estimator, stabilizing the joint training of physically-based models. Choose this for high-fidelity inverse rendering where material and lighting parameters must be optimized alongside geometry.

### [VaxNeRF](11379-vaxnerf.md)
An acceleration strategy that restricts ray sampling to the **visual hull** $V$, the maximum volume compatible with object silhouettes from multi-view segmentation masks. It sets itself apart from [Zip-NeRF](9538-zip-nerf-training-procedure.md) by using a single MLP with uniform sampling inside $V$ rather than hierarchical caching. Choose this when coarse foreground-background segmentation masks are available and training speed is prioritized over handling complex transparency or transient objects.

### [Block-NeRF](13537-block-nerf.md)
Scales NeRFs to city-scale scenes by partitioning the environment into overlapping spatial blocks, each trained with an independent NeRF model. It sets itself apart from [VaxNeRF](11379-vaxnerf.md) by using explicit spatial partitioning and multi-model composition rather than voxel-based culling. Choose this for large outdoor datasets where a single global NeRF is computationally infeasible and transient objects (pedestrians, cars) need to be masked out.

### Spatial Partitioning
A strategy to create tiles that ensures balanced workload (similar number of network evaluations/ray samples) across GPUs, avoiding suboptimal GPU utilization caused by uneven scene content distribution [S1]. If a sparse point cloud is available, recursively identify the plane where the Cumulative Distribution Function (CDF) equals 0.5 for the 3D point distribution along each axis to partition space [S1]. If no sparse point cloud is available, discretize randomly sampled training rays into 3D samples to estimate scene content distribution, then partition similarly [S1]. Partitioning runs typically within seconds [S1].

## Background
The concept of restricting NeRF queries to relevant volumes originated with early observations that uniform sampling wastes computation on empty space. VaxNeRF [S1] introduced the visual hull as a hard constraint. Block-NeRF [S1] addressed scalability by decomposing the scene. Zip-NeRF [S1] introduced sparse caching for inverse rendering. Spatial partitioning for multi-GPU scaling was formalized in NeRF-XL [S1].

## Sources
[S1] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
