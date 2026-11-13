# Color Network Architecture


## Overview
The Color Network Architecture is a Multi-Layer Perceptron (MLP) module within neural rendering pipelines that predicts RGB color from input features, view directions, and positions [S1]. It solves the problem of view-dependent appearance synthesis by mapping latent geometric and radiance features to final pixel values [S1]. It differs from its parent, [MLPRender_Fea](2414-mlprender-fea.md), by explicitly defining a three-layer fully connected structure with specific input dimensionality for position, view direction, and feature vectors, using ReLU activations for hidden layers and sigmoid for the output [S1].

## How it works
**Inputs and Outputs:**
*   **Position ($p$):** 3-dimensional vector (x, y, z) [S1].
*   **View Direction ($d$):** 2-dimensional vector (azimuth, elevation) [S1].
*   **Feature ($f$):** 12-dimensional vector representing latent radiance/geometry information [S1].
*   **Output ($c$):** 3-dimensional vector representing RGB color values [S1].

**Data Structures:**
The network consists of three fully connected (FC) layers. Each hidden layer has a width of 128 neurons [S1].

**Parameters:**
*   Hidden layer width: 128 [S1].
*   Activation function (hidden): Rectified Linear Unit (ReLU) [S1].
*   Activation function (output): Sigmoid [S1].

**Steps:**
1.  **Concatenation:** Concatenate the position vector $p \in \mathbb{R}^3$, view direction vector $d \in \mathbb{R}^2$, and feature vector $f \in \mathbb{R}^{12}$ into a single input vector $x \in \mathbb{R}^{17}$ [S1].
    $$ x = [p, d, f] $$
2.  **First Hidden Layer:** Apply a fully connected layer with 128 output units to $x$, followed by ReLU activation [S1].
    $$ h_1 = \text{ReLU}(W_1 x + b_1) $$
    where $W_1 \in \mathbb{R}^{128 \times 17}$ and $b_1 \in \mathbb{R}^{128}$.
3.  **Second Hidden Layer:** Apply a fully connected layer with 128 output units to $h_1$, followed by ReLU activation [S1].
    $$ h_2 = \text{ReLU}(W_2 h_1 + b_2) $$
    where $W_2 \in \mathbb{R}^{128 \times 128}$ and $b_2 \in \mathbb{R}^{128}$.
4.  **Third Hidden Layer:** Apply a fully connected layer with 128 output units to $h_2$, followed by ReLU activation [S1].
    $$ h_3 = \text{ReLU}(W_3 h_2 + b_3) $$
    where $W_3 \in \mathbb{R}^{128 \times 128}$ and $b_3 \in \mathbb{R}^{128}$.
5.  **Output Layer:** Apply a fully connected layer with 3 output units to $h_3$, followed by Sigmoid activation to constrain color values to the range $(0, 1)$ [S1].
    $$ c = \sigma(W_4 h_3 + b_4) $$
    where $W_4 \in \mathbb{R}^{3 \times 128}$ and $b_4 \in \mathbb{R}^{3}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x):
    return 1.0 / (1.0 + math.exp(-x))

def relu(x):
    return max(0.0, x)

class ColorNetwork:
    def __init__(self):
        # Initialize weights and biases for 3 hidden layers (width 128) and output layer (width 3)
        # Input dim: 17 (3 pos + 2 view + 12 feature)
        # Layer 1: 17 -> 128
        self.W1 = [[0.0 for _ in range(17)] for _ in range(128)]
        self.b1 = [0.0 for _ in range(128)]
        
        # Layer 2: 128 -> 128
        self.W2 = [[0.0 for _ in range(128)] for _ in range(128)]
        self.b2 = [0.0 for _ in range(128)]
        
        # Layer 3: 128 -> 128
        self.W3 = [[0.0 for _ in range(128)] for _ in range(128)]
        self.b3 = [0.0 for _ in range(128)]
        
        # Output Layer: 128 -> 3
        self.W4 = [[0.0 for _ in range(128)] for _ in range(3)]
        self.b4 = [0.0 for _ in range(3)]

    def forward(self, position, view_dir, feature):
        # 1. Concatenate inputs
        # position: 3, view_dir: 2, feature: 12 -> total 17
        x = position + view_dir + feature
        
        # 2. First hidden layer
        h1 = [0.0] * 128
        for i in range(128):
            val = self.b1[i]
            for j in range(17):
                val += self.W1[i][j] * x[j]
            h1[i] = relu(val)
            
        # 3. Second hidden layer
        h2 = [0.0] * 128
        for i in range(128):
            val = self.b2[i]
            for j in range(128):
                val += self.W2[i][j] * h1[j]
            h2[i] = relu(val)
            
        # 4. Third hidden layer
        h3 = [0.0] * 128
        for i in range(128):
            val = self.b3[i]
            for j in range(128):
                val += self.W3[i][j] * h2[j]
            h3[i] = relu(val)
            
        # 5. Output layer with sigmoid
        color = [0.0] * 3
        for i in range(3):
            val = self.b4[i]
            for j in range(128):
                val += self.W4[i][j] * h3[j]
            color[i] = sigmoid(val)
            
        return color
