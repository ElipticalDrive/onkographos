# View-Dependent MLP

Also called: Larger View-Dependent MLP, iNGP View-Dependent MLP

## Overview
The View-Dependent MLP is a specialized neural network component used in neural radiance field rendering to model complex view-dependent appearance effects, such as specularities, which simpler models fail to capture. It replaces the small bottleneck representation in InstantNGP with a larger, deeper architecture to reduce rendering artifacts known as "floaters" that arise when the model attempts to explain view-dependent effects using small, spurious geometry [S1]. It builds on [NeRF MLP Architecture](910-nerf-mlp-architecture.md) by expanding the capacity of the view-dependent processing stage.

## How it works
**Inputs and Outputs:**
The module takes a view-dependent bottleneck vector $b \in \mathbb{R}^{256}$ as input. This vector is typically derived from the 3D position and viewing direction via an initial encoding stage. The output is a directional emitted color vector $c \in \mathbb{R}^3$ (RGB).

**Data Structures:**
The network consists of a 3-layer Multi-Layer Perceptron (MLP) with 256 hidden units per layer.

**Algorithm:**
1.  **Input:** Receive bottleneck vector $b$ of size 256.
2.  **Layer 1:** Compute $h_1 = \text{ReLU}(W_1 b + \beta_1)$, where $W_1 \in \mathbb{R}^{256 \times 256}$ and $\beta_1 \in \mathbb{R}^{256}$.
3.  **Layer 2:** Compute $h_2 = \text{ReLU}(W_2 h_1 + \beta_2 + b)$. This layer includes a skip connection from the input bottleneck $b$ to the pre-activation sum. $W_2 \in \mathbb{R}^{256 \times 256}$ and $\beta_2 \in \mathbb{R}^{256}$.
4.  **Layer 3:** Compute $h_3 = \text{ReLU}(W_3 h_2 + \beta_3)$. $W_3 \in \mathbb{R}^{256 \times 256}$ and $\beta_3 \in \mathbb{R}^{256}$.
5.  **Output:** Compute $c = W_4 h_3 + \beta_4$, where $W_4 \in \mathbb{R}^{3 \times 256}$ and $\beta_4 \in \mathbb{R}^3$. The result $c$ is the predicted RGB color.

**Parameters:**
*   Bottleneck size: 256 [S1].
*   Hidden units per layer: 256 [S1].
*   Number of hidden layers: 3 [S1].
*   Activation function: ReLU (standard for MLPs in this context, implied by typical usage in Zip-NeRF/InstantNGP lineage).
*   Weight decay: $10^{-9}$ (used in ablation studies, though performance is largely insensitive to the multiplier) [S1].

**Edge Cases and Pitfalls:**
*   **Memory Usage:** Larger batch sizes with supersampling (e.g., 6x) can cause out-of-memory errors due to the increased parameter count and intermediate activations compared to smaller bottlenecks [S1]. Mitigation involves reducing batch size or training for more iterations [S1].
*   **Floaters:** Without this larger capacity, the model may optimize view-dependent effects by creating small, spurious geometric structures ("floaters") in front of the camera [S1].

## Subtopics

### [View-Conditioned Rendering](2783-view-conditioned-rendering.md)
Extends this approach by conditioning the RGB decoder $V_{rgb}$ on the viewpoint to explicitly model view-dependent effects.

## Operating characteristics
*   **Complexity:** Higher memory and compute cost than smaller bottleneck variants due to the 256-unit hidden layers and skip connections.
*   **Performance:** Reduces floaters and improves rendering quality on shiny surfaces compared to smaller models [S1].
*   **Limitations:** Susceptible to out-of-memory errors during training with high supersampling rates if batch size is not adjusted [S1].

## Background
This architecture was introduced in Zip-NeRF as a modification to the InstantNGP network structure to address limitations in modeling complex view-dependent effects [S1]. It was developed to mitigate the issue of "floaters" caused by insufficient capacity in the view-dependent bottleneck.

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
