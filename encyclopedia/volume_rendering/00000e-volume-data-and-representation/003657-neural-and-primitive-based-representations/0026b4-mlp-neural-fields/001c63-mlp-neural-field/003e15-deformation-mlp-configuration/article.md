# Deformation MLP Configuration

Also called: Conditional Batch Normalization, Deformation MLP Setup, Feature-wise Linear Modulation (FiLM), Feature-wise transformations, MLP setup

## Overview
The Deformation MLP Configuration defines the architecture, initialization, and conditioning mechanisms for a Multi-Layer Perceptron (MLP) used to predict spatial or temporal deformations in neural volume representations [S1]. It solves the problem of mapping canonical coordinates to deformed coordinates by providing a structured way to inject instance-specific or time-specific information into the network's feature space via feature-wise linear modulation [S2, unofficial]. It is distinguished from sibling methods like [MLP with Input Skips](1643-mlp-with-input-skips.md) by its explicit use of feature-wise modulation to handle conditional inputs, and from [Residual Block with Shortcut (ReLU MLP Extension)](10016-residual-block-with-shortcut-relu-mlp-extension.md) by its focus on deformation dynamics rather than static residual learning [S1][S2, unofficial].

## How it works
### Inputs and Outputs
* **Inputs**: Spatial coordinates $\mathbf{x}$, time $t$ (optional), and conditioning latent vectors $\mathbf{z}$ (optional) [S1][S2, unofficial].
* **Outputs**: Deformation vector $\delta\mathbf{x}$ (delta position) or deformed coordinates $\mathbf{x}' = \mathbf{x} + \delta\mathbf{x}$ [S1].

### Data Structures and Layout
The standard deformation MLP layout, as defined in [S1], consists of:
* **Depth**: 8 layers.
* **Width**: 256 neurons per layer.
* **Input Encoding**: Positional encoding is applied to spatial coordinates $\mathbf{x}$ with frequency $L_x = 10$ and time $t$ with frequency $L_t = 10$ [S1].
* **Conditioning**: Conditioning signals $\mathbf{z}$ are injected via Feature-wise Linear Modulation (FiLM) or Conditional Batch Normalization (CBN) [S2, unofficial]. For each layer $i$, scale $\boldsymbol{\gamma}_i(\mathbf{z})$ and bias $\boldsymbol{\beta}_i(\mathbf{z})$ vectors are predicted by small auxiliary networks and applied to the layer's output [S2, unofficial].

### Numbered Steps
1. **Encode Input**: Apply positional encoding to spatial coordinates $\mathbf{x}$ using frequency $L_x = 10$ and to time $t$ using $L_t = 10$ [S1].
2. **Initialize Weights**: Initialize weights for all hidden layers using Xavier normal initialization [S1]. Initialize all biases to zero [S1].
3. **Initialize Final Layer**: Initialize the weights of the final linear layer (mapping 256 features to the delta position $\delta\mathbf{x}$) from a normal distribution with standard deviation $\sigma = 10^{-5}$ [S1].
4. **Forward Pass**: For each layer $i$, compute the linear transformation $L_i(\boldsymbol{x})$, then apply FiLM modulation using the predicted scale $\boldsymbol{\gamma}_i(\mathbf{z})$ and bias $\boldsymbol{\beta}_i(\mathbf{z})$ [S2, unofficial].
5. **Activation**: Apply a non-linear activation function (e.g., ReLU) after each layer [G].
6. **Output**: Return the final displacement vector $\delta\mathbf{x}$ [S1].

### Formulas
The feature-wise linear modulation applied at layer $i$ is defined as:
$$ \hat{\boldsymbol{x}}_i = L_i(\boldsymbol{x}) \odot \boldsymbol{\gamma}_i(\mathbf{z}) + \boldsymbol{\beta}_i(\mathbf{z}) $$
where:
* $L_i(\boldsymbol{x})$ is the linear transformation output of layer $i$ [S2, unofficial].
* $\odot$ denotes element-wise multiplication [S2, unofficial].
* $\boldsymbol{\gamma}_i(\mathbf{z})$ and $\boldsymbol{\beta}_i(\mathbf{z})$ are the scale and bias vectors predicted by auxiliary networks conditioned on $\mathbf{z}$ [S2, unofficial].
* The size of $\boldsymbol{\gamma}_i$ and $\boldsymbol{\beta}_i$ matches the size of the layer's output [S2, unofficial].

### Parameters
* **Layer Output Size**: The size of the scale and bias vectors in FiLM/CBN matches the size of the layer's output [S2, unofficial].
* **Optimizer**: Adam optimizer with $\beta = (0.9, 0.999)$ and $\epsilon = 10^{-15}$ [S1].
* **Learning Rate**: The MLP learning rate $\gamma_{MLP}$ typically starts at $0.0008$ and decays exponentially (e.g., to $1.6 \times 10^{-6}$ over 60K iterations) [S1].

