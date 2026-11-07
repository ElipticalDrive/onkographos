# Neural Light Field Representations

Also called: NeLF, R2L (Ray-to-Light), Radiance-to-Light

## Overview
Neural Light Field Representations (NeLF) replace the iterative volumetric ray-marching of Neural Radiance Fields (NeRF) with a direct mapping from camera ray parameters to final pixel color [9257-neural-light-field-nelf.md]. This eliminates the sampling bottleneck, achieving significant runtime speedups and FLOPs reduction while maintaining comparable rendering quality [9256-neural-light-field-nelf-via-r2l-distillation.md]. Unlike standard NeRF, which requires iterative alpha-composition, NeLF predicts the final RGB value in a single forward pass [9257-neural-light-field-nelf.md].

## Shared principles
All methods in this family compute a radiance field function $f_\theta(\mathbf{x}, \mathbf{d})$ that outputs volume density $\sigma$ and color $\mathbf{c}$ for a query point $\mathbf{x}$ and direction $\mathbf{d}$. The final pixel color $\mathbf{C}(\mathbf{r})$ for a ray $\mathbf{r}$ is computed via the standard volume rendering integral:
$$
\mathbf{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i
$$
where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$. The architectural variations lie in how $\sigma_i$ and $\mathbf{c}_i$ are generated from the input coordinates, how the network is structured (depth, width, connectivity), and what auxiliary inputs (embeddings, priors, features) are fused into the computation.

## Subtopics

### [R2L (Neural Light Field)](8670-r2l-neural-light-field.md)
A deep neural light field network trained via distillation from a pre-trained NeRF teacher. It uses a much deeper network with extensive internal residual blocks, trading width for depth under the same FLOPs budget. It outperforms depth-dependent methods on reflective surfaces [8670-r2l-neural-light-field.md].

### [Neural Light Field (NeLF) via R2L Distillation](9256-neural-light-field-nelf-via-r2l-distillation.md)
A specific implementation of NeLF using R2L distillation. It maps a 4D oriented ray directly to 3D RGB color in a single forward pass, bypassing NeRF's sampling bottleneck. It uses concatenated spatial coordinates of sampled points along the ray as input [9256-neural-light-field-nelf-via-r2l-distillation.md].

### [Neural Light Field (NeLF)](9257-neural-light-field-nelf.md)
The general concept of representing a 3D scene as a direct mapping from camera ray parameters to RGB values. It eliminates iterative volumetric ray-marching overhead, achieving significant runtime speedups compared to NeRF and variants like KiloNeRF [9257-neural-light-field-nelf.md].

## Background
The concept of Neural Light Fields was introduced to address the computational inefficiency of iterative volume rendering in NeRF. By learning a direct mapping from ray parameters to color, these methods aim to retain the quality of neural radiance fields while enabling real-time or near-real-time rendering. The approach relies heavily on knowledge distillation from a pre-trained NeRF teacher to supervise the training of the deep student networks, as direct training from 2D images is often infeasible for such deep architectures [9257-neural-light-field-nelf.md].

## Sources
