# NeRF-ID
Also called: NeRF in detail

## Overview
NeRF-ID replaces the heuristic coarse-to-fine sampling procedure of vanilla NeRF with a differentiable 'proposer' module, enabling end-to-end training of the coarse network, proposer, and fine network to improve view synthesis quality [S1, unofficial]. Unlike the original NeRF which uses a handcrafted procedure involving coarse densities to define a piece-wise constant probability density function (PDF) for resampling, NeRF-ID uses a learnable proposer that takes coarse features and positions as input to directly output fine sample locations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by near and far planes. The coarse network outputs $N_c$ feature vectors and densities for equally spaced samples along the ray.
*   **Output:** A set of $N_f$ new sample positions along the ray for the fine network, and the final rendered color $\hat{\mathbf{C}}$.

**Data Structures:**
*   **Coarse Features:** A sequence of $N_c$ feature vectors. These are defined as the values of the last activations before the projection layer that produces the density $\sigma$ [S1, unofficial].
*   **Coarse Positions:** Scalars normalized to $[0, 1]$, where 0 corresponds to the near plane and 1 to the far plane [S1, unofficial].
*   **Proposer Output:** A set of $N_f$ scalar positions in $[0, 1]$, obtained by passing the raw output through a sigmoid function [S1, unofficial].

**Rendering Equation:**
The color $\hat{\mathbf{C}}$ for a ray is computed from $N$ samples (the union of coarse and fine samples) using the standard volume rendering equation [S1, unofficial]:

$$
\hat{\mathbf{C}} = \sum_{i=1}^N w_i \mathbf{c}_i
$$

where:
*   $\mathbf{c}_i$ is the color of the $i$-th sample.
*   $\sigma_i$ is the density of the $i$-th sample.
*   $t_i$ is the location of the $i$-th sample along the ray.
*   $w_i$ is the weight assigned to the sample:
    $$
    w_i = T_i \left( 1 - \exp(-\sigma_i(t_{i+1} - t_i)) \right)
    $$
*   $T_i$ is the transmittance up to the $i$-th sample:
    $$
    T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j(t_{j+1} - t_j)\right)
    $$

**Parameters:**
*   $N_c$: Number of equally spaced samples queried from the coarse network.
*   $N_f$: Number of additional samples proposed by the proposer for the fine network.
*   Typical values are not explicitly given in the sources, but $N_c$ is typically small (e.g., 64) and $N_f$ is larger (e.g., 128) in standard NeRF implementations, though NeRF-ID focuses on the *quality* of the $N_f$ samples rather than just their count [S1, unofficial].

**Steps:**
1.  **Coarse Sampling:** Query the coarse network on $N_c$ equally spaced samples along the ray to obtain feature vectors and densities [S1, unofficial].
2.  **Proposer Input Preparation:** Pass the $N_c$ coarse-network-produced feature vectors and their corresponding normalized positions (scalars in $[0, 1]$) into the proposer network [S1, unofficial].
3.  **Proposer Output Generation:** The proposer network processes the input and outputs $N_f$ scalar values. These values are passed through a sigmoid function to constrain them to $[0, 1]$, representing the new sample positions along the ray [S1, unofficial].
4.  **Fine Sampling:** Query the fine network on the union of the original $N_c$ coarse samples and the $N_f$ proposed fine samples (total $N_c + N_f$ samples) [S1, unofficial].
5.  **Rendering:** Compute the final ray color $\hat{\mathbf{C}}$ using the rendering equation (1) with all $N_c + N_f$ samples [S1, unofficial].

**Proposer Architectures:**
The proposer is a small neural network. Three architectures are considered [S1, unofficial]:

1.  **Transformer Proposer:**
    *   A transformer encoder processes the input features (sequence of $N_c$ feature vectors).
    *   A transformer decoder produces the final set of proposals via cross-attention with $N_f$ learned queries [S1, unofficial].

2.  **Pool Proposer:**
    *   For each of the $N_c$ points, sum the point feature with the positional encoding of its location along the ray.
    *   Pass the result through a fully connected (FC) layer followed by a ReLU non-linearity.
    *   Average the processed features to form a single "ray representation" vector.
    *   Pass this vector through an FC layer with output dimension $N_f$ to produce the proposals [S1, unofficial].
    *   *Note:* The non-linearity before pooling is required because simple average pooling of (feature + positional encoding) would lose feature-position association due to linearity [S1, unofficial].

3.  **MLPMix Proposer:**
    *   Stack the point features sorted by their position along the ray.
    *   Apply a single MLPMix block to the stacked features.
    *   Perform average pooling to get the ray representation.
    *   Decode proposals via an FC layer [S1, unofficial].

**Training Strategy:**
Training the proposer from scratch is unstable due to weak supervision (only the final ray color is available). A two-stage training strategy is used [S1, unofficial]:
1.  **Pre-training:** Train the coarse and fine networks to mimic vanilla NeRF (using heuristic sampling). The proposer is not active or is frozen/initialized to mimic the heuristic.
2.  **Fine-tuning:** Make the network free to learn better sampling strategies. The proposer is enabled and the entire system (coarse + proposer + fine) is trained end-to-end via gradient descent minimizing the L2 loss between predicted and ground truth ray colors [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Instability:** Training the proposer from scratch without pre-training can fail to reach good performance due to lack of supervision [S1, unofficial].
*   **Computational Savings:** The proposer can also output importance estimates for the samples, allowing the filtering of least promising samples to reduce computation by ~25% without sacrificing quality [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Adds the computational overhead of the proposer network to the standard NeRF coarse-to-fine pipeline. The proposer is designed to be small relative to the coarse and fine networks to have negligible additional parameters and computational overhead [S1, unofficial].
*   **Memory Complexity:** Requires storing the proposer network weights and the intermediate features from the coarse network for backpropagation through the proposer [S1, unofficial].
*   **Performance:** Achieves superior view synthesis quality over vanilla NeRF and state-of-the-art methods on the synthetic Blender benchmark, and on par or better performance on real LLFF-NeRF scenes [S1, unofficial].
*   **Strengths:** End-to-end differentiability allows the system to learn optimal sampling distributions directly from data, avoiding heuristic biases.
*   **Limitations:** Requires a two-stage training process to ensure stability. The quality of the fine network is critical for the proposer to learn effectively (chicken-and-egg problem) [S1, unofficial].

## Background
NeRF-ID (NeRF in detail) was introduced to address the limitation of vanilla NeRF's coarse-to-fine sampling, which relies on a heuristic, non-trainable piece-wise constant PDF based on coarse densities [S1, unofficial]. The method was proposed by the authors of the paper "NeRF in detail: Learning to sample for view synthesis" [S1, unofficial]. It builds on the foundational NeRF architecture [S1, unofficial].

## Sources
[S1] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
