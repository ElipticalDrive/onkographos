# Model Architecture and Encoding


## Overview
This family defines the structural components and embedding mechanisms used within NeRF training configurations to map spatial coordinates and viewing directions to radiance field properties. It addresses the problem of how to represent the continuous scene function efficiently and accurately, distinguishing between standard MLP-based encodings, hash-based sparse representations, and architectural modifications for specific priors like depth or semantic style.

## Shared principles
All methods in this family operate within the differentiable volume rendering framework, where a neural network predicts density ($\sigma$) and color ($c$) for points sampled along camera rays. The core mechanism involves transforming 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ into feature vectors. These methods share the requirement of defining a network topology (typically Multi-Layer Perceptrons) and an input encoding strategy. The output is integrated via volumetric rendering to produce pixel colors. The specific choice of architecture determines the trade-off between memory usage, training speed, and reconstruction fidelity.

## Subtopics

### [HashNeRF Parameters](3068-hashnerf-parameters.md)
Uses multi-resolution hashed embedding tables indexed by voxel coordinates instead of fixed grids or positional encoding. Chosen for high memory efficiency and fast training on large scenes.

### [Depth-Guided NeRF Training Configuration](11916-depth-guided-nerf-training-configuration.md)
Modifies the standard NeRF architecture by adding dropout layers to the positional encoding path to integrate depth priors. Chosen when monocular depth estimates are available to guide geometry reconstruction and reduce ambiguity.

### [CLIPNeRF Training Configuration](12073-clipnerf-training-configuration.md)
Freezes density-related parameters and modifies the optimization target to use CLIP embeddings for text-guided stylization. Chosen for semantic style transfer tasks where geometric consistency must be preserved while altering appearance.

## Background
The standard NeRF architecture [S1] introduced the use of fully connected networks with positional encoding. Subsequent works like HashNeRF [S1] introduced hash-based embeddings to scale to larger scenes. Depth-guided approaches [S1] adapted the architecture to handle noisy depth inputs, while CLIPNeRF [S1, S2] adapted the training loop and parameter freezing strategies for semantic guidance.

## Sources
[S1] https://github.com/bmild/nerf (unofficial notes)
[S2] https://github.com/dasparthik/clipnerf
