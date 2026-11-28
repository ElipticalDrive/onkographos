# Deformation MLP Configuration

Also called: Conditional Batch Normalization, Deformation MLP Setup, Feature-wise Linear Modulation (FiLM), Feature-wise transformations, MLP setup

## Overview
Deformation MLP Configuration defines the architecture, initialization, and conditioning mechanisms for a Multi-Layer Perceptron (MLP) that predicts spatial deformations (delta positions) for neural fields. It solves the problem of representing dynamic or variable scenes by mapping input coordinates and latent codes to displacement vectors, enabling the model to adapt its geometry based on context. It sets apart from sibling methods like [MLP with Input Skips](1643-mlp-with-input-skips.md) by focusing on the specific parameterization of deformation fields and from [Residual Block with Shortcut (ReLU MLP Extension)](10016-residual-block-with-shortcut-relu-mlp-extension.md) by employing specific initialization strategies (small final layer weights) to ensure stable optimization of small deformations.

## How it works
The method configures an MLP to output a delta position $\delta\mathbf{x}_k$ given a time $t$ and a Gaussian 3D position $\mathbf{x}_k$. The configuration involves specific architectural choices, initialization protocols, and conditioning strategies to incorporate latent codes $\mathbf{z}$.

### Inputs and Outputs
- **Inputs**:
  - Time $t$ (scalar).
  - Spatial coordinates $\mathbf{x}_k$ (3D vector).
  - Latent code $\mathbf{z}$ (variable dimension, depending on conditioning method).
- **Outputs**:
  - Delta position $\delta\mathbf{x}_k$ (3D vector).

### Data Structures and Layout
The core deformation MLP follows a fixed structure [S1]:
- **Layers**: 8 linear layers.
- **Neurons**: 256 neurons per hidden layer.
- **Input Encoding**: Positional encoding is applied to spatial coordinates $\mathbf{x}_k$ and time $t$ with frequencies $L_x = 10$ and $L_t = 10$ respectively [S1]. The encoded inputs are concatenated before entering the MLP.
- **Final Layer**: Maps the final 256 features to a 3D delta position vector.

### Initialization
Precise initialization is critical for convergence [S1]:
1. **Hidden Layers**: Weights are initialized using Xavier normal initialization. Biases are initialized to zero.
2. **Final Layer**: The weights mapping the last 256 features to the 3D delta position are initialized from a normal distribution with standard deviation $\sigma = 10^{-5}$. This ensures that initial deformations $\delta\mathbf{x}_k$ are small, promoting stable optimization [S1].

### Conditioning Mechanisms
The MLP can be conditioned on a latent code $\mathbf{z}$ to represent multiple instances or dynamic states. The method supports several conditioning strategies [S2, unofficial]:

#### 1. Concatenation Conditioning
The simplest approach [S2, unofficial]:
1. Concatenate the latent code $\mathbf{z}$ with the input features (encoded coordinates).
2. Feed the concatenated tensor into the first layer of the MLP.
- **Pitfall**: Only the first layer directly incorporates the condition. Remaining layers operate on internal feature representations, requiring the network to distinguish samples based on input alone [S2, unofficial].

#### 2. Feature-wise Linear Modulation (FiLM) / Conditional Batch Normalization
This method applies the latent code to each layer's output [S2, unofficial]:
1. For each layer $i$, use small neural networks $\boldsymbol{\beta_i}(\mathbf{z})$ and $\boldsymbol{\gamma_i}(\mathbf{z})$ to predict scale and bias vectors.
2. The size of these vectors matches the layer's output size.
3. Apply the transformation:
   $$ \hat{\boldsymbol{x}}_i = L_i(\boldsymbol{x}) \odot \boldsymbol{\gamma_i}(\mathbf{z}) + \boldsymbol{\beta_i}(\mathbf{z}) $$
   where $L_i(\boldsymbol{x})$ is the output of the $i$-th linear layer, $\odot$ is element-wise multiplication, and $\boldsymbol{\gamma_i}, \boldsymbol{\beta_i}$ are predicted from $\mathbf{z}$ [S2, unofficial].