### Edge Cases and Pitfalls
* **Precision**: The final layer's small weight initialization ($\sigma = 10^{-5}$) prevents large initial deformations that could destabilize optimization [S1].
* **Boundaries**: Positional encoding frequencies $L_x=10$ and $L_t=10$ are fixed; exceeding these frequencies without adjusting the encoding formula may cause aliasing [S1].
* **Degenerate Input**: If conditioning latent vectors $\mathbf{z}$ are omitted, the FiLM modulation defaults to identity scaling and zero bias, reducing the network to a standard MLP [G].

### Pseudocode
The following Python-like pseudocode implements the standard Deformation MLP configuration described in [S1]. It assumes standard vector operations.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def linear_layer(x, weights, bias):
    out = [0.0] * len(bias)
    for i in range(len(bias)):
        s = bias[i]
        for j in range(len(x)):
            s += x[j] * weights[i * len(x) + j]
        out[i] = s
    return out

def positional_encoding(x, L):
    # Exact positional encoding formula not provided in sources; using standard sinusoidal encoding
    encoded = []
    for val in x:
        for k in range(L):
            freq = math.pi * (2 ** k)
            encoded.extend([math.sin(val * freq), math.cos(val * freq)])
    return encoded

class DeformationMLP:
    def __init__(self, in_dim=3, out_dim=3, hidden_dim=256, num_layers=8, cond_dim=0):
        self.num_layers = num_layers
        self.hidden_dim = hidden_dim
        self.cond_dim = cond_dim
        self.weights = []
        self.biases = []
        self.gamma = []
        self.beta = []
        for i in range(num_layers):
            in_size = in_dim if i == 0 else hidden_dim
            out_size = out_dim if i == num_layers - 1 else hidden_dim
            # Hidden layers: Xavier normal initialization
            scale = math.sqrt(6.0 / (in_size + out_size))
            w = [random.gauss(0, scale) for _ in range(in_size * out_size)]
            b = [0.0] * out_size
            self.weights.append(w)
            self.biases.append(b)
            # FiLM parameters
            if cond_dim > 0:
                self.gamma.append([0.0] * out_size)
                self.beta.append([0.0] * out_size)
        # Final layer: weights initialized from normal distribution with std 1e-5
        final_in = hidden_dim if num_layers > 1 else in_dim
        self.weights[-1] = [random.gauss(0, 1e-5) for _ in range(final_in * out_dim)]

    def forward(self, x, z=None):
        x_enc = positional_encoding(x, 10)
        current = x_enc
        for i in range(self.num_layers):
            current = linear_layer(current, self.weights[i], self.biases[i])
            if z is not None and self.cond_dim > 0:
                # Auxiliary networks to predict gamma/beta from z are not specified in sources [G]
                # Assuming gamma/beta are pre-computed or passed directly
                current = [current[j] * self.gamma[i][j] + self.beta[i][j] for j in range(len(current))]
            # Activation function not specified in sources; applying standard ReLU [G]
            current = [max(0.0, v) for v in current]
        return current
