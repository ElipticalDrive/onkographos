# Mixture-of-Experts and Modular Architectures

Also called: MoE-INR, MIMO-NeRF

## Overview
This family encompasses architectures that replace or augment the single continuous function of standard NeRF with multiple specialized networks (experts) or parallel processing streams. It solves the limitations of conventional Implicit Neural Representations (INRs), which struggle with complex features using a single network, and grid-based INRs, which suffer from boundary discontinuities [S1, MIMO-NeRF](13765-mixture-of-experts-implicit-neural-representation.md). It sets itself apart from siblings like [DoF-NeRF](9432-dof-nerf.md) (which handles depth-of-field) or [Feature Fields](12995-feature-fields.md) (which add semantic channels) by explicitly partitioning the input domain or ray batch among distinct neural modules to improve compression, training speed, or representation capacity.

## Shared principles
The standard NeRF architecture [S6, unofficial] consists of two Multi-Layer Perceptrons (MLPs): a coarse network and a fine network. The coarse network takes positional encoded position $\gamma(\mathbf{x})$ and outputs a latent feature vector $\mathbf{h}$ and volume density $\sigma$. The fine network takes $\mathbf{h}$, positional encoded position $\gamma(\mathbf{x})$, and positional encoded view direction $\gamma(\mathbf{d})$ to output color $\mathbf{c}$.

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

### [Mixture-of-Experts Implicit Neural Representation](13765-mixture-of-experts-implicit-neural-representation.md)
[Mixture-of-Experts Implicit Neural Representation](13765-mixture-of-experts-implicit-neural-representation.md) is an architecture for time-varying volumetric data compression that automatically subdivides spatiotemporal fields and dynamically assigns input coordinates to specialized expert networks. It solves the limitations of conventional INRs (which struggle with complex features using a single network) and grid-based INRs (which suffer from boundary discontinuities due to manual decomposition) by using a shared encoder to maintain cross-boundary continuity while allowing experts to specialize in distinct subdomains. Unlike siblings like [MIMO-NeRF-self-R2](11400-mimo-nerf-self-r2.md) which may use multiple networks for different aspects of rendering, MoE-INR specifically targets the compression and representation of scientific volumetric data through a unified framework that can emulate conventional, grid-based, or ensemble INRs by adjusting the number of experts and pre-training schemes.

### [MIMO-NeRF-self-R2](11400-mimo-nerf-self-r2.md)
[MIMO-NeRF-self-R2](11400-mimo-nerf-self-r2.md) is a specific configuration of the MIMO-NeRF architecture [MIMO-NeRF](13765-mixture-of-experts-implicit-neural-representation.md) that utilizes two reformulated Multi-Input Multi-Output (MIMO) Multi-Layer Perceptrons (MLPs) with reduction factors $R^1=1$ and $R^2=1$. It solves the problem of high computational cost in standard NeRF training by grouping rays and processing them in parallel through shared MLPs, thereby reducing the total number of MLP evaluations per training step. It sets itself apart from the base MIMO-NeRF [MIMO-NeRF](13765-mixture-of-experts-implicit-neural-representation.md) by employing a specific self-grouping strategy with fixed reduction factors, and from its sibling MIMO-NeRF-self-R1 [MIMO-NeRF-self-R1](13765-mixture-of-experts-implicit-neural-representation.md) by using a different distribution of reduction factors across the two MLP stages.

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
The Mixture-of-Experts Implicit Neural Representation (MoE-INR) was introduced to address the inefficiencies of single-network INRs for complex scientific volumetric data [S1]. The MIMO-NeRF family, including the self-R variants, was proposed to accelerate neural rendering by reducing the computational burden of per-ray MLP evaluations through multi-input multi-output structures [S1]. These methods are used in contexts requiring efficient compression of spatiotemporal fields or fast training/inference for large-scale scenes where standard NeRF is prohibitively expensive [S1].

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
