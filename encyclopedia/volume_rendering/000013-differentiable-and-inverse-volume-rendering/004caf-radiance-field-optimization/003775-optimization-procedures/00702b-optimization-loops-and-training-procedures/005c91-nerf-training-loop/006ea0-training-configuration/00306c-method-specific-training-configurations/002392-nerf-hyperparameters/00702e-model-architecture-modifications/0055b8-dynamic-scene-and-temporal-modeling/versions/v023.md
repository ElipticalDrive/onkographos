# Dynamic Scene and Temporal Modeling

Also called: Dynamic NeRF, 4D Radiance Fields, Temporal NeRF

## Overview
Dynamic Scene and Temporal Modeling extends static Neural Radiance Fields (NeRF) to reconstruct 4D radiance fields $R(\mathbf{x}, \mathbf{d}, t)$ from multi-view video sequences where scene geometry and appearance change over time [S1]. It solves the ambiguity between camera and object motion by explicitly conditioning the network on temporal coordinates, employing deformation fields, or using particle-based representations [S1]. Unlike static NeRF, which suffers from severe ghosting and blurriness on dynamic data, these methods separate or warp dynamic elements to enable novel view synthesis at arbitrary times [S1].

## Shared principles
All methods in this family aim to predict volume density $\sigma$ and view-dependent color $\mathbf{c}$ for a ray $\mathbf{r}(s) = \mathbf{o} + s\mathbf{d}$ at time $t$ using the standard volume rendering integral [S1]:

$$
C(\mathbf{r}, t) = \int_{s_n}^{s_f} T(s, t) \sigma(\mathbf{r}(s), t) c(\mathbf{r}(s), \mathbf{d}, t) ds
$$

where $T(s, t) = \exp \left( -\int_{s_n}^{s} \sigma(\mathbf{r}(s'), t) ds' \right)$ is the accumulated transmittance [S1].

The core challenge is modeling the temporal variation of $\sigma$ and $\mathbf{c}$. Common strategies include:
1.  **Direct Conditioning:** Concatenating time $t$ (often with positional encoding) to the input of an MLP alongside spatial coordinates $\mathbf{x}$ [S1].
2.  **Deformation Fields:** Learning a displacement $\Delta \mathbf{x}$ that warps dynamic points $\mathbf{x}$ to a canonical static space $\mathbf{x}' = \mathbf{x} + \Delta \mathbf{x}$, where a static radiance field is queried [S1].
3.  **Hybrid Representations:** Superimposing a static grid-based field with a dynamic component (e.g., particles or sparse latent codes) weighted by a mask $w_t(\mathbf{x})$ [S1].

## Subtopics

### [Gear-NeRF](13848-gear-nerf.md)
Replaces the unified 4D field with stratified "gears" (feature volumes) assigned by semantic motion scale. High-motion regions use higher-resolution sampling. Choose for object tracking and high-fidelity dynamic synthesis.

### [NeRF with Time Parameter](14705-nerf-with-time-parameter.md)
A baseline that concatenates time $t$ to the spatial input of a standard MLP. Choose for simplicity; it lacks detailed dynamic reasoning and suffers from blurriness compared to deformation-based methods.

### [Forward Warping for Novel View Synthesis](15541-forward-warping-for-novel-view-synthesis.md)
Models a static canonical field and forward warps it to target times using differentiable splatting and an inpaint network. Choose for handling many-to-one mapping difficulties in dynamic scenes.

### [Dynamic Appearance Particle Neural Radiance Field](15550-dynamic-appearance-particle-neural-radiance-field.md)
Superimposes a static grid with a dynamic particle field for monocular video. Choose for single-view dynamic reconstruction where particles model only moving elements.

### [KFD-NeRF](15551-kfd-nerf.md)
Uses a Kalman filter to combine history-based motion prediction with current observations for deformation estimation. Choose for leveraging inductive smoothness bias of deformation fields over feature interpolation.

### [SLS4D](16576-sls4d.md)
Uses sparse latent codes aggregated via attention and a deformation field. Choose for reducing parameters while maintaining high quality by abandoning direct coordinate-to-feature mapping.

### ER-NeRF
- **Definition**: A Conditional NeRF framework for high-accuracy dynamic portrait creation synchronized with audio [S1].
- **Steps**: Utilizes spatial regions' distinct contributions to steer dynamic head modeling; develops a Tri-Plane Hash Representation; introduces Adaptive Pose Encoding to separate head and torso [S1].
- **Pitfall**: Additional fine-tuning may be necessary for poses outside the training dataset [S1].
- **Performance**: Attains instantaneous rendering and rapid convergence with a compact model size [S1].

### NeRF-MS
- **Definition**: A method that diminishes vagueness in visual alterations using a ternary loss to standardize visual code spread [S1].
- **Steps**: Suggests a temporary decomposition unit to distinguish still scenes from dynamic objects [S1].
- **Pitfall**: May struggle with significant sequence discrepancies, such as severe lighting changes [S1].
- **Performance**: Attains cutting-edge results in multi-sequence scenarios, enhancing efficiency and minimizing ghost effects [S1].

## Background
The extension of NeRF to dynamic scenes emerged to address the "ghosting" artifacts inherent in static models when applied to video data [S1]. Early approaches treated time as an additional input dimension to the MLP, serving as a baseline for more complex deformation-based methods [S1]. Subsequent developments introduced canonical space warping, particle systems, and sparse latent representations to better handle the many-to-one mapping ambiguities and computational costs of 4D reconstruction [S1]. These methods are primarily used in 3D perception, dynamic portrait reconstruction, and novel view synthesis for video [S1].

## Sources
[S1] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
