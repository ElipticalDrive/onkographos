# Multi-Resolution Feature Grids with MLP Decoder

Also called: Frequency encoding, Hash grid encoding, Instant-NGP Hash Encoding, Multi-resolution Hash Grid Encoding

## Overview
This method enhances neural implicit representations by replacing a single feature grid with a set of multi-resolution feature grids $\{\Phi_l^\theta\}_{l=1}^L$, indexed via spatial hash functions at finer levels to manage memory [S2]. It solves the problem of capturing multi-scale geometric details without the cubic memory explosion associated with dense grids [S2]. It sets itself apart from [Standard Multi-resolution Hash Encoding](13152-standard-multi-resolution-hash-encoding.md) by explicitly conditioning an MLP decoder on the concatenated interpolated features from all resolution levels, rather than using the hash encoding solely as a positional encoding input [S1].

## How it works
**Inputs:**
- 3D point $x \in \mathbb{R}^3$ [S2].
- Number of resolution levels $L$ [S2].
- Coarsest resolution $R_{min}$ and finest resolution $R_{max}$ [S2].
- Feature grids $\{\Phi_l^\theta\}_{l=1}^L$, where each grid stores learnable feature vectors [S1].
- MLP decoder $f_\theta$ [S1].
- Positional encoding function $\gamma(\cdot)$ [S2].

**Outputs:**
- Predicted scalar value $\hat{s}$ (e.g., SDF or density) [S2].

**Data Structures:**
- **Feature Grids $\Phi_l^\theta$**: Each level $l$ is constructed using a hash table of size $T$, where each entry stores $n$ trainable parameters (features) [S1]. For finer levels, spatial hash functions index the feature vectors to handle cubic cell growth [S2].
- **MLP $f_\theta$**: A multi-layer perceptron that takes the positional encoding of $x$ and the concatenated interpolated features as input [S1].

**Steps:**
1. **Calculate Resolutions**: Compute the resolution $R_l$ for each level $l \in \{1, \dots, L\}$ using geometric spacing:
   $$ R_l = \left\lfloor R_{min} \left( \frac{R_{max}}{R_{min}} \right)^{\frac{l-1}{L-1}} \right\rfloor $$
   This ensures resolutions are sampled in geometric space to combine features at different frequencies [S2].
2. **Interpolate Features**: For each level $l$, extract the feature vector $z_l$ by interpolating the feature grid $\Phi_l^\theta$ at point $x$:
   $$ z_l = \text{interp}(x, \Phi_l^\theta) $$
   The interpolation method (e.g., trilinear) depends on the grid structure [S1].
3. **Concatenate Features**: Combine the feature vectors from all levels into a single vector $Z$:
   $$ Z = \text{concat}(z_1, z_2, \dots, z_L) $$
   The final feature vector has size $N = n \times L$ [S1].
4. **Encode Position**: Apply positional encoding to the input point:
   $$ \gamma(x) $$
5. **Decode**: Pass the positional encoding and concatenated features through the MLP decoder:
   $$ \hat{s} = f_\theta(\gamma(x), Z) $$
   This input encoding method is key to supporting high-fidelity volume rendering [S1].

**Parameters:**
- $R_{min}, R_{max}$: Define the range of spatial frequencies [S2].
- $L$: Number of resolution levels [S2].
- $T$: Hash table size per level, fixed to bound memory usage for fine-level hash grids [S1].
- $n$: Number of trainable parameters (features) per hash table entry [S1].

**Edge Cases and Pitfalls:**
- **Memory Growth**: Total grid cells grow cubically with resolution [S2]. This is mitigated by using a fixed number of parameters and spatial hash functions for indexing at finer levels [S2].
- **Precision**: Interpolation at multiple resolutions requires consistent coordinate scaling to avoid aliasing or blending artifacts across levels [G].

## Subtopics

### [Hierarchical Implicit Grids (HIGS)](21083-hierarchical-implicit-grids-higs.md)
Distinguishes between dense and sparse grid indexing for geometric vs. semantic features and uses a fixed decoder to prevent catastrophic forgetting.

## Operating characteristics
- **Memory**: Linear in the number of active cells, but bounded by the fixed parameter count for fine levels, avoiding cubic growth [S2].
- **Compute**: Requires interpolation at multiple resolutions and MLP evaluation [S1].
- **Strengths**: Captures multi-scale features efficiently; accelerates overall training time and reduces the number of training steps required to achieve desired quality compared to fV-SRN [S1].
- **Limitations**: Hash collisions may affect accuracy; requires careful tuning of $R_{min}, R_{max}, L$ [G].

## Background
Originated in the context of neural implicit surface reconstruction, specifically in MonoSDF, which explores monocular geometric cues [S2]. It builds on the idea of multi-resolution feature grids introduced in prior works [10, 22, 45, 69, 92] as cited in [S2]. The method was also adopted for scientific volume visualization due to its effectiveness in training MLPs [S1].

## Sources
[S1] Interactive Volume Visualization via Multi-Resolution Hash Encoding ..., https://arxiv.org/abs/2207.11620, fetched 2026-10-26
[S2] MonoSDF: Exploring Monocular Geometric Cues for Neural Implicit Surface Reconstruction, https://doi.org/10.52202/068431-1814, fetched 2026-10-26
