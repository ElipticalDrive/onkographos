# Neural Light Field Representations

Also called: NeLF, Neural Light Fields

## Overview
Neural Light Field Representations map camera ray parameters directly to final pixel color, bypassing the volumetric ray-marching and alpha-composition steps required by Neural Radiance Fields (NeRF) [S2]. Unlike NeRF, which predicts density and radiance at sampled points along a ray requiring multiple forward passes per pixel, NeLF performs a single forward pass per ray to output the integrated RGB value directly [S2]. This approach enables orders-of-magnitude faster rendering speeds and reduced computational complexity (FLOPs) while maintaining or exceeding the visual quality of NeRF [S2].

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

### [Neural Light Field (NeLF)](9257-neural-light-field-nelf.md)
Neural Light Field (NeLF) represents a 3D scene as a direct mapping from camera ray parameters to final pixel color, bypassing the volumetric ray-marching and alpha-composition steps required by Neural Radiance Fields (NeRF). Unlike NeRF, which predicts density and radiance at sampled points along a ray requiring multiple forward passes per pixel, NeLF performs a single forward pass per ray to output the integrated RGB value directly. This approach enables orders-of-magnitude faster rendering speeds and reduced computational complexity (FLOPs) while maintaining or exceeding the visual quality of NeRF. The method is distinguished by its use of a deep residual Multi-Layer Perceptron (MLP) architecture and knowledge distillation from a pre-trained NeRF teacher model to handle the data scarcity inherent in training such deep networks.

### Neural Multi-Light Implicit Representation (Stage 1)
An extension of Neuralangelo that incorporates light position as an input to the network to enable multi-light implicit representation [S1]. Input includes camera pose, light position, and points (x, y, z) [S1]. Training time for Stage 1 follows Neuralangelo training time; Stage 2 takes 19 hours [S1]. Experiments were conducted on a single Nvidia RTX 3090 [S1].

## Background
Not given in the sources.

## Sources
[S1] MLI-NeRF: Multi-Light Intrinsic-Aware Neural Radiance Fields, https://arxiv.org/abs/2411.17235v1, fetched 2026-10-26
