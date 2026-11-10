# Geometric Priors and Depth Integration

Also called: Geometric Regularization, Depth-Guided NeRF

## Overview
This family of methods enhances Neural Radiance Fields (NeRF) by integrating explicit geometric constraints—such as Structure-from-Motion (SfM) features, monocular depth priors, or RGB-D sensor data—into the optimization loop. While standard NeRF relies solely on photometric consistency to resolve the shape-radiance ambiguity, these methods enforce geometric consistency to improve reconstruction quality under sparse input conditions or to extract explicit mesh geometry [S1]. They differ from sibling architectures like [AdaNeRF](6195-adanerf.md) or [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md) by explicitly coupling the implicit radiance field with external geometric signals (depth, normals, or epipolar geometry) rather than relying purely on volumetric density regularization or sparse voxel structures.

## Shared principles
The core mechanism remains the parametric function $F_\Theta$ that maps a 3D position $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$ to a volume density $\sigma \in \mathbb{R}$ and an RGB color $\mathbf{c} \in \mathbb{R}^3$. This function is typically implemented as one or more neural networks.

**Input Representation:**
Raw coordinates are rarely used directly due to the high-frequency nature of natural images. They are mapped to a higher-dimensional space using positional encoding $\gamma(\mathbf{x})$. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [Parent].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [Parent].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [Parent]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [Parent].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [Parent].

## Subtopics

### SfMNeRF
SfMNeRF improves novel view synthesis and 3D geometry reconstruction under sparse input conditions by integrating Structure-from-Motion (SfM) constraints into the training loop [S1]. It distinguishes itself from standard NeRF and its sibling DASNeRF by explicitly enforcing geometric consistency through matched feature positions, epipolar geometry, and multi-view photometric consistency, thereby reducing the shape-radiance ambiguity and preventing convergence to local minima [S1].

### DASNeRF
DASNeRF is a NeRF optimization method for sparse-view 3D reconstruction that integrates monocular depth priors to guide learning and employs a dynamic sampling mechanism to preserve geometric details [S1]. It distinguishes itself from standard NeRF and its sibling [SfMNeRF](8664-sfmnerf.md) by explicitly enforcing relative depth ordering fidelity and structural consistency using DepthAnythingV2 priors, combined with a five-dimensional input fusion strategy in the MLP to enhance detail preservation under sparse input conditions [S1].

### [Neural RGB-D Surface Reconstruction](11352-neural-rgb-d-surface-reconstruction.md)
Neural RGB-D Surface Reconstruction solves the problem of generating metric, clean triangle meshes from color and depth data, addressing the limitation of standard NeRF which produces noisy, semi-transparent volumes unsuitable for explicit geometry extraction. It sets itself apart by replacing the density-based volumetric representation with a hybrid model: a Truncated Signed Distance Field (TSDF) for geometry and a volumetric radiance field for appearance, optimized jointly with camera poses.

### DINER
DINER is a method for novel view synthesis that reconstructs a volumetric scene representation (NeRF) from a sparse set of RGB input views, specifically designed to handle high relative distances between source cameras [S1]. It differs from standard image-based NeRFs by leveraging a state-of-the-art depth estimator to predict dense depth maps for each source view, which are then used to guide implicit geometry estimation and improve sampling efficiency [S1]. Unlike methods relying on sparse landmarks or implicit correspondence search, DINER uses dense depth maps that are not restricted to specific object categories, allowing for reconstruction from views with large disparities [S1].

### GeoNeRF
**Definition**
GeoNeRF is a model with two parts: a geometric morphology reasoner and a renderer. The reasoner builds a cascaded cost volume for source view sequences. The renderer infers geometric and appearance representations using a Transformer-based attention mechanism and cascaded cost volumes, then renders fine images using volume rendering [S1].

**Steps**
1.  The geometric morphology reasoner builds a cascaded cost volume for the source view sequence [S1].
2.  The renderer uses a Transformer-based attention mechanism and the cascaded cost volumes to infer geometric and appearance representations [S1].
3.  Volume rendering technology is used to render fine images [S1].

## Background
The integration of geometric priors into NeRF optimization emerged to address the "shape-radiance ambiguity" inherent in single-view or sparse-view reconstruction, where multiple 3D configurations can produce the same 2D image [S1]. Early methods like SfMNeRF and DASNeRF introduced external geometric signals (SfM features, monocular depth) to regularize the density field, ensuring that the learned geometry aligns with physical constraints [S1]. Concurrently, methods like Neural RGB-D Surface Reconstruction and DINER leveraged explicit depth sensors or estimators to bypass the implicit nature of NeRF, enabling direct mesh extraction or improved sampling in sparse-view scenarios [S1]. GeoNeRF further explored this by using cost volumes and Transformers for geometric reasoning [S1].

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
