# Progressively-connected Light Field Network (ProLiF) Architecture


## Overview
ProLiF is a neural light field representation that decouples view-independent geometry and material properties from view-dependent lighting, enabling efficient view synthesis by training with a progressively connected architecture [S1, unofficial]. It sets itself apart from standard NeRF or NeLF by replacing a single monolithic MLP with a set of independent subnetworks that are merged during training, enforcing an inductive bias for consistent scene geometry at early stages to prevent overfitting and improve detail reconstruction [S1, unofficial].

## How it works
ProLiF builds on [Specific Model Architectures](8585-specific-model-architectures.md) by decomposing the scene representation into $N_k$ independent subnetworks at training stage $k$, which are progressively merged to form a single fully-connected MLP by the final stage.

**Inputs and Outputs:**
*   **Input:** Ray parameters $(u, v, s, t)$ and depth samples. At stage $k$, the input vector $p_k$ consists of sampled $x-y$ coordinates: $p_k = [x^k_0, y^k_0, \dots, x^k_{D_k-1}, y^k_{D_k-1}]$ [S1, unofficial].
*   **Output:** Concatenated color and density values for the sampled points [S1, unofficial].

**Data Structures:**
*   **Subnetworks:** At stage $k$, there are $N_k = D \times 2^{-k}$ subnetworks $\Phi^k = \{F^k_0, F^k_1, \dots, F^k_{N_k-1}\}$ [S1, unofficial].
*   **Hidden Width:** Each subnetwork has a hidden width of $W \times 2^k$, where $W$ is a preset parameter [S1, unofficial]. The total hidden width across all subnetworks remains constant ($D \times W$) throughout training [S1, unofficial].
*   **Depth Samples:** The number of depth samples at stage $k$ is $D_k = D \times 2^k$ [S1, unofficial].
*   **Parameters:** $D$ (initial number of subnetworks/depth samples) and $W$ (base hidden width) are preset [S1, unofficial].

**Numbered Steps:**

1.  **Initialization (Stage $k=0$):**
    *   Initialize $N_0 = D$ independent subnetworks.
    *   Each subnetwork $F^0_i$ has hidden width $W$.
    *   Input coordinates are sampled at $D_0 = D$ depth locations.

2.  **Forward Pass:**
    *   For each ray, compute sampled $x-y$ coordinates $x^k_i, y^k_i$ based on ray origin and direction.
    *   Each subnetwork $F^k_i$ processes its assigned input coordinates to predict density $\sigma$ and color $\mathbf{c}$ [S1, unofficial].
    *   Rendered color is computed via volumetric rendering using the predicted $\sigma$ and $\mathbf{c}$ [S1, unofficial].

3.  **Progressive Connection (Merging Subnetworks):**
    *   At the end of stage $k$, merge neighboring subnetworks $F^k_{2i}$ and $F^k_{2i+1}$ into a single subnetwork $F^{k+1}_i$ for $i \in \{0, \dots, D \times 2^{-(k+1)} - 1\}$ [S1, unofficial].
    *   **Weight Matrix Construction:** The new weight matrix $W^{k+1}_i$ is constructed by placing the weights of the two merged subnetworks on the diagonal:
        $$
        W^{k+1}_i = \begin{bmatrix} W^k_{2i} & 0 \\ 0 & W^k_{2i+1} \end{bmatrix}
        $$
        This ensures the output $z^{k+1}_i = [z^k_{2i}, z^k_{2i+1}]$ remains unchanged [S1, unofficial].
    *   **Bias Vector Construction:** The new bias vector is the concatenation of the original biases:
        $$
        b^{k+1}_i = [b^k_{2i}, b^k_{2i+1}]
        $$
        [S1, unofficial].

4.  **Depth Subdivision (Input Expansion):**
    *   When switching from stage $k$ to $k+1$, the number of depth samples doubles from $D_k$ to $D_{k+1} = D \times 2^{k+1}$ [S1, unofficial].
    *   The input vector $p$ changes from $p_k$ to $p_{k+1}$, requiring the network to handle double the input dimension [S1, unofficial].
    *   **Reparameterization:** To maintain output invariance despite the input dimension doubling, the weights and biases of the first and last layers are reparameterized using `subdiv_input_layers` [S1, unofficial].

