# NeRF-ID

Also called: NeRF in detail

## Overview
NeRF-ID (NeRF in detail) is a method that replaces the heuristic coarse-to-fine sampling of NeRF with a differentiable 'proposer' module, enabling end-to-end training of the entire system (coarse network, proposer, and fine network) to improve view synthesis quality [S1, unofficial]. Unlike the original NeRF heuristic which uses a handcrafted procedure involving coarse densities to define a piece-wise constant PDF, NeRF-ID uses a learnable proposer that takes coarse features and positions as input to directly output fine sample locations [S1, unofficial]. It builds on [Coarse-to-Fine Hierarchical Sampling](14488-coarse-to-fine-hierarchical-sampling.md).

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, direction $\mathbf{d}$, near $t_n$, far $t_f$.
*   **Outputs:** Final ray color $\hat{\mathbf{C}}$.

**Data Structures:**
*   **Coarse Samples:** $N_c$ depths $t_c^{(i)}$ for $i=0,\dots,N_c-1$.
*   **Coarse Features:** Feature vectors $\mathbf{f}_c^{(i)}$ produced by the coarse network at each $t_c^{(i)}$. These are defined as the values of the last activations before the projection that produced the density [S1, unofficial].
*   **Proposer Output:** $N_f$ proposed depths $t_f^{(j)}$ for $j=0,\dots,N_f-1$.
*   **Fine Samples:** Union of coarse and fine depths, sorted: $t_{all} = \text{sort}([t_c, t_f])$.

**Parameters:**
*   $N_c$: Number of equally spaced samples queried from the coarse network [S1, unofficial].
*   $N_f$: Number of additional samples proposed by the proposer for the fine network [S1, unofficial].
*   Typical values: Not given in the sources.

**Steps:**

1.  **Coarse Pass:**
    *   Generate $N_c$ equally spaced sample depths $t_c^{(i)}$ along the ray [S1, unofficial].
    *   Evaluate $\text{MLP}_{coarse}$ at positions $\mathbf{x}_c^{(i)} = \mathbf{o} + t_c^{(i)} \mathbf{d}$ to obtain raw outputs. Extract the feature vectors $\mathbf{f}_c^{(i)}$ (last activations before density projection) and density $\sigma_c^{(i)}$ [S1, unofficial].
    *   Compute coarse weights $w_c$ using alpha compositing to form a probability distribution [S1, unofficial].

2.  **Proposer (Hierarchical Sampling):**
    *   Prepare input for the proposer: For each coarse sample $i$, create an input vector consisting of the feature vector $\mathbf{f}_c^{(i)}$ and its normalized position $p_i = \frac{t_c^{(i)} - t_n}{t_f - t_n} \in [0, 1]$ [S1, unofficial].
    *   Process the sequence of $N_c$ input vectors through the chosen proposer architecture (Transformer, Pool, or MLPMix) to produce $N_f$ scalar outputs [S1, unofficial].
    *   Apply a sigmoid function to the scalar outputs to constrain the proposed positions to the range $[0, 1]$ [S1, unofficial].
    *   Convert normalized positions to depths: $t_f^{(j)} = t_n + (t_f - t_n) \cdot \text{sigmoid}(\text{output}_j)$ [S1, unofficial].
    *   Detach $t_c$ and $w_c$ from the computation graph to prevent gradients from flowing back through the non-differentiable sampling step to the coarse network [S4, S5, unofficial].