#### 3. HyperNetwork Conditioning
A secondary network (HyperNetwork) predicts the weights of the deformation MLP decoder from $\mathbf{z}$ [S1]. This creates a distinct Neural Field (unique weights) for each input instance, allowing the primary decoder to maintain lower complexity [S1].

### Training Parameters (Typical Values)
Based on implementation details for dynamic radiance fields [S1]:
- **Optimizer**: Adam with $\beta_1 = 0.9, \beta_2 = 0.999, \epsilon = 10^{-15}$.
- **Learning Rate**: Starts at $\gamma_{MLP} = 0.0008$, decays exponentially to $1.6 \times 10^{-6}$ over 60,000 iterations.
- **Loss Weights**: $\alpha = 5$ (for real scenes) or $1$ (for synthetic); $\beta = 0.0008$.

## Subtopics

### [HyperNetwork Conditioning](12585-hypernetwork-conditioning.md)
HyperNetwork Conditioning uses a secondary neural network to predict the weights of the deformation MLP decoder from a latent code. Unlike concatenation-based conditioning, which shares decoder parameters across all instances, this approach creates a distinct Neural Field (with unique weights) for each input instance. This separation allows the primary decoder network to maintain lower complexity, resulting in a more compact representation.

### Concatenation Conditioning
Concatenation Conditioning involves concatenating the latent code with the input coordinates or features and feeding the combined tensor into the Neural Field decoder [S2, unofficial]. The primary limitation is that only the first layer directly incorporates the condition; remaining layers operate only on internal feature representations, requiring the network to distinguish samples based on input alone [S2, unofficial].

### Autoencoder/Autodecoder Conditioning
This refers to the training paradigm rather than the MLP structure itself. In the autoencoder manner, an encoder produces latent codes from input data. In the autodecoder manner, a learnable latent vector per sample is optimized jointly with the decoder weights [S2, unofficial]. Both use the decoder network to represent the Neural Field, accepting coordinates and the latent code as inputs [S2, unofficial].

### Global Conditioning
Global conditioning encodes the whole information about a data sample in one single latent vector [S2, unofficial]. This vector is typically shared across all spatial coordinates for that sample.

### Local Conditioning
Local conditioning spreads information about a data sample over multiple latent codes assigned to specific subsets of spatio-temporal coordinates (regions or time steps) [S2, unofficial]. A hybrid approach may use one global code for shared basic information and several local codes for instance-specific details [S2, unofficial].

## Operating characteristics
- **Complexity**: The base deformation MLP has $8 \times 256 \times 256$ (hidden) + $256 \times 3$ (output) parameters, approximately 524,000 weights, excluding input encoding and conditioning networks [S1].
- **Performance**: The small initialization of the final layer ($10^{-5}$) is crucial for stable optimization of deformations, preventing large initial jumps in geometry [S1].
- **Strengths**: The structured initialization and FiLM conditioning allow for efficient representation of dynamic scenes or multiple shapes with a single base architecture.
- **Limitations**: Concatenation conditioning may struggle with complex variations because deeper layers do not directly see the condition [S2, unofficial]. HyperNetwork conditioning increases memory usage due to the need to store/predict additional weights [S1].

## Background
The specific configuration of 8 layers, 256 neurons, and the $10^{-5}$ final layer initialization originates from work on optimizing depth indirectly in dynamic radiance fields using Gaussian splatting [S1]. The broader concepts of conditioning neural fields via latent codes, including FiLM (Conditional Batch Normalization) and HyperNetworks, were introduced in the context of image generation and neural field representations [S2, unofficial]. FiLM was originally proposed by [de Vries et al., 2018] [S2, unofficial], and HyperNetworks by [Ha et al., 2016] [S2, unofficial].

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Introduction to Neural Fields - inovex GmbH, https://www.inovex.de/de/blog/introduction-to-neural-fields/, fetched 2026-10-29 (unofficial)
