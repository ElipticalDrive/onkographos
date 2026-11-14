# Color Network Architecture

Also called: NeRF MLP

## Overview
The Color Network Architecture is a Multi-Layer Perceptron (MLP) module within neural radiance fields that predicts the RGB color of a scene point conditioned on its 3D position, viewing direction, and optional learned features. It solves the problem of view-dependent appearance modeling by mapping continuous spatial coordinates to radiance values. It differs from its parent, [MLPRender_Fea](2414-mlprender-fea.md), by explicitly defining the standard MLP structure (linear layers with ReLU activations) used to model the radiance field, often initialized with specific weight scaling to ensure stable training [S2, unofficial]. Unlike sibling architectures that may decouple geometry and appearance (e.g., [Neural Textures Architecture](9534-neural-textures-architecture.md)) or decompose intrinsic properties (e.g., [Intrinsic Color Decomposition Network](9940-intrinsic-color-decomposition-network.md)), this architecture typically processes position, view direction, and features jointly through a single deep network to predict color directly [S1].

## How it works
The method consists of an MLP that takes concatenated input vectors and produces a color output.

**Inputs and Outputs:**
*   **Input:** A concatenated vector containing:
    *   Position ($p$): 3D coordinates.
    *   View Direction ($d$): Normalized viewing vector.
    *   Feature ($f$): Optional learned feature vector (e.g., from a deformation network or voxel grid).
*   **Output:** RGB color vector ($c$), typically in the range $[0, 1]$.

**Data Structures:**
The network is defined by a list of layer parameters. Each layer $i$ consists of:
*   Weight matrix $W_i \in \mathbb{R}^{d_{in} \times d_{out}}$.
*   Bias vector $b_i \in \mathbb{R}^{d_{out}}$.

**Initialization:**
1.  Generate random keys for initialization.
2.  For the first layer (input to first hidden):
    *   $W_0 \sim \mathcal{N}(0, 0.1^2)$ with shape $(d_{in}, h)$.
    *   $b_0 = 0$ with shape $(h,)$.
3.  For intermediate hidden layers $i = 1 \dots L-2$:
    *   $W_i \sim \mathcal{N}(0, 0.1^2)$ with shape $(h, h)$.
    *   $b_i = 0$ with shape $(h,)$.
4.  For the output layer $L-1$:
    *   $W_{L-1} \sim \mathcal{N}(0, 0.1^2)$ with shape $(h, d_{out})$.
    *   $b_{L-1} = 0$ with shape $(d_{out},)$.
    *   [G] Note: Some implementations scale weights by $1/\sqrt{d_{in}}$ for stability, but the source specifies 0.1 normal distribution [S2, unofficial].

**Parameters:**
*   $d_{in}$: Input dimensionality. Example: 63 (from positional encoding of position) [S2, unofficial]. In specific implementations like [S1], inputs are separate: position (3), view direction (2), feature (12), concatenated to 17.
*   $h$: Hidden dimension. Example: 256 [S2, unofficial]. In [S1], width is 128.
*   $L$: Number of layers. Example: 10 [S2, unofficial]. In [S1], there are 3 fully connected layers.
*   $d_{out}$: Output dimensionality. Example: 64 (representing density and spherical harmonic coefficients) [S2, unofficial]. In [S1], output is 3 (RGB).

**Algorithm:**
1.  Concatenate input vectors: $x = [p, d, f]$.
2.  Initialize hidden state $h_{curr} = x$.
3.  Iterate through layers $i = 0$ to $L-2$:
    *   Linear transformation: $z = h_{curr} W_i + b_i$.
    *   Activation: $h_{curr} = \text{ReLU}(z)$.
