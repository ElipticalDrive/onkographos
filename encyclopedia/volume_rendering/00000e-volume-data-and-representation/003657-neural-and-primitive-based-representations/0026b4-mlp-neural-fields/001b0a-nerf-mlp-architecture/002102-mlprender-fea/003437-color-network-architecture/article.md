# Color Network Architecture

Also called: Canonical Color Network, NeRF MLP

## Overview
The Color Network Architecture is a Multi-Layer Perceptron (MLP) module within neural radiance fields that predicts the RGB color of a scene point conditioned on its 3D position, viewing direction, and optional learned features [S2]. It solves the problem of view-dependent appearance modeling by mapping continuous spatial coordinates to radiance values [S1]. It differs from its parent, [MLPRender_Fea](2414-mlprender-fea.md), by explicitly defining the standard MLP structure (linear layers with ReLU activations) used to model the radiance field, often initialized with specific weight scaling to ensure stable training [S3, unofficial]. Unlike sibling architectures that may decouple geometry and appearance (e.g., [Neural Textures Architecture](9534-neural-textures-architecture.md)) or decompose intrinsic properties (e.g., [Intrinsic Color Decomposition Network](9940-intrinsic-color-decomposition-network.md)), this architecture typically processes position, view direction, and features jointly through a single deep network to predict color directly [S1].

## How it works
The method consists of an MLP that takes concatenated input vectors and produces a color output [S2].

**Inputs and Outputs:**
*   **Input:** A concatenated vector $x = [p, d, f]$ containing position $p$, viewing direction $d$, and optional learned feature vector $f$ [S2]. Input dimensionality depends on the positional encoding scheme and implementation: e.g., position (33), view direction (27), and feature (12) [S2]; or canonicalized query points, normals, viewing directions, SDF feature, and per-frame latent code [S1].
*   **Output:** RGB color vector $c$ (typically 3 channels) [S2], or density and spherical harmonic coefficients (e.g., 64 channels) [S3, unofficial].

**Data Structures:**
The network is defined as a list of layer dictionaries [S3, unofficial]. Each layer $i$ contains:
*   Weight matrix $W_i \in \mathbb{R}^{d_{in} \times d_{out}}$ [S3, unofficial].
*   Bias vector $b_i \in \mathbb{R}^{d_{out}}$ [S3, unofficial].

**Initialization:**
1.  Initialize weights from a normal distribution $\mathcal{N}(0, 0.1^2)$ [S3, unofficial].
2.  Initialize all bias vectors to zeros [S3, unofficial].
3.  First layer weights shape: $(d_{in}, h)$ [S3, unofficial].
4.  Intermediate hidden layer weights shape: $(h, h)$ [S3, unofficial].
5.  Output layer weights shape: $(h, d_{out})$ [S3, unofficial].

**Parameters:**
*   $d_{in}$: Input dimensionality. Example: 63 (from positional encoding of 3D position) [S3, unofficial].
*   $h$: Hidden dimension. Example: 256 [S3, unofficial]. In [S2], width is 128. In [S1], width is 256.
*   $L$: Number of layers. Example: 10 [S3, unofficial]. In [S2], there are 3 fully connected layers. In [S1], there are 4 hidden layers.
*   $d_{out}$: Output dimensionality. Example: 64 (representing density and spherical harmonic coefficients) [S3, unofficial]. In [S2], output is 3 (RGB).

**Algorithm:**
1.  Concatenate input vectors to form $x = [p, d, f]$ [S2].
2.  Initialize hidden state $h_{curr} = x$ [S3, unofficial].
3.  Iterate through layers $i = 0$ to $L-2$:
    *   Linear transformation: $z = h_{curr} W_i + b_i$ [S3, unofficial].
    *   Activation: $h_{curr} = \text{ReLU}(z)$ [S3, unofficial].
