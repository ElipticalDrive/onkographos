# Mixture-of-Experts and Modular Architectures

Also called: MoE-NeRF, MIMO-NeRF, Modular NeRF

## Overview
This family modifies the standard NeRF architecture [Parent](11313-model-architecture-modifications.md) by replacing or augmenting the single global MLP with modular structures that process inputs in parallel or route them to specialized sub-networks. It solves the computational bottleneck of evaluating a large MLP for every sampled point and the representational limitation of a single network fitting complex, heterogeneous scenes. It sets itself apart from sibling modifications like [Feature Fields](12995-feature-fields.md) (which add auxiliary outputs) or [Voxel and Sparse Grid Representations](13738-voxel-and-sparse-grid-representations.md) (which replace the MLP entirely) by retaining the continuous implicit representation while introducing internal parallelism or conditional computation.

## Shared principles
The standard NeRF architecture [Parent](11313-model-architecture-modifications.md) consists of two Multi-Layer Perceptrons (MLPs): a coarse network and a fine network. The coarse network takes positional encoded position $\gamma(\mathbf{x})$ and outputs a latent feature vector $\mathbf{h}$ and volume density $\sigma$. The fine network takes $\mathbf{h}$, positional encoded position $\gamma(\mathbf{x})$, and positional encoded view direction $\gamma(\mathbf{d})$ to output color $\mathbf{c}$.

$$
\gamma(\mathbf{x}, d_1, d_2) = [\mathbf{x}, \sin(2^{d_1}\pi\mathbf{x}), \cos(2^{d_1}\pi\mathbf{x}), \dots, \sin(2^{d_2-1}\pi\mathbf{x}), \cos(2^{d_2-1}\pi\mathbf{x})]
$$

Modifications generally fall into three categories:
1.  **Input Expansion:** Adding new conditioning vectors (e.g., appearance embeddings $\mathbf{z}_i$, fluid volume $v$, or previous stage RGB) to the input concatenation.
2.  **Output Expansion:** Adding auxiliary heads to predict values other than $\sigma$ and $\mathbf{c}$, such as semantic labels $y_{aux}$, object codes $o$, or area encoding probabilities $p$. These are rendered using the same transmittance weights $T_i$ and opacity $\alpha_i$ as color:
    $$
    \hat{Y}(r) = \sum_{i=1}^{N} T_i \alpha_i y_{aux}(x_i)
    $$
3.  **Representation Substitution:** Replacing the continuous MLP function with discrete structures (voxel grids, hash tables) or different implicit surfaces (SDFs).

## Subtopics

### [MIMO-NeRF-self-R2](11400-mimo-nerf-self-r2.md)
MIMO-NeRF-self-R2 is a specific architectural configuration of the MIMO-NeRF method. It reduces the computational cost of the original NeRF by employing two reformulated Multi-Input Multi-Output (MIMO) Multi-Layer Perceptrons (MLPs) with reduction factors $R^1=1$ and $R^2=1$. It sets itself apart from the standard NeRF architecture [Parent](11313-model-architecture-modifications.md) by processing multiple rays simultaneously through shared MLP computations, thereby reducing the total number of MLP forward passes required per training batch. It is a variant within the MIMO-NeRF family, distinguished from other variants like MIMO-NeRF-self-R1 by its specific choice of reduction factors and number of MLP stages. [11400-mimo-nerf-self-r2.md]

### [Mixture-of-Experts Implicit Neural Representation](13765-mixture-of-experts-implicit-neural-representation.md)
MoE-INR is an implicit neural representation architecture that uses a mixture-of-experts framework to automatically subdivide spatiotemporal fields and dynamically assign input coordinates to specialized expert networks for time-varying volumetric data compression. Unlike conventional INRs that use a single network for the entire field or grid-based INRs that manually decompose the field into spatial blocks (causing boundary discontinuity), MoE-INR automates irregular subdivisions and uses a shared encoder to maintain cross-boundary continuity. It is designed for scientific visualization of time-varying volumetric data, accommodating conventional, grid-based, and ensemble INR types. [13765-mixture-of-experts-implicit-neural-representation.md]

### MIMO-NeRF-self-R3
#### definition
- [S1] A variant employing $N_p-1$ reformulated MIMO MLPs with reduction factors $R^1=1, \dots, R^{N_p-1}=1$.
#### formula
- [S1] The total number of MLPs running is calculated as $\sum_{m=1}^{N_p-1} \frac{N}{N_p} R^m = N \frac{N_p-1}{N_p} < N$.
#### number
- [S1] On the Blender dataset with $N_p=4$, MIMO-NeRF-self-R3 achieves PSNR 30.89, SSIM 0.949, LPIPS 0.060, and T-time 5.05h.
- [S1] On the LLFF dataset with $N_p=4$, MIMO-NeRF-self-R3 achieves PSNR 27.27, SSIM 0.860, LPIPS 0.171, and T-time 3.78h.
- [S1] On the Blender dataset with $N_p=8$, MIMO-NeRF-self-R3 achieves PSNR 29.88, SSIM 0.937, LPIPS 0.080, and T-time 7.75h.
- [S1] On the LLFF dataset with $N_p=8$, MIMO-NeRF-self-R3 achieves PSNR 25.66, SSIM 0.797, LPIPS 0.243, and T-time 5.97h.

## Background
MIMO-NeRF was introduced in 2023 by the authors of the MIMO-NeRF paper [S1]. It addresses the high computational cost of standard NeRF training. MoE-INR is designed for scientific visualization of time-varying volumetric data [S1].

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
