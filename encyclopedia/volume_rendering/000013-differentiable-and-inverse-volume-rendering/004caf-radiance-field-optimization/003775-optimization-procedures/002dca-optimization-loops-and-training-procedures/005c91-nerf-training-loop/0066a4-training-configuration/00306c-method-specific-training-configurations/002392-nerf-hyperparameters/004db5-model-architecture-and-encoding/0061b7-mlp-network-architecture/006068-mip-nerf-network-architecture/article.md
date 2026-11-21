# Mip-NeRF Network Architecture

Also called: Mip-NeRF RGB-D Network Architecture

## Overview
This architecture defines the specific Multi-Layer Perceptron (MLP) configuration used in the Mip-NeRF RGB-D method for predicting radiance field properties [S1]. It solves the problem of efficient neural radiance field representation by employing a smaller, more efficient network structure compared to the original NeRF baseline [S1]. While structurally similar to the standard NeRF approach, it is distinguished by its reduced hidden layer dimensions and specific output layer configuration, optimized for integration with depth-assisted training [S1].

## How it works
The method maps 3D spatial coordinates and viewing directions to volume density and RGB color using a fully-connected MLP. The input consists of a 3D point $\mathbf{x}$ and a viewing direction $\mathbf{d}$, which are first processed by the shared positional encoding function $\gamma(\cdot)$ defined in [Model Architecture and Encoding](13550-model-architecture-and-encoding.md).

**Data Structures and Layout**
The network consists of a sequence of fully-connected (FC) layers. The internal feature vectors have the following sizes:
- Input encoded position $\gamma(\mathbf{x})$: Size 96 (derived from the standard NeRF encoding of 3 dimensions $\times$ 10 frequency bands $\times$ 2 trig functions + 1 bias or similar standard configuration implied by the "96" label in the source diagram, though the text emphasizes the layer sizes) [S1].
- Hidden layers: 4 layers, each with 256 channels [S1].
- Intermediate concatenation: The encoded direction $\gamma(\mathbf{d})$ is concatenated to the features from the last hidden layer [S1].
- Output layer: 128 channels [S1].
- Final Output: RGB color vector (3 channels) [S1].

*Note: The source diagram [S1] explicitly labels the input vector size as 96 and the direction encoding input size as 27. The direction encoding $\gamma(\mathbf{d})$ size of 27 implies 3 dimensions $\times$ 4 frequency bands $\times$ 2 trig functions + 1 bias, or a specific subset of frequencies used for direction in this variant.*

**Numbered Steps**
1.  **Position Encoding**: Compute $\gamma(\mathbf{x})$ for the input 3D point $\mathbf{x}$ [S1].
2.  **Density Branch Processing**: Pass $\gamma(\mathbf{x})$ through four sequential fully-connected layers. Each layer has 256 output channels and applies the ReLU activation function [S1].
    -   Layer 1: $\mathbf{h}_1 = \text{ReLU}(\text{FC}_{256}(\gamma(\mathbf{x})))$
    -   Layer 2: $\mathbf{h}_2 = \text{ReLU}(\text{FC}_{256}(\mathbf{h}_1))$
    -   Layer 3: $\mathbf{h}_3 = \text{ReLU}(\text{FC}_{256}(\mathbf{h}_2))$
    -   Layer 4: $\mathbf{h}_4 = \text{ReLU}(\text{FC}_{256}(\mathbf{h}_3))$
3.  **Direction Encoding**: Compute $\gamma(\mathbf{d})$ for the viewing direction $\mathbf{d}$ [S1].
4.  **Feature Concatenation**: Concatenate $\gamma(\mathbf{d})$ to the output of the fourth hidden layer $\mathbf{h}_4$ [S1].
    -   $\mathbf{h}_{cat} = [\mathbf{h}_4; \gamma(\mathbf{d})]$
5.  **Color Branch Processing**: Pass $\mathbf{h}_{cat}$ through a final fully-connected layer with 128 channels [S1].
    -   $\mathbf{h}_{out} = \text{FC}_{128}(\mathbf{h}_{cat})$