```

**Edge Cases and Pitfalls:**
*   **Output Range:** The sigmoid activation ensures color values are strictly between 0 and 1. If linear RGB values outside this range are required, the output activation must be modified [S1].
*   **Input Dimensionality:** The architecture assumes fixed input dimensions (3, 2, 12). Deviations require adjusting the first weight matrix $W_1$ [S1].

## Subtopics

### [Neural Textures Architecture](9534-neural-textures-architecture.md)
Neural Textures is a view synthesis method that extends traditional 2D RGB texture maps into high-dimensional Neural Feature Maps associated with an explicit 3D mesh model. It solves the problem of view-dependent appearance rendering by decoupling geometry (stored as a mesh) from appearance (stored as learnable features), allowing for real-time rendering and efficient training compared to implicit neural fields. It differs from its parent, the Color Network Architecture, by explicitly utilizing UV-parameterized feature maps on a mesh rather than querying a continuous MLP at arbitrary 3D coordinates.

### [Intrinsic Color Decomposition Network](9940-intrinsic-color-decomposition-network.md)
The Intrinsic Color Decomposition Network disentangles color-shape ambiguities by predicting separate albedo (view-invariant) and shading (view-dependent) components, which are summed to produce the final color. Unlike standard NeRF architectures that use a single view-dependent MLP for color, this method improves extrapolation performance in novel view synthesis by allowing the view-independent factor to provide reasonable predictions when facing large viewing direction shifts.

### [Decomposition Network](12532-decomposition-network.md)
The Decomposition Network is a multi-layer perceptron (MLP) architecture used within neural fields to simultaneously reconstruct geometry and decompose material properties (diffuse color, roughness) from multi-view images. It distinguishes itself from standard NeRF architectures by explicitly optimizing for physically-based rendering (PBR) parameters rather than just color, employing a specialized loss schedule to handle the instability of normal estimation and BRDF fitting during early training stages.

### [Squeeze-and-Excitation Module](14050-squeeze-and-excitation-module.md)
The Squeeze-and-Excitation (SE) Module is a channel attention mechanism used to dynamically recalibrate feature responses by explicitly modeling interdependencies between channels. In the context of Mixer-NeRF, it is placed between the sampling layer and the inference layer to filter out redundant information and highlight important features, thereby improving detail restoration and color accuracy in complex, high-texture scenes. It distinguishes itself from sibling architectures like Neural Textures or Decomposition Networks by focusing specifically on channel-wise feature weighting rather than spatial decomposition or texture synthesis.

### EC-deep
EC-deep is an alternative model for comparison against RenderNet [S2, unofficial]. Lighting and pose parameters are fed through fully-connected layers to map each to a 512-dimensional vector [S2, unofficial]. These two vectors are concatenated to form a latent vector [S2, unofficial]. This vector is fed through 2 fully-connected layers to map to a 1024-dimensional vector, then another FC layer to an $(8 \times 8 \times 512)$-dimensional vector [S2, unofficial]. The output is reshaped to a tensor of size $8 \times 8 \times 512$ before being fed to the decoder [S2, unofficial]. The decoder replaces each non-strided convolution in EC with two 2D residual blocks [S2, unofficial].

### Per-Image Latent Embedding
An addition to the color network using per-image latent embeddings to model exposure variation across frames in datasets with wild capture conditions [S3]. This modification helps reduce floating artifacts caused by exposure differences [S3].

## Operating characteristics
**Time and Memory Complexity:**
The network has 3 hidden layers of width 128. The number of parameters is approximately $17 \times 128 + 128 + 128 \times 128 + 128 + 128 \times 128 + 128 + 128 \times 3 + 3 \approx 33,000$ parameters [S1]. Inference is fast as it involves only matrix multiplications and element-wise activations [S1].

**Measured Performance:**
Specific PSNR or training times for this exact architecture are not provided in the sources, though it is part of a system achieving competitive results on dynamic view synthesis tasks [S1].

**Strengths and Limitations:**
*   **Strengths:** Lightweight, efficient for dynamic scenes when combined with deformation networks [S1].
*   **Limitations:** The use of sigmoid activation limits output to $(0, 1)$, which may not support HDR or linear color spaces without post-processing [S1]. The fixed input dimensions limit flexibility [S1].

**When to use:**
Use when computational efficiency is critical and the scene can be represented with the specified input feature dimensions [S1].

**When not to use:**
Avoid if HDR output is required or if the input feature dimensionality differs significantly from 12 [S1].

## Background
The Color Network Architecture, specifically denoted as $F_{\theta c2}$, originates from the work on Neural Deformable Voxel Grids for Fast Optimization of Dynamic View Synthesis [S1]. It was designed to work in conjunction with a deformation network to handle dynamic scenes [S1].

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S2] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26 (unofficial)
[S3] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
