# View-Dependent MLP

Also called: Larger View-Dependent MLP, iNGP View-Dependent MLP

## Overview
The View-Dependent MLP replaces the small view-dependent bottleneck in InstantNGP with a larger MLP to better express view-dependent effects and reduce floaters [S1]. It addresses the problem where insufficient bottleneck capacity forces the network to explain angular color variations by reconstructing spurious semi-transparent geometry near the camera [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A bottleneck feature vector $b \in \mathbb{R}^{256}$ derived from position encoding and previous network layers.
*   **Output:** A directional emitted color vector $c \in \mathbb{R}^3$.

**Data Structures:**
The network is a 3-layer Multi-Layer Perceptron (MLP) with 256 hidden units per layer. A skip connection feeds the original bottleneck vector directly into the second hidden layer.

**Parameters:**
*   Bottleneck size: 256 [S1].
*   Hidden units per layer: 256 [S1].
*   Number of hidden layers: 3 [S1].
*   Weight decay multiplier: $10^{-9}$ (performance is largely insensitive to this value) [S1].

**Algorithm:**
Let $W_i$ and $\mathbf{b}_i$ denote the weight matrix and bias vector for layer $i$, $W_{\text{skip}}$ and $\mathbf{b}_{\text{skip}}$ denote the skip connection weights, and $\sigma(\cdot)$ be the ReLU activation [S1].
1. Compute the first hidden representation:
   $$h_1 = \sigma(W_1 b + \mathbf{b}_1)$$
2. Compute the second hidden representation with a skip connection from the input $b$:
   $$h_2 = \sigma(W_2 h_1 + W_{\text{skip}} b + \mathbf{b}_{\text{skip}})$$
3. Compute the third hidden representation:
   $$h_3 = \sigma(W_3 h_2 + \mathbf{b}_3)$$
4. Compute the final color output:
   $$c = W_4 h_3 + \mathbf{b}_4$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear(x, W, b):
    """Element-wise linear transformation: x @ W + b"""
    return x @ W + b

def relu(x):
    """Element-wise ReLU activation"""
    return x * (x > 0)

def view_dependent_mlp(bottleneck, W1, b1, W2, b2, W_skip, b_skip, W3, b3, W4, b4):
    """
    Implements the larger view-dependent MLP from Zip-NeRF.
    
    Args:
        bottleneck: Tensor of shape (..., 256) representing the view-dependent feature vector.
        W1, b1: Weight matrix and bias for layer 1.
        W2, b2: Weight matrix and bias for layer 2.
        W_skip, b_skip: Weight matrix and bias for the skip connection to layer 2.
        W3, b3: Weight matrix and bias for layer 3.
        W4, b4: Weight matrix and bias for the output layer.
        
    Returns:
        Tensor of shape (..., 3) representing the predicted RGB color.
    """
    h1 = relu(linear(bottleneck, W1, b1))
    h2 = relu(linear(h1, W2, b2) + linear(bottleneck, W_skip, b_skip))
    h3 = relu(linear(h2, W3, b3))
    color = linear(h3, W4, b4)
    return color
```

**Edge Cases and Pitfalls:**
*   **Memory Usage:** Training with naive or jittered supersampling (6×) causes out-of-memory errors, requiring half the batch size and twice the iterations to complete [S1].
*   **Floaters:** Without sufficient capacity, the network optimizes view-dependent effects by placing small, spurious geometry ("floaters") in front of the camera; this architecture mitigates that by providing adequate capacity to model the effects directly [S1].

## Subtopics

### [View-Conditioned Rendering](2783-view-conditioned-rendering.md)
A technique that conditions the RGB decoder on the camera viewpoint to model specularities. It differs from the View-Dependent MLP by keeping the alpha (opacity) branch view-independent, whereas the View-Dependent MLP focuses on expanding the MLP's architectural capacity to handle view-dependent features directly.

## Operating characteristics
*   **Memory Complexity:** Higher than compact variants due to the 256-dimensional bottleneck and 3-layer MLP structure, increasing memory consumption during training [S1].
*   **Performance:** Reduces rendering artifacts such as floaters and incorrect semi-transparent voxels on shiny surfaces compared to smaller bottlenecks [S1].
*   **Strengths:** Better expression of view-dependent effects; reduced floaters.
*   **Limitations:** Increased memory footprint; requires reduced batch size and increased iterations when using high supersampling rates to avoid out-of-memory errors [S1].
*   **When to use:** Rendering scenes with significant specularities or complex view-dependent appearance where smaller models produce artifacts.
*   **When not to use:** When memory is constrained and view-dependent effects are minimal.

## Background
Introduced in Zip-NeRF, an anti-aliased grid-based neural radiance field method [S1]. Developed to address limitations in the original InstantNGP architecture, specifically the tendency of small bottlenecks to generate floaters when explaining view-dependent effects [S1]. Part of the broader development of efficient neural radiance fields for real-time rendering.

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