3.  **Fine Pass:**
    *   Concatenate coarse and fine depths: $t_{all} = \text{sort}([t_c, t_f])$ [S4, S5, unofficial].
    *   Evaluate $\text{MLP}_{fine}$ at positions $\mathbf{x}_{all}^{(k)} = \mathbf{o} + t_{all}^{(k)} \mathbf{d}$ to obtain densities $\sigma_{all}^{(k)}$ and colors $\mathbf{c}_{all}^{(k)}$ [S4, S5, unofficial].
    *   Compute final color $\hat{\mathbf{C}}$ using the rendering equation [S1, unofficial]:
        $$ \hat{\mathbf{C}} = \sum_{k=1}^{N} w_k \mathbf{c}_{all}^{(k)} $$
        where $N = N_c + N_f$, and weights are derived from density and depth intervals:
        $$ w_k = T_k \left( 1 - \exp(-\sigma_{all}^{(k)}(t_{all}^{(k+1)} - t_{all}^{(k)})) \right) $$
        $$ T_k = \exp\left(-\sum_{j=1}^{k-1} \sigma_{all}^{(j)}(t_{all}^{(j+1)} - t_{all}^{(j)})\right) $$
        Here $\sigma_{all}^{(k)}$ is density, $\mathbf{c}_{all}^{(k)}$ is color, and $t_{all}^{(k)}$ is location along the ray for the $k$-th sample [S1, unofficial].

**Proposer Architectures:**
The proposer is a small neural network. Three architectures are considered [S1, unofficial]:

1.  **Transformer Proposer:**
    *   A transformer encoder processes the input features (features + positional encodings).
    *   A transformer decoder produces the final set of proposals via cross-attention with $N_f$ learned queries [S1, unofficial].

2.  **Pool Proposer:**
    *   Each point feature $\mathbf{f}_c^{(i)}$ is summed with the positional encoding of its location $p_i$.
    *   Pass through a fully connected (FC) layer followed by ReLU.
    *   Average the processed features to form a single ray representation vector.
    *   Pass through an FC with output dimension $N_f$ to produce proposals [S1, unofficial].

3.  **MLPMix Proposer:**
    *   Stack point features sorted by their position along the ray.
    *   Apply a single MLPMix block.
    *   Average pool to get the ray representation.
    *   Decode proposals via an FC [S1, unofficial].

**Training Strategy:**
*   **Two-stage training:**
    1.  **Pre-training:** Train the network to mimic vanilla NeRF (coarse and fine networks only, or with a fixed heuristic proposer) [S1, unofficial].
    2.  **Fine-tuning:** Make the network free to learn better sampling strategies by enabling end-to-end training of the coarse network, proposer, and fine network [S1, unofficial].
*   **Loss:** L2 loss between predicted ray color $\hat{\mathbf{C}}$ and ground truth ray color [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Training from scratch can be unstable due to lack of supervision for the proposer module [S1, unofficial]. The two-stage strategy mitigates this [S1, unofficial].
*   The proposer must be small relative to the coarse and fine networks to have negligible additional parameters and computational overhead [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Adds the computational cost of the proposer module to the standard coarse-to-fine rendering. The proposer is designed to be small [S1, unofficial].
*   **Memory Complexity:** Adds memory for the proposer network parameters and intermediate activations during the proposal step [S1, unofficial].
*   **Measured Performance:** Achieves superior view synthesis quality over NeRF and state-of-the-art on the synthetic Blender benchmark and on par or better performance on real LLFF-NeRF scenes [S1, unofficial].
*   **Scaling:** By leveraging predicted sample importance, a 25% saving in computation can be achieved without significantly sacrificing rendering quality [S1, unofficial].
*   **Strengths:** End-to-end differentiability allows the system to learn optimal sampling strategies. Better scene representations than vanilla NeRF [S1, unofficial].
*   **Limitations:** Training from scratch is challenging and may fail to reach good performance without the two-stage strategy [S1, unofficial].
*   **When to use:** When view synthesis quality is prioritized and computational resources allow for the additional proposer network. When adaptive sampling based on learned importance is beneficial.
*   **When not to use:** When computational budget is extremely tight and the overhead of the proposer is unacceptable, or when training stability is a major concern and the two-stage strategy cannot be employed.

## Background
*   **Origin:** Introduced in "NeRF in detail: Learning to sample for view synthesis" [S1, unofficial].
*   **History:** Addresses the limitation of vanilla NeRF's coarse-to-fine approach, which relies on a heuristic and is not trained end-to-end [S1, unofficial].
*   **Usage:** Used for view synthesis on synthetic (Blender) and real (LLFF-NeRF) benchmarks [S1, unofficial].

## Sources
[S1] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
