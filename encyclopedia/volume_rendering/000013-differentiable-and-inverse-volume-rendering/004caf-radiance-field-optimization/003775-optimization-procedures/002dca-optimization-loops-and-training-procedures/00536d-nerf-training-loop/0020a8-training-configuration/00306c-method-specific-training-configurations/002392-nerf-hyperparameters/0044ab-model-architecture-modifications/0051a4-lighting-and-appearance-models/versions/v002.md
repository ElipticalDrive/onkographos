# Lighting and Appearance Models

Also called: Per-image appearance conditioning, Explicit illumination modeling, Transient object handling

## Overview
Lighting and Appearance Models modify the standard NeRF color prediction head to decouple scene geometry from illumination variations and transient artifacts. They solve the problem of rendering failure in real-world image collections where lighting changes, shadows, and moving objects cause significant per-image appearance differences that a single static MLP cannot capture [S1]. This family sets itself apart from analytical lighting models (like Spherical Harmonics) by either learning implicit appearance embeddings (NeRF-W) or replacing the view-dependent MLP with explicit pre-integrated lighting queries (Neural-PIL) [S1].

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

### [NeRF in the Wild (NeRF-W)](11995-nerf-in-the-wild-nerf-w.md)
NeRF-W addresses real-world image collections containing per-image appearance variations and transient objects. It introduces per-image appearance embeddings $\mathbf{z}_i$ to condition the static color prediction and a separate transient network with embeddings $\mathbf{t}_i$ to model occluders. Unlike Spherical Harmonics NeRF, which models lighting analytically, NeRF-W learns appearance variations implicitly.

### [Neural Pre-Integrated Lighting (Neural-PIL)](13783-neural-pre-integrated-lighting-neural-pil.md)
Neural-PIL replaces numerical light integration with a neural query predicting pre-integrated light $\tilde{L}_i(\omega_r, b_r)$ from an illumination embedding, direction, and roughness. It uses FiLM-SIREN layers modulated by mapping networks to capture fine illumination details, differing from SH or SG representations by avoiding basis function limitations.

### [Spherical Harmonics NeRF](14480-spherical-harmonics-nerf.md)
Spherical Harmonics NeRF replaces the view-dependent color MLP with an explicit 2nd-order SH illumination model. It allows direct relighting by varying learnable SH coefficients $\mathbf{L}$, addressing the lack of semantic lighting control in standard NeRF. It differs from NeRF-W by explicitly modeling illumination rather than using appearance embeddings, and from Neural-PIL by using a global SH basis.

## Background
Standard NeRF assumes a static scene with consistent lighting [S6, unofficial]. Real-world captures often violate this, leading to artifacts. NeRF-W [S1] and SH-NeRF [S1] were developed to handle these variations. Neural-PIL [S1, unofficial] was introduced to efficiently approximate complex lighting interactions without the cost of full path tracing or the rigidity of low-order SH.

## Sources
[S1] NeRF in the Wild (NeRF-W) notes
[S6] Standard NeRF Architecture notes (unofficial)
