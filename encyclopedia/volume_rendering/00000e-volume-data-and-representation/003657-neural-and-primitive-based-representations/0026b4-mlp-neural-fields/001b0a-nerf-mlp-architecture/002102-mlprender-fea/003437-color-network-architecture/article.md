# Color Network Architecture


## Overview
The Color Network Architecture is a Multi-Layer Perceptron (MLP) module that predicts RGB color from input features, view directions, and positions. It solves the problem of view-dependent appearance rendering by decoupling geometry (handled by a preceding network) from appearance. It sets itself apart from its parent, [MLPRender_Fea](2414-mlprender-fea.md), by explicitly defining a three-layer structure with specific input dimensionality for position, view direction, and features, and by applying positional encoding to features and view direction but not position [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Position vector $\mathbf{p} \in \mathbb{R}^3$, view direction vector $\mathbf{d} \in \mathbb{R}^2$ (azimuth and elevation angles), and feature vector $\mathbf{f} \in \mathbb{R}^{12}$ [S1].
*   **Output:** Color vector $\mathbf{c} \in \mathbb{R}^3$ [S1].

**Data Structures:**
The network consists of three fully connected (FC) layers. Each layer has a width of 128 neurons [S1].

**Steps:**
1.  **Input Concatenation:** Concatenate the position $\mathbf{p}$, view direction $\mathbf{d}$, and feature $\mathbf{f}$ into a single input vector $\mathbf{x}_{in} = [\mathbf{p}; \mathbf{d}; \mathbf{f}]$ of dimension $3 + 2 + 12 = 17$ [S1].
2.  **First Hidden Layer:** Apply a fully connected layer with 128 units: $\mathbf{h}_1 = \text{ReLU}(\mathbf{W}_1 \mathbf{x}_{in} + \mathbf{b}_1)$ [S1].
3.  **Second Hidden Layer:** Apply a fully connected layer with 128 units: $\mathbf{h}_2 = \text{ReLU}(\mathbf{W}_2 \mathbf{h}_1 + \mathbf{b}_2)$ [S1].
4.  **Third Hidden Layer:** Apply a fully connected layer with 128 units: $\mathbf{h}_3 = \text{ReLU}(\mathbf{W}_3 \mathbf{h}_2 + \mathbf{b}_3)$ [S1].
5.  **Output Layer:** Apply a fully connected layer with 3 units and a Sigmoid activation function to normalize the output to the range $(0, 1)$: $\mathbf{c} = \sigma(\mathbf{W}_4 \mathbf{h}_3 + \mathbf{b}_4)$ [S1].

**Parameters:**
*   Hidden layer width: 128 [S1].
*   Activation functions: ReLU for hidden layers, Sigmoid for output [S1].

**Pitfalls:**
*   The output is constrained to $(0, 1)$ by the Sigmoid function, which may limit the dynamic range of predicted colors if not properly scaled or if the training data contains values outside this range [S1].
*   Position is not processed through positional encoding, unlike view direction and features, which may limit the network's ability to capture high-frequency spatial variations in color [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class ColorNetwork(nn.Module):
    def __init__(self):
        super(ColorNetwork, self).__init__()
        # Input dim: 3 (pos) + 2 (view) + 12 (feat) = 17
        # Hidden dim: 128
        # Output dim: 3 (color)
        self.fc1 = nn.Linear(17, 128)
        self.fc2 = nn.Linear(128, 128)
        self.fc3 = nn.Linear(128, 128)
        self.fc4 = nn.Linear(128, 3)
        self.relu = nn.ReLU()
        self.sigmoid = nn.Sigmoid()

    def forward(self, pos, view_dir, feat):
        # Concatenate inputs: [pos (3), view_dir (2), feat (12)]
        x = torch.cat([pos, view_dir, feat], dim=-1)
        
        # Three hidden layers with ReLU
        h1 = self.relu(self.fc1(x))
        h2 = self.relu(self.fc2(h1))
        h3 = self.relu(self.fc3(h2))
        
        # Output layer with Sigmoid
        color = self.sigmoid(self.fc4(h3))
        return color
```

## Subtopics

### [Neural Textures Architecture](9534-neural-textures-architecture.md)
Neural Textures is a view synthesis method that extends traditional 2D RGB texture maps into high-dimensional Neural Feature Maps associated with an explicit 3D mesh model. It solves the problem of view-dependent appearance rendering by decoupling geometry (stored as a mesh) from appearance (stored as learnable features), allowing for real-time rendering and efficient training compared to implicit neural fields. It differs from its parent, the Color Network Architecture, by explicitly utilizing UV-parameterized feature maps on a mesh rather than querying a continuous MLP at arbitrary 3D coordinates.

### [Decomposition Network](12532-decomposition-network.md)
The Decomposition Network is a multi-layer perceptron (MLP) architecture used within neural fields to simultaneously reconstruct geometry and decompose material properties (diffuse color, roughness) from multi-view images. It distinguishes itself from standard NeRF architectures by explicitly optimizing for physically-based rendering (PBR) parameters rather than just color, employing a specialized loss schedule to handle the instability of normal estimation and BRDF fitting during early training stages.

### EC-deep
Alternative model EC-deep for comparison against RenderNet [S2, unofficial]. Input Processing: Lighting and pose parameters fed through fully-connected layers to map each to 512-dimensional vector [S2, unofficial]. Input Concatenation: Two 512-dim vectors concatenated to shape latent vector [S2, unofficial]. FC Layers: Concatenated vector fed through 2 fully-connected layers to map to 1024-dim vector, then another FC layer to (8*8*512)-dim vector [S2, unofficial]. Reshape: Output reshaped to tensor of size 8x8x512 [S2, unofficial]. Decoder: Replace each non-strided convolution in EC with two 2D residual blocks [S2, unofficial].

### Intrinsic Color Decomposition Network
A color network design that disentangles color-shape ambiguities by predicting separate albedo (view-invariant) and shading (view-dependent) components, summed to produce the final color [S3]. The final color C is computed as $C = \Phi(C_a + C_s)$, where $\Phi$ is the Sigmoid function, $C_a$ is the albedo RGB vector, and $C_s$ is the shading scalar [S3]. Albedo branch: Input is point locations and SDF MLP features; Output is $C_a \in \mathbb{R}^3$ (RGB, view-invariant). Shading branch: Input is point locations, SDF MLP features, reflection direction, and view direction; Output is $C_s \in \mathbb{R}$ (gray, view-dependent). Both branches use two hidden layers [S3]. Flat surfaces tend to be carved away when using this decomposition, potentially due to instability in the dot product computation for reflection [S3].

### Per-Image Latent Embedding
An addition to the color network using per-image latent embeddings to model exposure variation across frames in datasets with wild capture conditions [S3].

## Operating characteristics
Not given in the sources.

## Background
The Color Network Architecture, specifically referred to as $F_{\theta c2}$, was introduced in the context of neural deformable voxel grids for fast optimization of dynamic view synthesis [S1]. It is designed to be lightweight, using three fully connected layers with a width of 128 [S1]. The architecture is part of a larger system that includes a deformation network $F_{\theta d1}$ [S1].

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S2] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26 (unofficial)
[S3] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
