# Lighting and Intrinsic Decomposition

Also called: Inverse Rendering, Intrinsic Decomposition, Relightable NeRF

## Overview
Lighting and Intrinsic Decomposition refers to methods that separate a Neural Radiance Field (NeRF) into view-independent intrinsic properties (such as albedo, normals, and density) and view-dependent lighting components, enabling realistic relighting and scene understanding [16200-urbanir.md]. Unlike standard NeRFs that bake lighting into the observed color, these methods explicitly model the illumination environment (e.g., sun, ambient, sky) or decompose color into lighting and reflectance bases to solve the ill-posed inverse rendering problem [16200-urbanir.md, 18127-unsupervised-decomposition-and-nerf.md].

## Shared principles
All methods in this family share the core mechanism of decomposing the radiance field output into distinct semantic or physical components rather than a single RGB color value.

1.  **Intrinsic Representation:** The scene is represented by attributes $\mathbf{A}$ (e.g., albedo $\mathbf{a}$, normal $\mathbf{n}$, semantics $\mathbf{s}$, density $\sigma$) or a decomposition of color into a lighting component $v$ and a reciprocal color basis $r$ [16200-urbanir.md, 18127-unsupervised-decomposition-and-nerf.md].
2.  **Volume Rendering of Intrinsics:** These attributes are accumulated along rays using standard alpha-composition weights $w_i$ [16200-urbanir.md, 18127-unsupervised-decomposition-and-nerf.md]:
    $$ w_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) (1 - \exp(-\sigma_i \delta_i)) $$
    The rendered intrinsic value $\mathbf{I}(\mathbf{r})$ for a ray $\mathbf{r}$ is:
    $$ \mathbf{I}(\mathbf{r}) = \sum_{i=1}^{N} w_i \mathbf{A}_i $$
3.  **Lighting Model Integration:** The final color is computed by combining rendered intrinsics with a lighting model $\mathcal{L}$ [16200-urbanir.md]. For example, UrbanIR uses a sun, ambient, and sky model [16200-urbanir.md], while Unsupervised Decomposition uses a neural enhancement function $\phi$ to modify the lighting component $v$ [18127-unsupervised-decomposition-and-nerf.md].
4.  **Ill-posed Constraint:** Since decomposition is ill-posed, these methods rely on priors (e.g., gray-world, smoothness, multi-view consistency) to constrain the optimization [16200-urbanir.md, 18127-unsupervised-decomposition-and-nerf.md].

## Subtopics

### UrbanIR
UrbanIR is a method for the inverse rendering of large-scale, unbounded urban scenes from a single monocular video under single illumination [16200-urbanir.md]. It infers shape, albedo, surface normals, semantics, and sun/sky illumination to enable realistic relighting and night simulation [16200-urbanir.md]. Unlike standard NeRFs that bake lighting into color, or methods requiring LiDAR/multiple cameras (e.g., FEGR, LightSim) or multiple light sources (e.g., NeRF-OSR), UrbanIR uses only RGB video and known camera poses [16200-urbanir.md]. It distinguishes itself by using a spatial hash-based voxel representation (Instant-NGP) for scalability and a visibility estimation technique that avoids per-scene optimized MLP heads for shadows, allowing the sun position to change during relighting [16200-urbanir.md].

### [Unsupervised Decomposition and Enhancement in NeRF](18127-unsupervised-decomposition-and-enhancement-in-nerf.md)
This method decomposes the view-dependent color in a Neural Radiance Field (NeRF) into a view-dependent lighting component $v$ and a view-independent reciprocal color basis $r$, then enhances $v$ using a neural network to produce an enhanced image $I_e$ without ground truth supervision [18127-unsupervised-decomposition-and-nerf.md]. It sets itself apart from 2D Retinex-based methods by leveraging 3D geometry and multi-view consistency to constrain the ill-posed decomposition, allowing for unsupervised training [18127-unsupervised-decomposition-and-nerf.md].

## Background
Inverse rendering in NeRFs emerged as a way to move beyond static appearance synthesis to scene understanding and relighting. UrbanIR [16200-urbanir.md] adapted these concepts to large-scale urban environments using hash-based representations for efficiency. Unsupervised Decomposition [18127-unsupervised-decomposition-and-nerf.md] addressed low-light enhancement by decomposing color into lighting and reflectance components, leveraging 3D consistency to avoid the need for paired low/high-light training data.

## Sources
[S1] UrbanIR and Unsupervised Decomposition notes provided in the prompt.
