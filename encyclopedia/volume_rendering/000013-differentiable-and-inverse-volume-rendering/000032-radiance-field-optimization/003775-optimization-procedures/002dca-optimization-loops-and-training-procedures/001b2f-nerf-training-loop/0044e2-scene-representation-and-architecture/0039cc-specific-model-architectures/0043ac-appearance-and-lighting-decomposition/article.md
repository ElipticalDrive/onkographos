# Appearance and Lighting Decomposition


## Overview
Appearance and Lighting Decomposition refers to a family of neural radiance field methods that explicitly separate scene geometry, material properties (such as diffuse albedo, roughness, and specular tint), and lighting conditions from the raw view-dependent color predictions of standard NeRF. This separation solves the ambiguity where standard MLPs conflate geometry and appearance, leading to "foggy" volumes, noisy normals, and unrealistic specular highlights [S1, S2]. It differs from its siblings by structuring the output space into physically plausible components (e.g., diffuse vs. specular, or view-dependent vs. reflection-based) rather than predicting a single monolithic color vector, enabling relighting, material editing, and more robust normal estimation [S1, S2].

## Shared principles
All methods in this family share the mechanism of mapping 6D input coordinates (3D position $\mathbf{x}$ and 2D viewing direction $\mathbf{d}$) to 4D output values (density $\sigma$ and color $\mathbf{c}$) using a differentiable neural network. This allows the radiance field to be optimized through the volumetric rendering integral.

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs, which struggle to fit high-frequency details. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [S1][S3]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [S1][S3].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [S1][S3].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1][S3]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1][S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [S1][S3].

## Subtopics

### [Anisotropic Neural Representation Learning](9475-anisotropic-neural-representation-learning.md)
Models volumetric density and latent features as view-dependent functions using spherical harmonics (SH) to eliminate directional ambiguity, replacing isotropic point-sampled features.

### [Ref-NeRF](10843-ref-nerf.md)
Improves specular reflection representation by separating outgoing radiance into explicit diffuse and specular components and reparameterizing the directional input by reflecting the viewing vector about the local surface normal.

### DE-NeRF
DE-NeRF decomposes lighting, geometry, and appearance to allow relighting of high-frequency environments and appearance editing [S1]. It trains a signed distance field (SDF) function using a set of 2D images to reconstruct a display grid as the input scene for the MLP [S1]. The scene is decomposed into geometric features and appearance features (diffuse reflection, roughness, specular tint) [S1]. Geometric features, appearance features, and distance $h$ (from sampling point to grid) are input into four corresponding MLPs to predict diffuse albedo, roughness value, and specular tint [S1]. Diffuse reflectance is integrated with the environment map to get diffuse color $\mathbf{c}_d$ [S1]. A specular lighting MLP $F_s$ is trained to get specular lighting $\mathbf{c}_l$ [S1]. The specular lighting $\mathbf{c}_l$ is multiplied with specular hue to get specular color $\mathbf{c}_s$ [S1]. The final color $\mathbf{c}$ is calculated as $\mathbf{c}_s + \mathbf{c}_d$ [S1]. On NeRF Synthetic and Shiny Blender data sets, PSNR is 2 points higher than latest papers [S1].

### NeRFLight
NeRFLight extends grid-based methods by dividing the scene into eight different regions to split NeRF into separate density and color fields, using a shared feature grid [S1]. This avoids increasing the number of features for seamless reconstruction while reducing stored feature values [S1]. The scene is divided into eight different regions, each corresponding to an MLP that outputs density and color [S1]. A shared feature grid is used, with features symmetrically placed at the center of the scene [S1].

### Convex Combination Color Decoder
This variant feeds reflected features $\bar{\mathbf{f}}$ into the color decoder at every sample along the camera ray, rather than just in image space, to aid convergence when geometry is fuzzy [S2]. The final color $\mathbf{c}(\mathbf{x}^{(i)}, \mathbf{d})$ is computed as a convex combination of view-dependent color $\mathbf{c}_v$ and reflection color $\mathbf{c}_r$: $\mathbf{c} = \beta \mathbf{c}_v + (1 - \beta) \mathbf{c}_r$, where $\beta$ is output by geometry encoder followed by sigmoid [S2]. The view-dependent color component $\mathbf{c}_v(\mathbf{x}^{(i)}, \mathbf{d})$ is computed using MLP $g$ with inputs $\mathbf{x}^{(i)}$, bottleneck $\mathbf{b}^{(i)}$, normal $\mathbf{n}^{(i)}$, and direction $\mathbf{d}$: $\mathbf{c}_v = g(\mathbf{x}^{(i)}, \mathbf{b}^{(i)}, \mathbf{n}^{(i)}, \mathbf{d})$ [S2]. The reflection color component $\mathbf{c}_r(\mathbf{x}^{(i)}, \mathbf{d}')$ is computed using MLP $h$ with inputs $\mathbf{x}^{(i)}$, bottleneck $\mathbf{b}^{(i)}$, normal $\mathbf{n}^{(i)}$, dot product $\mathbf{d} \cdot \mathbf{n}^{(i)}$, reflected direction $\mathbf{d}'$, and reflection feature $\bar{\mathbf{f}}$: $\mathbf{c}_r = h(\mathbf{x}^{(i)}, \mathbf{b}^{(i)}, \mathbf{n}^{(i)}, \mathbf{d} \cdot \mathbf{n}^{(i)}, \mathbf{d}', \bar{\mathbf{f}})$ [S2].

## Background
The concept of decomposing appearance and lighting in neural rendering emerged as a solution to the inherent ambiguities in standard NeRF, which predicts a single color value per ray sample, conflating material properties with illumination [S1, S2]. Early works like Ref-NeRF [S2] demonstrated that explicit modeling of specular reflections via normal prediction and reflection geometry significantly improved visual quality. Subsequent methods like DE-NeRF [S1] and NeRFLight [S1] expanded this by introducing explicit SDFs, grid-based decompositions, and multi-region architectures to facilitate relighting and material editing. These methods are primarily used in high-fidelity novel view synthesis, digital twin creation, and scenes with complex glossy or specular materials where standard NeRF fails to capture sharp highlights or consistent geometry.

## Sources
[S1] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S2] NeRF-Casting: Improved View-Dependent Appearance with Consistent Reflections, https://arxiv.org/abs/2405.14871v1, fetched 2026-10-26
