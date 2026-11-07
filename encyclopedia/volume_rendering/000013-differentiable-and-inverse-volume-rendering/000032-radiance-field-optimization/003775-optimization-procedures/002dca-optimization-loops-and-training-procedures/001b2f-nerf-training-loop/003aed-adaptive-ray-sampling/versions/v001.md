# Adaptive Ray Sampling

## Overview
Adaptive Ray Sampling refers to a family of strategies in neural radiance field (NeRF) training that dynamically adjust the distribution of sample points along viewing rays to improve reconstruction quality and training efficiency. Unlike uniform sampling, which treats all regions of a ray equally, these methods allocate computational resources to areas of high geometric or photometric complexity, such as object boundaries or high-frequency details [S1, S3, unofficial]. This family addresses the inefficiency of uniform sampling in sparse or complex scenes by clustering samples where they contribute most to the final rendered image [S1, S3, unofficial].

## Shared principles
All methods in this family share the goal of optimizing the placement of sample points $t_i$ along a ray segment $[t_{near}, t_{far}]$ to minimize rendering error or training time. The core mechanism involves replacing the uniform distribution of samples with a non-uniform distribution derived from scene statistics or network predictions.

1.  **Importance Metric**: Each method defines a metric to identify "important" regions. This metric may be based on:
    *   **Network Output**: Volume density $\sigma$ or opacity weights $w_i$ from a coarse pass [S3, S4, unofficial].
    *   **Image Statistics**: Color variance, gradient magnitude, or depth variation in the input images [S1].
    *   **Rendering Error**: The difference between rendered and ground-truth colors, used to guide spatial subdivision [S1].
2.  **Sampling Distribution**: The importance metric is converted into a probability density function (PDF) or a set of weights. Samples are then drawn from this distribution. Common techniques include:
    *   **Inverse Transform Sampling**: Using the cumulative distribution function (CDF) of the weights to select sample depths [S3, S4, unofficial].
    *   **Stratified Sampling**: Adding jitter to uniform bins to break symmetry, often used in the coarse pass [S3, S4, unofficial].
    *   **Spatial Subdivision**: Dividing the image plane into regions (e.g., quadtrees) and adjusting ray counts per region based on local error [S1].
3.  **Differentiability**: The sampling process itself is typically non-differentiable. Gradients are computed only with respect to the network parameters (MLP weights) and not the sample positions, requiring the sample generation step to be detached from the computation graph [S3, S4, unofficial].

## Subtopics

### [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md)
Uses a two-stage hierarchical approach. A coarse network samples uniformly (with jitter) to identify opaque regions. A fine network then densely samples these regions using inverse transform sampling based on coarse weights. It improves high-frequency detail reconstruction without proportional coarse cost.

### [Probability-based Ray Sampling with Quadtree Subdivision](7481-probability-based-ray-sampling-with-quadtree-subdivision.md)
Accelerates training by subdividing the image plane into a quadtree. Nodes are marked as converged if their rendering error is below a threshold, reducing ray counts in simple regions. Complex regions are subdivided and sampled more densely based on color variance.

### [TermiNeRF](8828-terminerf.md)
Trains a dedicated sampling network to predict importance weights directly from ray parameters. It uses centered-logarithmic sampling and distribution matching to learn where to place samples, allowing the color network to skip free space. It differs from hierarchical methods by using a single network evaluation per ray for sampling guidance.

### [Pixel Region and Depth Guided Ray Sampling](8866-pixel-region-and-depth-guided-ray-sampling.md)
Increases sampling probability in pixels with high color and depth variation. It uses depth estimation and gradient magnitude to guide ray allocation, reducing redundant rays in flat regions and improving geometry perception.

## Background
Adaptive sampling in volume rendering has roots in early ray tracing techniques that used stratified sampling to reduce noise [G]. In the context of NeRF, the seminal work by Mildenhall et al. introduced hierarchical sampling (Coarse-to-Fine) to handle the sparsity of samples in empty space [S3, unofficial]. Subsequent works like [S1] explored spatial adaptivity via quadtree subdivision to reduce training time, while [S2] and [S1] investigated learned sampling distributions to further optimize the trade-off between quality and computation.

## Sources
[S1] Probability-based Ray Sampling with Quadtree Subdivision, https://example.com/quadtree-sampling, 2023-10-01
[S2] TermiNeRF: Efficient Neural Rendering with Learned Sampling, https://example.com/terminerf, 2023-10-01
[S3] NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis, https://example.com/nerf-paper, 2020-01-01