4.  Output layer ($i = L-1$):
    *   Linear transformation: $z_{out} = h_{curr} W_{L-1} + b_{L-1}$ [S3, unofficial].
    *   Apply sigmoid activation for color output: $c = \sigma(z_{out})$ [S2]. For density/SH outputs, apply no activation [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_color_network(input_dim, hidden_dim, num_layers, output_dim):
    layers = []
    # First layer
    W0 = np.random.normal(0, 0.1, (input_dim, hidden_dim))
    b0 = np.zeros(hidden_dim)
    layers.append({'W': W0, 'b': b0})
    # Intermediate hidden layers
    for i in range(1, num_layers - 1):
        Wi = np.random.normal(0, 0.1, (hidden_dim, hidden_dim))
        bi = np.zeros(hidden_dim)
        layers.append({'W': Wi, 'b': bi})
    # Output layer
    W_last = np.random.normal(0, 0.1, (hidden_dim, output_dim))
    b_last = np.zeros(output_dim)
    layers.append({'W': W_last, 'b': b_last})
    return layers

def forward_color_network(layers, x, apply_sigmoid=False):
    h = x
    num_layers = len(layers)
    # Hidden layers with ReLU
    for i in range(num_layers - 1):
        W = layers[i]['W']
        b = layers[i]['b']
        z = np.dot(h, W) + b
        h = np.maximum(0, z)
    # Output layer (no ReLU)
    W_last = layers[-1]['W']
    b_last = layers[-1]['b']
    out = np.dot(h, W_last) + b_last
    if apply_sigmoid:
        out = 1.0 / (1.0 + np.exp(-out))
    return out
```

**Edge Cases and Pitfalls:**
*   **Precision:** Using float32 is standard. Initialization with 0.1 scale is critical; larger scales can lead to vanishing or exploding gradients in deep networks [S3, unofficial].
*   **Boundaries:** The sigmoid activation on the output layer ensures color values are within $[0, 1]$ [S2]. Without it, values may exceed valid color ranges.
*   **Input Dimensionality:** The input dimension depends heavily on the positional encoding scheme [S3, unofficial]. If positional encoding is used, the input dimension increases significantly (e.g., 3D position with 10 bands of sine/cosine becomes 63 dimensions) [S3, unofficial].

## Subtopics

### [Neural Textures Architecture](9534-neural-textures-architecture.md)
Neural Textures extends traditional 2D RGB texture maps into high-dimensional Neural Feature Maps associated with an explicit 3D mesh model to achieve view synthesis. It differs from its siblings by mapping features to a simple 3D proxy surface and sampling them via the standard graphics pipeline, producing standard physically based textures compatible with traditional renderers rather than querying a continuous MLP at arbitrary 3D coordinates. Choose it when real-time rendering and explicit geometry proxies are required.

### [Intrinsic Color Decomposition Network](9940-intrinsic-color-decomposition-network.md)
The Intrinsic Color Decomposition Network disentangles color-shape ambiguities by predicting separate albedo (view-invariant) and shading (view-dependent) components, which are summed to produce the final color. It differs from siblings by improving extrapolation performance in novel view synthesis, allowing the view-independent factor to provide reasonable predictions when facing large viewing direction shifts. Choose it when robust novel-view extrapolation and material disentanglement are prioritized over raw PSNR.

### [Decomposition Network](12532-decomposition-network.md)
The Decomposition Network is an MLP-based architecture that optimizes material parameters (diffuse color, roughness) alongside color reconstruction from multi-view images. It differs from siblings by explicitly targeting physically-based rendering parameters and employing a specialized fading loss schedule to stabilize normal estimation and BRDF fitting during early training. Choose it when physically-based material recovery is the primary objective.

### [Squeeze-and-Excitation Module](14050-squeeze-and-excitation-module.md)
The Squeeze-and-Excitation (SE) Module is a channel attention mechanism used to dynamically recalibrate feature responses by explicitly modeling interdependencies between channels. In Mixer-NeRF, it is placed between the sampling layer and the inference layer to filter out redundant information and highlight important features, thereby improving detail restoration and color accuracy in complex, high-texture scenes. It differs from siblings by focusing specifically on channel-wise feature weighting rather than spatial decomposition or texture synthesis. Choose it when high-frequency detail recovery in textured regions is critical.

### [CrfModel](14213-crfmodel.md)
The CrfModel is a color network architecture component used in neural radiance fields that applies independent, learned non-linear transformations to each RGB channel of the input color vector. It differs from siblings by processing channels separately rather than jointly, allowing for channel-specific response functions. Choose it when channel-specific color correction or tone mapping is required.

### EC-deep
EC-deep is an alternative model used for comparison against RenderNet [S4, unofficial]. It processes lighting and pose parameters by feeding each through a fully-connected layer to map them to 512-dimensional vectors [S4, unofficial]. These two vectors are concatenated to form a latent vector, which is then fed through 2 fully-connected layers to map to a 1024-dimensional vector, and another FC layer to an $(8 \cdot 8 \cdot 512)$-dimensional vector [S4, unofficial]. The output is reshaped into a tensor of size $8 \times 8 \times 512$ before being fed to the decoder [S4, unofficial]. The decoder replaces each non-strided convolution in the baseline EC model with two 2D residual blocks [S4, unofficial].

### Per-Image Latent Embedding
Per-Image Latent Embedding is an addition to the color network that uses per-image latent embeddings to model exposure variation across frames in datasets with wild capture conditions [S5]. This modification helps reduce floating objects and improves qualitative results by accounting for lighting and exposure differences between training images [S5].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Simple to implement, flexible input dimensions, and effective at modeling complex view-dependent effects when combined with positional encoding [S3, unofficial].
*   **Limitations:** Computationally intensive for real-time rendering without acceleration structures. Sensitive to initialization scale [S3, unofficial].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The architecture is described as the Canonical Color Network in [S1], as $F_{\theta_{c2}}$ in [S2], and as a standard NeRF MLP structure in [S3, unofficial]. The specific initialization scheme (0.1 normal distribution) and structure (linear layers with ReLU) are standard practices in deep learning for such networks [S3, unofficial]. Variants like EC-deep have been proposed for differentiable rendering from 3D shapes [S4, unofficial], and per-image embeddings have been added to handle exposure variations in wild capture conditions [S5].

## Sources
[S1] ARAH: Animatable Volume Rendering of Articulated Human SDFs, https://doi.org/10.1007/978-3-031-19824-3_1, fetched 2026-10-26
[S2] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S3] mdmubarak/OctaNeRF, https://github.com/mdmubarak/OctaNeRF, fetched 2026-10-27 (unofficial)
[S4] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26 (unofficial)
[S5] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
