# Geometric Priors and Depth Integration

Also called: Geometric Regularization, Depth-Guided NeRF, Explicit Geometry Constraints

## Overview
This family of methods enhances standard Neural Radiance Fields (NeRF) by integrating explicit geometric constraints—such as Structure-from-Motion (SfM) features, monocular depth priors, or signed distance fields (SDF)—into the optimization loop. While standard NeRF relies solely on photometric consistency to infer geometry, these methods solve the ambiguity inherent in sparse-view or ill-posed reconstruction by enforcing structural consistency, depth ordering, or metric surface accuracy. They differ from siblings like [AdaNeRF](6195-adanerf.md) or [SinNeRF](6199-sinnerf.md) by explicitly leveraging external geometric signals (depth maps, epipolar geometry, or cost volumes) rather than relying purely on implicit neural representations or frequency-based positional encoding.

## Shared principles
All methods in this family share the mechanism of mapping 6D input coordinates to 4D output values using a differentiable neural network, allowing the radiance field to be optimized end-to-end through the volumetric rendering integral.

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs, which struggle to fit high-frequency details. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [S1, unofficial][S3]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [S1, unofficial][S3].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [S1, unofficial][S3].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1, unofficial][S3]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1, unofficial][S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [S1, unofficial][S3].

## Subtopics

### [SfMNeRF](8664-sfmnerf.md)
Integrates SfM constraints (matched features, epipolar geometry) to reduce geometric ambiguity in sparse views. Choose when camera poses are known but feature correspondence is available.

### [DASNeRF](8668-dasnerf.md)
Uses monocular depth priors (DepthAnythingV2) and dynamic sampling for sparse-view reconstruction. Choose when high-quality monocular depth estimates are available and uniform sampling is insufficient.

### [GeoNeRF](8710-geonerf.md)
Replaces the standard MLP with a geometric morphology reasoner using 3D cost volumes and Transformers. Choose when generalization to new scenes is required and explicit multi-view stereo cues are beneficial.

### [Neural RGB-D Surface Reconstruction](11352-neural-rgb-d-surface-reconstruction.md)
Uses a hybrid TSDF/radiance field representation for metric 3D reconstruction from RGB-D sequences. Choose when watertight mesh extraction and metric accuracy are prioritized over appearance.

### [DINER](11354-diner.md)
Conditions the NeRF on depth deviation from estimated depth maps to handle large disparities. Choose when source cameras have high relative disparity and implicit correspondence fails.

## Background
The integration of geometric priors into NeRF-based optimization emerged to address the limitations of implicit representations in sparse-view settings, where photometric loss alone leads to geometric ambiguities and artifacts. Early works like [SfMNeRF](8664-sfmnerf.md) introduced explicit cross-view constraints [S1]. Subsequent methods leveraged deep monocular depth estimators (e.g., DepthAnythingV2) to provide dense geometric priors [S1], while others explored hybrid representations like TSDFs for direct mesh extraction [S1]. The field has evolved from simple depth regularization to complex architectures involving cost volumes [S1] and depth-deviation conditioning [S1].

## Sources
