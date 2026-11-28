# Network Structure Modifications

Also called: Network Architectures, Multi-input MLP (mi-MLP), NeRF Model Architecture

## Overview
Network Structure Modifications alter the baseline continuous volumetric representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ [G] to address specific constraints such as dynamic scenes , noise [S1, unofficial], semantic understanding [S1, unofficial], or inference speed . These modifications retain the core volume rendering equation but change the input space, network topology, or output fields to enable features like temporal consistency, object manipulation, or noise-aware rendering [S1, unofficial].

## Shared principles
The baseline NeRF architecture [Parent](3065-nerf-hyperparameters.md) defines a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is the spatial position, $\mathbf{d} \in \mathbb{R}^3$ is the viewing direction, $\sigma$ is the volume density, and $\mathbf{c} \in \mathbb{R}^3$ is the view-dependent color [G]. The rendering process integrates these outputs along rays using standard volume rendering equations [G].

Most modifications in this family retain the core volume rendering equation but alter one or more of the following components:
1.  **Input Space:** Augmenting $\mathbf{x}$ and $\mathbf{d}$ with additional signals such as latent codes $\mathbf{z}$, time $t$, illumination embeddings $\mathbf{z}_l$, or external feature vectors $\mathbf{f}$ from vision models [S1, unofficial].
2.  **Network Topology:** Replacing the monolithic MLP with specialized branches (e.g., separate density and color networks), modular experts, or hybrid structures combining MLPs with convolutional or transformer backbones [S1, unofficial].
3.  **Output Space:** Extending the output tuple $(\sigma, \mathbf{c})$ to include additional fields such as uncertainty, semantic logits, or physical properties like albedo and normals [S1, unofficial].

## Subtopics

### [Network Architectures](15530-network-architectures.md)
Network architectures in neural radiance fields modify the baseline continuous volumetric representation to address specific constraints such as dynamic scenes, noise, semantic understanding, or inference speed. These architectures alter the network topology, input space, or output fields to enable features like temporal consistency, object manipulation, or noise-aware rendering.

### [Multi-input MLP (mi-MLP)](16599-multi-input-mlp-mi-mlp.md)
The multi-input MLP (mi-MLP) is a network architecture modification for Neural Radiance Fields (NeRF) that incorporates the raw input embeddings (3D location and viewing direction) into every layer of the network to prevent overfitting in few-shot view synthesis without reducing model capacity. Unlike the vanilla NeRF architecture [Parent](20497-network-structure-modifications.md), which relies on deep hierarchical mappings where each layer only sees the output of the previous one, mi-MLP ensures that every layer is explicitly aware of the input embeddings, creating shorter paths between inputs and outputs. This structural change alleviates overfitting by balancing gradient amplitudes across layers, allowing the model to synthesize high-frequency details with fewer effective parameters.

### NeRF Model Architecture
#### definition
- [S1, unofficial] The neural network architecture for NeRF, consisting of dense layers with ReLU activations, potentially using view directions as input.
#### parameter
- [S1, unofficial] `D`: The depth of the network (number of hidden layers). Default is 8.
- [S1, unofficial] `input_ch_views`: The number of input channels for view directions. Default is 3.
- [S1, unofficial] `output_ch`: The number of output channels. Default is 4 (RGB + density).
- [S1, unofficial] `use_viewdirs`: Boolean flag to determine if view directions are used as input to the network.

## Background
The baseline NeRF architecture [Parent](3065-nerf-hyperparameters.md) defines a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ [G]. The rendering process integrates these outputs along rays using standard volume rendering equations [G].

## Sources
[S1] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
