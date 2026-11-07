# Specific Method Implementations


## Overview
This article details the specific mathematical formulations and hyperparameter configurations for the total loss functions used in distinct radiance-field optimization methods. It serves as the concrete implementation reference for the abstract concepts defined in [Total Loss Function](4177-total-loss-function.md). These implementations differ significantly in their component terms, reflecting the underlying representation (volumetric, octree, or Gaussian) and optimization objectives (surface-based, recursive decoding, or mobile-efficient) of each method.

## Shared principles
Not given in the sources.

## Subtopics

### HybridNeRF Optimization Loss
The HybridNeRF method employs a composite loss function designed to train a proposal network that balances photometric accuracy with surface regularity. The total loss $\mathcal{L}(\mathbf{r})$ for a ray $\mathbf{r}$ is defined as the sum of three terms: the photometric loss $\mathcal{L}_{\text{photo}}$, the Eikonal loss $\mathcal{L}_{\text{Eik}}$ weighted by $\lambda_{\text{Eik}}$, and the interlevel proposal loss $\mathcal{L}_{\text{prop}}$ [S1].

$$
\mathcal{L}(\mathbf{r}) := \mathcal{L}_{\text{photo}}(\mathbf{r}) + \lambda_{\text{Eik}} \mathcal{L}_{\text{Eik}}(\mathbf{r}) + \mathcal{L}_{\text{prop}}(\mathbf{r})
$$

The Eikonal loss term enforces the Eikonal equation constraint, promoting surface-like properties in the density field. The weight for this term is fixed at $\lambda_{\text{Eik}} = 0.01$ [S1]. This configuration is used during the training of the proposal network to guide the sampling strategy.

### RocNet Loss Function
RocNet utilizes a recursive octree structure where the total loss $L$ is composed of two distinct parts: the node labeling loss $L_l$ and the leaf reconstruction loss $L_r$ [S2].

$$
L = L_l + L_r
$$

The node labeling loss $L_l$ is calculated using a cross-entropy loss based on a node classifier that predicts one of four node types (empty, full, mixed, or interior) [S2]. The loss for a single node is given by:

$$
L_l = - \sum_{i=1}^{4} c_i \log(s_i)
$$

where $c_i$ is the ground-truth label for node type $i$ and $s_i$ is the classifier score for that type [S2]. The total node labeling loss for an octree is the sum of the label losses of all nodes in the tree [S2]. The leaf reconstruction loss $L_r$ is calculated from the decoded leaves, though the specific formula for $L_r$ is not provided in the source notes [S2].

### Mobile-GS Training Loss
Mobile-GS adapts the training loss from 3D Gaussian Splatting for mobile efficiency, incorporating distillation and depth constraints. The rendering loss $\mathcal{L}_{\text{rgb}}$ balances L1 and DSSIM losses:

$$
\mathcal{L}_{\text{rgb}} = \lambda \mathcal{L}_1(\mathbf{C}, \mathbf{C}_{\text{gt}}) + (1 - \lambda) \mathcal{L}_{\text{DSSIM}}(\mathbf{C}, \mathbf{C}_{\text{gt}})
$$

where $\mathbf{C}$ is the rendered image and $\mathbf{C}_{\text{gt}}$ is the ground-truth image [S3, unofficial]. The total loss $\mathcal{L}$ adds distillation and depth terms:

$$
\mathcal{L} = \mathcal{L}_{\text{rgb}} + \lambda_{\text{distill}} \mathcal{L}_{\text{distill}} + \lambda_{\text{depth}} \mathcal{L}_{\text{depth}}
$$

The hyperparameters are set as follows: $\lambda = 0.8$ balances the L1 and DSSIM contributions; $\lambda_{\text{distill}} = 1$ weights the distillation loss; and $\lambda_{\text{depth}} = 0.1$ weights the depth loss [S3, unofficial].

## Background
These specific implementations originate from distinct research papers focusing on efficient neural rendering. HybridNeRF was introduced to improve efficiency via adaptive volumetric surfaces [S1]. RocNet proposes a recursive octree network for efficient 3D deep representation [S2]. Mobile-GS targets real-time Gaussian Splatting for mobile devices, adapting standard 3DGS losses with distillation [S3, unofficial].

## Sources
[S1] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
[S2] RocNet: Recursive Octree Network for Efficient 3D Deep Representation, https://doi.org/10.1109/3dv50981.2020.00051, fetched 2026-10-26
[S3] Mobile-GS: Real-time Gaussian Splatting for Mobile Devices, https://doi.org/10.48550/arXiv.2603.11531, fetched 2026-10-29 (unofficial)