4.  Output layer ($i = L-1$):
    *   Linear transformation: $z_{out} = h_{curr} W_{L-1} + b_{L-1}$.
    *   [G] Apply sigmoid activation if output is RGB: $c = \sigma(z_{out})$. If output includes density/SH, no activation or different activation may be used [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_color_network(input_dim, hidden_dim, num_layers, output_dim):
    """
    Initializes the weights and biases for the Color Network Architecture.
    """
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

def forward_color_network(layers, x):
    """
    Forward pass through the Color Network.
    x: input vector (concatenated position, view direction, features)
    """
    h = x
    num_layers = len(layers)
    
    # Hidden layers with ReLU
    for i in range(num_layers - 1):
        W = layers[i]['W']
        b = layers[i]['b']
        z = np.dot(h, W) + b
        h = np.maximum(0, z) # ReLU
        
    # Output layer (no ReLU)
    W_last = layers[-1]['W']
    b_last = layers[-1]['b']
    out = np.dot(h, W_last) + b_last
    
    return out

# Example usage for [S1] configuration
# Input: pos(3) + view(2) + feat(12) = 17
# Layers: 3 FC layers, width 128
# Output: 3 (RGB)
layers_s1 = initialize_color_network(17, 128, 3, 3)
# Note: [S1] applies sigmoid to output for color range (0,1)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Using float32 is standard. Initialization with 0.1 scale is critical; larger scales can lead to vanishing/exploding gradients in deep networks [S2, unofficial].
*   **Boundaries:** The sigmoid activation on the output layer ensures color values are within $[0, 1]$. Without it, values may exceed valid color ranges [S1].
*   **Input Dimensionality:** The input dimension depends heavily on the positional encoding scheme. If positional encoding is used, the input dimension increases significantly (e.g., 3D position with 10 bands of sine/cosine becomes 63 dimensions) [S2, unofficial].

## Subtopics

### [Neural Textures Architecture](9534-neural-textures-architecture.md)
Neural Textures is a view synthesis method that extends traditional 2D RGB texture maps into high-dimensional Neural Feature Maps associated with an explicit 3D mesh model. It solves the problem of view-dependent appearance rendering by decoupling geometry (stored as a mesh) from appearance (stored as learnable features), allowing for real-time rendering and efficient training compared to implicit neural fields. It differs from its parent, the Color Network Architecture, by explicitly utilizing UV-parameterized feature maps on a mesh rather than querying a continuous MLP at arbitrary 3D coordinates.

### [Intrinsic Color Decomposition Network](9940-intrinsic-color-decomposition-network.md)
The Intrinsic Color Decomposition Network disentangles color-shape ambiguities by predicting separate albedo (view-invariant) and shading (view-dependent) components, which are summed to produce the final color. Unlike standard NeRF architectures that use a single view-dependent MLP for color, this method improves extrapolation performance in novel view synthesis by allowing the view-independent factor to provide reasonable predictions when facing large viewing direction shifts.

### [Decomposition Network](12532-decomposition-network.md)
The Decomposition Network is a multi-layer perceptron (MLP) architecture used within neural fields to simultaneously reconstruct geometry and decompose material properties (diffuse color, roughness) from multi-view images. It distinguishes itself from standard NeRF architectures by explicitly optimizing for physically-based rendering (PBR) parameters rather than just color, employing a specialized loss schedule to handle the instability of normal estimation and BRDF fitting during early training stages.

### [Squeeze-and-Excitation Module](14050-squeeze-and-excitation-module.md)
The Squeeze-and-Excitation (SE) Module is a channel attention mechanism used to dynamically recalibrate feature responses by explicitly modeling interdependencies between channels. In the context of Mixer-NeRF, it is placed between the sampling layer and the inference layer to filter out redundant information and highlight important features, thereby improving detail restoration and color accuracy in complex, high-texture scenes. It distinguishes itself from sibling architectures like Neural Textures or Decomposition Networks by focusing specifically on channel-wise feature weighting rather than spatial decomposition or texture synthesis.

### [CrfModel](14213-crfmodel.md)
The CrfModel is a color network architecture component used in neural radiance fields that applies independent, learned non-linear transformations to each RGB channel of the input color vector. It differs from sibling architectures like the Neural Textures Architecture or Intrinsic Color Decomposition Network by processing channels separately rather than jointly, allowing for channel-specific response functions.

#### EC-deep
EC-deep is an alternative model used for comparison against RenderNet [S3, unofficial]. It processes lighting and pose parameters by feeding each through a fully-connected layer to map them to 512-dimensional vectors. These two vectors are concatenated to form a latent vector, which is then fed through 2 fully-connected layers to map to a 1024-dimensional vector, and another FC layer to an $(8 \cdot 8 \cdot 512)$-dimensional vector. The output is reshaped into a tensor of size $8 \times 8 \times 512$ before being fed to the decoder. The decoder replaces each non-strided convolution in the baseline EC model with two 2D residual blocks [S3, unofficial].

#### Per-Image Latent Embedding
Per-Image Latent Embedding is an addition to the color network that uses per-image latent embeddings to model exposure variation across frames in datasets with wild capture conditions [S4]. This modification helps reduce floating objects and improves qualitative results by accounting for lighting and exposure differences between training images [S4].

## Operating characteristics
*   **Time Complexity:** $O(L \cdot h^2)$ per query, where $L$ is the number of layers and $h$ is the hidden dimension. For typical values ($L=10, h=256$), this is computationally expensive for dense sampling but manageable for sparse sampling [S2, unofficial].
*   **Memory Complexity:** $O(L \cdot h^2)$ to store weights. For $L=10, h=256$, this is approximately $10 \times 256^2 \times 4$ bytes $\approx 2$ MB, which is small [S2, unofficial].
*   **Performance:** The architecture is capable of high-fidelity view synthesis but requires significant training time due to the depth of the network and the need for positional encoding [S1][S2, unofficial].
*   **Strengths:** Simple to implement, flexible input dimensions, and effective at modeling complex view-dependent effects when combined with positional encoding [S1][S2, unofficial].
*   **Limitations:** Computationally intensive for real-time rendering without acceleration structures (e.g., octrees, hash grids) [S2, unofficial]. Sensitive to initialization scale [S2, unofficial].
*   **When to use:** When high-quality novel view synthesis is required and computational resources allow for offline training.
*   **When not to use:** When real-time rendering on low-end hardware is required without acceleration structures.

## Background
The Color Network Architecture is a fundamental component of Neural Radiance Fields (NeRF) and related neural rendering methods. It was introduced in the context of modeling radiance fields using MLPs [S2, unofficial]. The specific initialization scheme (0.1 normal distribution) and structure (linear layers with ReLU) are standard practices in deep learning for such networks [S2, unofficial]. Variants like EC-deep have been proposed for differentiable rendering from 3D shapes [S3, unofficial], and per-image embeddings have been added to handle exposure variations in wild capture conditions [S4].

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S2] mdmubarak/OctaNeRF, https://github.com/mdmubarak/OctaNeRF, fetched 2026-10-27 (unofficial)
[S3] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26 (unofficial)
[S4] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
