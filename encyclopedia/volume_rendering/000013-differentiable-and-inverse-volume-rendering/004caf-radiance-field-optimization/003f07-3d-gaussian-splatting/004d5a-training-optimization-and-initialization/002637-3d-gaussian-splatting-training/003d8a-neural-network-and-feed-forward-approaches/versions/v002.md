# Neural Network and Feed-Forward Approaches

Also called: Feed-forward 3DGS, Neural 3DGS, Generalizable 3D Gaussian Splatting

## Overview
This family encompasses methods that replace or augment the iterative per-scene gradient descent optimization of standard 3D Gaussian Splatting with neural networks. It solves the latency and computational cost of per-scene training by either regressing Gaussian parameters directly from input images (feed-forward) or using neural priors to guide the optimization process. It differs from [Base 3DGS Optimization](7549-base-3d-gaussian-splatting.md) by eliminating or reducing the need for scene-specific iterative updates, and from [Physical Parameter Optimization](4111-physical-parameter-optimization.md) by relying on learned data-driven priors rather than physical constraints alone.

## Shared principles
The core mechanism involves a neural network that maps input data (typically multi-view images or sparse views) to 3D Gaussian parameters or features.
1.  **Input Representation:** Inputs are usually a set of $N$ images $\{I_1, \dots, I_N\}$ with known camera parameters.
2.  **Feature Extraction:** A backbone (e.g., ResNet, Transformer) extracts 2D features.
3.  **Geometric Aggregation:** Methods often use cost volumes or attention mechanisms to aggregate multi-view information. For cost volumes, similarity is computed between reference and target view features at discrete depth levels:
    $$ C(x, y, d) = \text{Similarity}(F_r(x, y), F_t(x', y')) $$
    where $(x', y')$ is the projected pixel in the target view for depth $d$.
4.  **Parameter Regression:** A decoder (e.g., U-Net, MLP) predicts Gaussian properties $\{\mathbf{\mu}_i, \mathbf{R}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_i\}$ or high-dimensional feature vectors $\mathcal{F}$.
5.  **Differentiable Rendering:** During training, predicted Gaussians are rasterized using the standard 3DGS rasterizer. The rendered output is compared to ground truth (images, depth, or feature maps) to compute loss and update network weights.

## Subtopics

### [MVSplat](9213-mvsplat.md)
MVSplat is a feed-forward deep neural network that directly regresses 3D Gaussian Splatting parameters from sparse multi-view images in a single pass. It distinguishes itself by building upon PixelSplat, combining multi-view feature extraction through transformers and cost volumes, processed by a U-Net. Unlike per-scene optimization, it learns a prior that generalizes to unseen scenes, enabling immediate reconstruction without iterative training. See [MVSplat](9213-mvsplat.md) for implementation details.

### [VRP-UDF Training & Rendering Setup](14646-vrp-udf-training-rendering-setup.md)
VRP-UDF integrates a neural network-based volume rendering prior into the 3DGS pipeline to learn Unsigned Distance Functions (UDFs) and guide optimization. It distinguishes itself by using a learned prior to resolve ambiguities in thin structures and surface reconstruction, addressing inaccuracies caused by random hierarchical sampling and ill-defined UDF fields near surfaces. See [VRP-UDF Training & Rendering Setup](14646-vrp-udf-training-rendering-setup.md) for implementation details.

### GS-Net
GS-Net is a 3D neural network that predicts high-dimensional feature vectors for optimized 3D Gaussians, using Q-Render to efficiently render these features into 2D maps for distillation from 2D foundation models [S1, unofficial]. Given optimized 3D Gaussians $\mathcal{G} = \{g_i\}_{i=1}^N$, a 3D neural network predicts a set of $C$-dimensional Gaussian features $\mathcal{F} = \{f_i\}_{i=1}^N$ where $f_i \in \mathbb{R}^C$ [S1, unofficial]. It uses Q-Render to transform predicted Gaussian features into rendered feature maps [S1, unofficial]. The 3D neural network is trained to minimize the rendering loss between the rendered feature maps and target 2D feature maps extracted from CLIP’s vision encoder [S1, unofficial]. To ensure compatibility with typical 3D backbones like Point Transformer v3 (PTv3) and MinkUnet, 3D Gaussians are voxized by sampling their center location $\mu$ [S1, unofficial]. De-voxelization steps convert predicted voxel features into predicted Gaussian features $\mathcal{F}$ [S1, unofficial]. Unlike methods that force per-scene feature optimization, GS-Net predicts features in a generalizable manner using a 3D neural network [S1, unofficial].

### Masked Auto Encoder for Generalizable 3DGS
This approach uses a masked autoencoder to learn a generalizable 3D Gaussian splatting-based representation [S2]. It combines a masked autoencoder with a 3D transformer-based architecture to create a generalized representation capable of reconstructing unseen scenes and supporting transfer learning [S2].

## Background
Standard 3D Gaussian Splatting requires per-scene optimization, which is computationally expensive and slow. Neural network approaches emerged to provide generalizable solutions. MVSplat adapts multi-view stereo techniques (like PixelSplat) to 3DGS [S1, unofficial][S2]. VRP-UDF addresses surface reconstruction ambiguities in 3DGS by incorporating UDF priors [S1, unofficial]. GS-Net enables high-dimensional feature distillation from 2D foundation models (like CLIP) into 3DGS using quantile rendering [S1, unofficial]. Masked Autoencoders have been adapted for 3DGS to learn generalizable representations [S2].

## Sources
[S1] Quantile Rendering: Efficiently Embedding High-dimensional Feature on 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2512.20927, fetched 2026-10-26 (unofficial)
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