5.  **Final Stage:**
    *   When $k = \log_2(D)$, the network consists of a single fully-connected MLP ($\Phi^k = \{F^k_0\}$) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def subdiv_input_layers(org_weight, org_bias):
    """
    Reparameterizes weights and biases to handle input dimension doubling
    during depth subdivision, ensuring output invariance.
    
    Args:
        org_weight: numpy array of shape [n_subnetworks, d_out, d_in]
        org_bias: numpy array of shape [n_subnetworks, d_out]
        
    Returns:
        new_weight: numpy array of shape [n_subnetworks, d_out, 2*d_in]
        new_bias: numpy array of shape [n_subnetworks, d_out]
    """
    bias = org_bias
    # Rearrange weight to separate the two halves of the input dimension
    # Original shape: [n, o, d*c] where c=2 (for x and y coordinates)
    # Rearranged shape: [n, o, d, c]
    weight = rearrange(org_weight, 'n o (d c) -> n o d c', c=2)
    
    # Concatenate the weight matrix along the input dimension (dim=3)
    # and scale by 0.5 to maintain the same output magnitude
    weight = concatenate([weight, weight], dim=3) * 0.5
    
    # Rearrange back to the original flattened input dimension format
    # New shape: [n, o, 2*d*c]
    weight = rearrange(weight, 'n o d c -> n o (d c)')
    
    return weight, bias

def add_connections(org_weight, org_bias):
    """
    Merges neighboring subnetworks by constructing block-diagonal weight matrices.
    
    Args:
        org_weight: numpy array of shape [n_subnetworks, d_out, d_in]
        org_bias: numpy array of shape [n_subnetworks, d_out]
        
    Returns:
        new_weight: numpy array of shape [n_subnetworks/2, 2*d_out, 2*d_in]
        new_bias: numpy array of shape [n_subnetworks/2, 2*d_out]
    """
    n_subnetworks, d_out, d_in = org_weight.shape
    new_n = n_subnetworks // 2
    new_weight = np.zeros((new_n, 2 * d_out, 2 * d_in))
    new_bias = np.zeros((new_n, 2 * d_out))
    
    for i in range(new_n):
        # Merge F_{2i} and F_{2i+1} into F_i
        # Construct block-diagonal weight matrix
        new_weight[i, :d_out, :d_in] = org_weight[2 * i]
        new_weight[i, d_out:, d_in:] = org_weight[2 * i + 1]
        
        # Concatenate bias vectors
        new_bias[i, :d_out] = org_bias[2 * i]
        new_bias[i, d_out:] = org_bias[2 * i + 1]
        
    return new_weight, new_bias
```

**Edge Cases and Pitfalls:**
*   **Output Invariance:** Failure to reparameterize weights during subdivision will cause the network output to change, breaking the progressive training continuity [S1, unofficial].
*   **Precision:** The scaling factor of 0.5 in `subdiv_input_layers` is critical for maintaining numerical stability and output consistency [S1, unofficial].

## Subtopics

### [Progressive Training Scheme](17248-progressive-training-scheme.md)
A training strategy that incrementally increases neuron connections across training stages to prevent overfitting and improve detail reconstruction. It sets itself apart from direct MLP training by enforcing an inductive bias for consistent scene geometry at the start, avoiding the overfitting and geometric inaccuracies typical of standard end-to-end optimization.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** The total hidden width remains constant ($D \times W$) throughout training, but the number of subnetworks decreases while their individual width increases [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The architecture scales by increasing $D$ (initial subnetworks) and $W$ (base width), with the final network being a single MLP of width $D \times W$ [S1, unofficial].
*   **Strengths:** Prevents overfitting in early stages, improves detail reconstruction, and ensures geometric consistency [S1, unofficial].
*   **Limitations:** Requires careful reparameterization to maintain output invariance during transitions [S1, unofficial].
*   **When to use:** When efficient view synthesis is required with high-quality detail reconstruction, especially in scenarios where standard NeRF might overfit or produce geometric inaccuracies [S1, unofficial].
*   **When not to use:** When a simple, single-stage training process is preferred, or when computational resources for managing multiple subnetworks are limited [S1, unofficial].

## Background
ProLiF was introduced in a 2022 preprint titled "Progressively-connected Light Field Network for Efficient View Synthesis" [S1, unofficial]. It addresses the limitations of standard neural radiance field training by introducing a progressive connection mechanism that starts with sparse, independent subnetworks and merges them over time [S1, unofficial].

## Sources
[S1] Progressively-connected Light Field Network for Efficient View Synthesis, https://doi.org/10.48550/arxiv.2207.04465, fetched 2026-10-27 (unofficial)