```

## Subtopics

### CoDeF
CoDeF (Computational Deformation Field) is a deformation representation used within Neural and primitive-based representations for dynamic volume rendering, specifically in the context of 4D Gaussian Splatting [14869-codef.md]. It models temporal motion by mapping points from a canonical space to a deformed space using a multiresolution hash table [14869-codef.md]. It sets itself apart from sibling methods like [Multi-MLP Linear Layer](15116-multi-mlp-linear-layer.md) by relying on hash-grid-based deformation rather than explicit MLP parameterizations or latent code conditioning, prioritizing storage efficiency and compatibility with standard image processing pipelines [14869-codef.md].

### [Multi-MLP Linear Layer](15116-multi-mlp-linear-layer.md)
The Multi-MLP Linear Layer (also called ECNR Linear Layer) is a linear transformation implementation for the Efficient Compressive Neural Representation (ECNR) that supports parallel training of multiple MLPs by using 3D tensors for weights and biases, indexed by MLP ID. Unlike conventional linear layers that represent weights and biases as two 2D matrices, this layer uses 3D tensors where the extra dimension denotes which MLP to use, allowing parallel training of all MLPs simultaneously without complex coding.

### [Spatial-Ratio Neural Field Configuration](16086-spatial-ratio-neural-field-configuration.md)
The Spatial-Ratio Neural Field is a module that predicts position and shape transformations for selected 3D Gaussians to patch missing areas caused by masking, mapping them from an "all-selected" state to an adapted state under a given elastic ratio $e$. It serves as the core component of the Gaussian Transform Field, enabling flexible 3D Gaussian Splatting where the number of Gaussians can be adjusted dynamically without degrading rendering quality. It is distinguished from sibling methods like [Multi-MLP Linear Layer](15116-multi-mlp-linear-layer.md) by its use of a multi-plane implicit representation conditioned on a scalar elastic ratio, rather than static latent codes or simple linear projections.

### [Deformation Network (Static)](16191-deformation-network-static.md)
The Deformation Network (Static) is a neural module that maps 3D points from an original camera space to a canonical voxel space by predicting a static displacement vector. It solves the problem of temporal alignment in dynamic scenes by ensuring that corresponding 3D points from different time steps map to the same position in the canonical space. Unlike variants that predict time-varying deformations, this method assumes the geometric deformation is fixed or approximates the dominant static structure, relying on shallow MLPs to enhance features for time-varying color and density.

### [Multi-Head MLP Decoder](16267-multi-head-mlp-decoder.md)
The Multi-Head MLP Decoder is an uncertainty-aware neural field configuration that attaches multiple lightweight MLP decoders (heads) to a shared feature grid encoder. It solves the problem of estimating reconstruction uncertainty in implicit neural representations (INRs) without the prohibitive parameter cost of full ensemble methods. It sets itself apart from siblings like [Multi-MLP Linear Layer](15116-multi-mlp-linear-layer.md) by using distinct, lightweight MLP heads rather than a single linear layer with multiple outputs, and from [Latent Code Conditioning](16345-latent-code-conditioning.md) by relying on architectural diversity (multiple heads) rather than latent variable conditioning to model uncertainty.

### [Latent Code Conditioning](16345-latent-code-conditioning.md)
Latent Code Conditioning is a family of techniques for injecting instance-specific information into a Neural Field decoder by associating a unique latent vector $z_i$ with each data sample (e.g., a specific 3D volume, shape, or dynamic sequence). It solves the problem of representing multiple distinct instances using a shared neural field architecture, allowing the decoder to generalize across a dataset while preserving instance-specific details. It is distinguished from sibling methods by its specific mechanism of conditioning: [Concatenation Conditioning](12582-concatenation-conditioning.md) mixes the code into the input features, [HyperNetwork Conditioning](12585-hypernetwork-conditioning.md) generates decoder weights from the code, and [Autoencoder/Autodecoder Conditioning](12588-autoencoder-autodecoder-conditioning.md) optimizes the code as a learnable parameter alongside the network weights.

### [1D Convolutional MLP](20590-1d-convolutional-mlp.md)
The 1D Convolutional MLP is a neural network module that stacks 1D convolutional layers with kernel size 1 to process feature vectors, optionally employing residual connections and batch normalization. It is used in volume rendering to predict volume values (such as density or color) from concatenated view-dependent features and interpolated spatial features. It differs from standard fully connected MLPs by explicitly structuring the transformation as a sequence of 1D convolutions, which allows for consistent channel-wise mixing, and from sibling methods like CoDeF by relying on this specific residual 1D convolutional architecture rather than coordinate-based deformation fields or multi-head decoders.

### Fused Hash Encoding MLP Architecture
Composes the sampler from two consecutive neural density fields [S3]. Represents each density field using 'fused' MLPs that employ hash encoding to process 3D positions [S3]. Predicts density using an MLP following the hash encoding [S3]. Minimum hash grid resolution is 16 [S3]. Number of hash encoding levels is 5 [S3]. Hash feature dimension is 2-dim features per level [S3].

### MLP Lidar Encoding
Processes LiDAR features using an MLP with three hidden layers [S3]. Hidden layer feature sizes are 64, 96, 128 [S3]. Output layer feature size is a 64-dim feature vector [S3].

### Consistency Loss for Canonical Blend Weights
A loss function to optimize the neural blend weight field $w_{can}$ in canonical space by enforcing consistency between blend weights of corresponding points in observation and canonical spaces [S4]. For an observation-space point $x$ at frame $i$, maps it to the canonical-space point $T_i(x)$ using Equation (4) [S4]. Computes the L1 difference between the observation-space blend weight $w_i(x)$ and the canonical-space blend weight $w_{can}(T_i(x))$ [S4]. The coefficient weight for $L_{nsf}$ is set to 1 [S4].

## Operating characteristics

## Background

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Introduction to Neural Fields - inovex GmbH, https://www.inovex.de/de/blog/introduction-to-neural-fields/, fetched 2026-10-29 (unofficial)
[S3] LidaRF: Delving into Lidar for Neural Radiance Field on Street Scenes, https://doi.org/10.1109/cvpr52733.2024.01850, fetched 2026-10-26
[S4] Animatable Neural Radiance Fields for Modeling Dynamic Human Bodies, https://doi.org/10.1109/iccv48922.2021.01405, fetched 2026-10-26
