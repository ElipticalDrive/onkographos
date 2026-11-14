# Lighting and Appearance Models

Also called: Appearance Conditioning, Illumination Modeling, View-Dependent Rendering

## Overview
Lighting and Appearance Models modify the standard NeRF radiance field to handle complex illumination, view-dependent effects, and environmental variations that a single static color MLP cannot capture. They address the limitation where standard NeRF assumes a fixed lighting environment and uniform material properties, failing in "in-the-wild" scenes with changing lighting or dynamic transients [S1]. This family sets itself apart by explicitly decoupling geometry from appearance or replacing implicit color prediction with physically-based lighting integrations, enabling robust novel view synthesis under varying conditions [S1].

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
Decouples static scene geometry from per-image appearance changes and transient objects using image-specific embeddings. Choose when dealing with uncontrolled "in-the-wild" photos where lighting and content vary significantly across views.

### [Neural Pre-Integrated Lighting (Neural-PIL)](13783-neural-pre-integrated-lighting-neural-pil.md)
Replaces numerical light integration with a neural network query for pre-integrated light, allowing for fine-grained illumination details and relighting. Choose when accurate, view-dependent specular highlights and complex environment lighting are required, surpassing the limitations of Spherical Harmonics.

### [Spherical Harmonics NeRF](14480-spherical-harmonics-nerf.md)
Replaces the view-dependent color MLP with an explicit 2nd-order Spherical Harmonics (SH) illumination model. Choose when explicit relighting capabilities are needed with lower computational overhead than full neural lighting models.

## Background
The standard NeRF formulation [S6, unofficial] assumes a static scene with fixed lighting, predicting color directly from position and view direction. This approach fails in real-world scenarios ("in-the-wild") where lighting changes between shots or transient objects (people, cars) appear inconsistently [S1]. To address this, NeRF-W introduced appearance embeddings to condition color prediction on specific images, separating static and transient components [S1]. Concurrently, efforts to improve material realism led to Neural-PIL, which uses SIREN-based networks to approximate pre-integrated lighting, offering a more physically plausible alternative to simple view-dependent color MLPs or low-order SH expansions [S1, unofficial].

## Sources
[S1] NeRF in the Wild (NeRF-W) and Neural-PIL notes, URL not provided, date fetched not provided.
