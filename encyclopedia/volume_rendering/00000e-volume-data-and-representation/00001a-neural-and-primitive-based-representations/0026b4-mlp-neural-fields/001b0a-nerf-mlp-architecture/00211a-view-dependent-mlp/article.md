# View-Dependent MLP

Also called: Larger View-Dependent MLP, iNGP View-Dependent MLP

## Overview
This method replaces the small bottleneck in Instant Neural Graphics Primitives (iNGP) with a larger view-dependent MLP to better express complex view-dependent effects and reduce the formation of "floaters" (spurious geometry) [S1]. It builds on [NeRF MLP Architecture](910-nerf-mlp-architecture.md) by expanding the capacity of the view-dependent color decoding stage [S1].

## How it works
**Inputs and Outputs:**
*   Input: A 256-dimensional bottleneck vector $b$ representing the view-dependent features [S1].
*   Output: Directional emitted color $c$ [S1].

**Data Structures:**
*   Bottleneck vector $b \in \mathbb{R}^{256}$ [S1].
*   MLP weights and biases for a 3-layer network with 256 hidden units [S1].

**Formulas:**
The forward pass computes the color $c$ as:
$$h_1 = \text{ReLU}(W_1 b + \beta_1)$$
$$h_2 = \text{ReLU}(W_2 h_1 + \beta_2 + b)$$
$$h_3 = \text{ReLU}(W_3 h_2 + \beta_3)$$
$$c = W_{\text{out}} h_3 + \beta_{\text{out}}$$
where $W_i, \beta_i$ are the weight matrices and bias vectors for hidden layer $i$, and $W_{\text{out}}, \beta_{\text{out}}$ are the output layer parameters [S1].

**Steps:**
1. Input the 256-dimensional bottleneck vector $b$ [S1].
2. Apply a linear transformation followed by a ReLU activation to produce the first hidden representation $h_1$ [S1].
3. Apply a second linear transformation, add the original bottleneck vector $b$ as a skip connection, and apply ReLU activation to produce $h_2$ [S1].
4. Apply a third linear transformation followed by a ReLU activation to produce $h_3$ [S1].
5. Apply a final linear output layer to produce the directional color $c$ [S1].

**Parameters:**
*   Bottleneck size: 256 [S1].
*   Hidden units per layer: 256 [S1].
*   Number of hidden layers: 3 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear(x, W, b):
    return x @ W + b

def relu(x):
    return [max(0.0, v) for v in x]

def view_dependent_mlp(bottleneck, W1, b1, W2, b2, W3, b3, W_out, b_out):
    """
    Implements the larger view-dependent MLP from iNGP.
    
    Args:
        bottleneck: A list of 256 floats representing the view-dependent features.
        W1, b1: Weights and biases for the first hidden layer.
        W2, b2: Weights and biases for the second hidden layer.
        W3, b3: Weights and biases for the third hidden layer.
        W_out, b_out: Weights and biases for the output layer.
        
    Returns:
        A list of floats representing the emitted color.
    """
    # Layer 1: Linear transformation followed by ReLU
    h1 = relu(linear(bottleneck, W1, b1))
    
    # Layer 2: Linear transformation, skip connection from bottleneck, then ReLU
    h2 = relu(linear(h1, W2, b2) + bottleneck)
    
    # Layer 3: Linear transformation followed by ReLU
    h3 = relu(linear(h2, W3, b3))
    
    # Output layer: Linear transformation to color
    color = linear(h3, W_out, b_out)
    
    return color
```

## Subtopics

### View-Conditioned Rendering
Conditioning the RGB decoder $V_{rgb}$ on the viewpoint models view-dependent effects such as specularities without specifying a functional form [S2]. The normalized camera direction is input to the decoder alongside the encoding [S2]. The architecture uses separate convolutional branches for RGB and $\alpha$ values, conditioning only the RGB branch on the viewpoint [S2]. On a human face dataset, view conditioning reduces validation MSE from 117.8 to 85.7 [S2]. The view-conditioned model outperforms non-view-conditioned models on validation viewpoints, as non-view-conditioned models incorrectly reconstruct extra semi-transparent voxels near the surface to explain view-dependent phenomena [S2].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** The larger model reduces floaters and improves rendering quality on shiny surfaces compared to smaller bottlenecks [S1].
*   **Strengths:** Better expression of view-dependent effects [S1].
*   **Limitations:** Increased model size compared to smaller bottlenecks [S1].

## Background
This method was introduced in the context of iNGP (Instant Neural Graphics Primitives) to address limitations in capturing complex view-dependent effects [S1]. It is part of the broader effort to improve neural radiance field rendering quality and reduce artifacts like floaters [S1].

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S2] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
