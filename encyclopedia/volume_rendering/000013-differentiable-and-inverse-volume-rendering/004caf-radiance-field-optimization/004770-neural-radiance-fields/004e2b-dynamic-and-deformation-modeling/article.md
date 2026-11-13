# Dynamic and Deformation Modeling

Also called: Dynamic NeRF, Time-varying NeRF, Deformable NeRF

## Overview
Dynamic and Deformation Modeling extends static Neural Radiance Fields to reconstruct time-varying scenes from multi-view video or sparse image collections, solving the problem of novel view synthesis for non-static content. It sets itself apart from static NeRF variants by explicitly incorporating temporal information, either through direct time-conditioned radiance fields or by warping spatial coordinates to a canonical space, allowing for the coherent reconstruction of moving objects that would otherwise appear inconsistent in a static model.

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [44-radiance-field-optimization.md]. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) [44-radiance-field-optimization.md].

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [44-radiance-field-optimization.md]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [44-radiance-field-optimization.md].

Differentiation strategies fall into three main categories [44-radiance-field-optimization.md]:
1. **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [44-radiance-field-optimization.md].
2. **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [44-radiance-field-optimization.md].
3. **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [44-radiance-field-optimization.md].

## Subtopics

### [Deformation Field Parametrization](12731-deformation-field-parametrization.md)
Deformation Field Parametrization addresses the limitation of static Neural Radiance Fields (NeRFs) by enabling the representation of dynamic, non-rigid scenes. It solves the problem of reconstructing time-varying geometry and appearance by mapping 3D spatial coordinates and time into a static "canonical" space, where a standard static NeRF can be applied. What sets this family apart from static NeRF variants is the explicit modeling of temporal evolution through a deformation function, allowing for the reconstruction of moving objects from sparse or multi-view data that would otherwise be incoherent in a static model.

### [Dynamic Scene Representation](13340-dynamic-scene-representation.md)
Dynamic Scene Representation extends static Neural Radiance Fields (NeRF) to handle time-varying content, solving the problem of novel view synthesis for video sequences. It sets itself apart from static NeRF by incorporating temporal information into the radiance field prediction, distinguishing it from its sibling [Deformation Field Parametrization](12731-deformation-field-parametrization.md) which focuses on the geometric mapping, and from [Deformable Neural Radiance Fields (Nerfies)](13428-deformable-neural-radiance-fields-nerfies.md) which specifically targets non-rigid human deformation using canonical space warping.

### [Deformable Neural Radiance Fields (Nerfies)](13428-deformable-neural-radiance-fields-nerfies.md)
A method for reconstructing non-rigidly deforming scenes by mapping observation-space points to a canonical template space using a learned deformation field conditioned on a per-frame latent code.

## Background
The extension of NeRF to dynamic scenes originated with the recognition that static radiance fields cannot represent temporal changes [S1]. Early approaches focused on direct time-conditioning, where the temporal variable is concatenated with spatial coordinates to predict density and color [S1]. Subsequent work introduced deformation fields to handle non-rigid motion, mapping points from observation space to a canonical space to leverage static NeRF priors [S1, S3, unofficial]. These methods are primarily used in novel view synthesis for video, 4D reconstruction, and dynamic scene analysis.

## Sources
[S1] Dynamic Neural Radiance Fields, URL, date fetched
[S2] Deformation Field Parametrization, URL, date fetched
[S3] Deformable Neural Radiance Fields (Nerfies), URL, date fetched
