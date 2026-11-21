# Output Activation & Time Encoding

Also called: Output Activation, Time Encoding

## Overview
This module handles the final transformation of neural features into visible RGB color and the encoding of temporal dynamics for dynamic scene representation. It solves the problem of mapping latent feature vectors to physically plausible color values and representing time-varying scene geometry or appearance. It differs from its siblings by explicitly separating the handling of static spatial features from dynamic temporal inputs, using either linear or sinusoidal encoding for time depending on the specific variant, and applying dataset-specific output constraints.

## How it works
The method processes input features and a time variable to produce an RGB color output.

**Inputs:**
- Feature vector $\mathbf{f}$ from the preceding MLP or feature grid.
- Time variable $t$, normalized to a specific range (e.g., $[-1.0, 1.0]$ or $[-0.8, 0.8]$).

**Outputs:**
- RGB color vector $\mathbf{c} \in [0, 1]^3$.

**Data Structures:**
- No specific complex data structures are defined for this module itself; it operates on scalar or vector inputs.

**Steps:**

1. **Time Encoding:**
   The time variable $t$ is encoded. The base method uses a linear form without positional encoding [S1]. However, variants may use sinusoidal encoding [S2].
   
   *Linear Form (Base):*
   $$ \mathbf{e}_t = t $$
   
   *Sinusoidal Form (Variant):*
   $$ \mathbf{e}_t = [\sin(2^0 \pi t), \cos(2^0 \pi t), \dots, \sin(2^{L-1} \pi t), \cos(2^{L-1} \pi t)] $$
   where $L$ is the number of sinusoidal functions [S2].

2. **Feature Concatenation:**
   The encoded time vector $\mathbf{e}_t$ is concatenated with the input feature vector $\mathbf{f}$ (and potentially view direction, though the parent article notes position is not encoded, time is a distinct dynamic input) [S1, S2]. Let the combined input be $\mathbf{x}_{in}$.

3. **Color Prediction:**
   A final linear layer or small MLP predicts the raw RGB values $\mathbf{c}_{raw}$ from $\mathbf{x}_{in}$ [S1, S2].

4. **Output Activation:**
   The raw color values are constrained to the valid display range $[0, 1]$ using one of two methods depending on the dataset [S1]:
   - **Sigmoid:** Applied when training on the Neural 3D dataset.
     $$ \mathbf{c} = \sigma(\mathbf{c}_{raw}) = \frac{1}{1 + e^{-\mathbf{c}_{raw}}} $$
   - **Clamp:** Applied for other datasets (e.g., Google Immersive, Technicolor).
     $$ \mathbf{c} = \text{clamp}(\mathbf{c}_{raw}, 0, 1) $$

**Parameters:**
- $L$: Number of sinusoidal functions for time encoding. Default is 10 [S2].
- Time normalization range: Typically $[-1.0, 1.0]$, but can be reduced to $[-0.8, 0.8]$ for K-Planes experiments without harming performance [S2].

**Edge Cases and Pitfalls:**
- If $L=0$ in sinusoidal encoding, the model fails to represent dynamic scenes [S2].
- Using $L=5$ yields lower quality in dynamic regions compared to $L=10$ [S2].
- Precision issues may arise if the time normalization range is not consistent with the training data distribution [S2].

## Subtopics

### Sinusoidal Time Encoding
A variant where the normalized time input is encoded using a set of sinusoidal functions to better represent scene movement in implicit function-based approaches [S2]. It sets itself apart from the linear time encoding by providing higher frequency resolution for dynamic changes. Choose this when modeling complex temporal dynamics in implicit fields. The number of functions $L$ defaults to 10, as $L=0$ fails to represent dynamics and $L=5$ offers lower quality than $L=10$ [S2].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of sinusoidal functions $L$ if used; otherwise constant for linear encoding.
- **Memory Complexity:** $O(L)$ for storing sinusoidal coefficients if precomputed, or negligible if computed on the fly.
- **Strengths:** Sinusoidal encoding allows the network to capture high-frequency temporal variations [S2]. Linear encoding is simpler and sufficient for some datasets [S1].
- **Limitations:** Sinusoidal encoding requires tuning $L$; too few functions lose detail, too many may overfit or increase computation [S2].
- **When to use:** Use sinusoidal encoding for dynamic scenes in implicit representations [S2]. Use linear encoding with dataset-specific activation (sigmoid/clamp) for explicit feature splatting methods [S1].

## Background
The use of sigmoid activation for Neural 3D dataset and clamp for others originates from Spacetime Gaussian Feature Splatting [S1]. The sinusoidal time encoding for implicit dynamic NeRFs was introduced in Sync-NeRF [S2].

## Sources
[S1] Spacetime Gaussian Feature Splatting for Real-Time Dynamic View Synthesis, https://huggingface.co/papers/2312.16812, fetched 2026-10-30
[S2] Sync-NeRF: Generalizing Dynamic NeRFs to Unsynchronized Videos, https://doi.org/10.1609/aaai.v38i3.28057, fetched 2026-10-26