6.  **Output Activation**: Apply the Sigmoid activation function to $\mathbf{h}_{out}$ to produce the final RGB radiance values [S1].
    -   $\mathbf{c} = \text{Sigmoid}(\mathbf{h}_{out})$

**Parameters**
-   Hidden layer size: 256 channels [S1].
-   Number of hidden layers: 4 [S1].
-   Output layer size: 128 channels [S1].
-   Hidden layer activation: ReLU [S1].
-   Output layer activation: Sigmoid [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def mip_nerf_rgb_d_network(x, d, pos_encoding_func):
    """
    Implements the Mip-NeRF RGB-D network architecture.
    
    Args:
        x: numpy array of shape (3,), 3D position.
        d: numpy array of shape (3,), viewing direction.
        pos_encoding_func: Function implementing gamma() from parent article.
        
    Returns:
        c: numpy array of shape (3,), RGB color values.
    """
    # Step 1: Encode position
    # Source [S1] indicates input size 96. Standard NeRF uses L=10.
    # We assume pos_encoding_func handles the dimensionality.
    x_enc = pos_encoding_func(x)
    
    # Step 2: Process through 4 hidden layers (256 channels, ReLU)
    # Layer 1
    h1 = np.maximum(0, np.dot(x_enc, W1) + b1)
    # Layer 2
    h2 = np.maximum(0, np.dot(h1, W2) + b2)
    # Layer 3
    h3 = np.maximum(0, np.dot(h2, W3) + b3)
    # Layer 4
    h4 = np.maximum(0, np.dot(h3, W4) + b4)
    
    # Step 3: Encode direction
    # Source [S1] diagram shows direction input size 27.
    d_enc = pos_encoding_func(d)
    
    # Step 4: Concatenate direction encoding to last hidden layer features
    h_cat = np.concatenate([h4, d_enc], axis=0)
    
    # Step 5: Final fully connected layer (128 channels)
    h_out = np.dot(h_cat, W5) + b5
    
    # Step 6: Sigmoid activation for RGB output
    c = 1.0 / (1.0 + np.exp(-h_out))
    
    return c
```
*Note: The weights $W_i$ and biases $b_i$ are learned parameters. The specific initialization is not provided in the sources [G: Standard Xavier/He initialization is assumed for ReLU networks, but not specified in [S1]].*

**Edge Cases and Pitfalls**
-   **Input Dimensionality**: The network expects specific input sizes for $\gamma(\mathbf{x})$ (96) and $\gamma(\mathbf{d})$ (27) as per the source diagram [S1]. Deviations in the positional encoding frequency bands $L$ will break the concatenation or layer dimensions.
-   **Output Range**: The Sigmoid activation constrains RGB values to $[0, 1]$. If the training data requires different ranges, post-processing or a different activation is needed [S1].

## Operating characteristics
-   **Efficiency**: The architecture is explicitly described as "smaller and more efficient" than the original NeRF architecture [S1]. The original NeRF uses 8 hidden layers of 256 units for density and 256 for color, whereas this variant uses 4 hidden layers of 256 units total, with a 128-unit output layer [S1].
-   **Memory**: Reduced number of parameters compared to standard NeRF due to fewer hidden layers and a smaller output layer dimension before the final RGB projection.
-   **Performance**: The source notes that the method maintains quality despite noisy depth measurements, suggesting robustness in the architectural design for RGB-D tasks [S1].

## Background
-   **Origin**: This architecture was proposed in the context of "Mip-NeRF RGB-D: Depth Assisted Fast Neural Radiance Fields" [S1].
-   **Development**: It was designed to be a more efficient alternative to the original NeRF [16] while maintaining structural similarities, specifically tailored for depth-assisted training scenarios [S1].
-   **Usage**: It is used in experiments involving simulated and real noisy depth datasets, demonstrating robustness to Gaussian noise in inverse depth measurements [S1].

## Sources
[S1] Mip-NeRF RGB-D: Depth Assisted Fast Neural Radiance Fields, https://zenodo.org/records/7945227, fetched 2026-10-26
