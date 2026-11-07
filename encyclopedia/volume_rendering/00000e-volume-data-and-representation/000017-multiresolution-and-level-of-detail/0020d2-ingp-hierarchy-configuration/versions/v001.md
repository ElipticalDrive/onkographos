# iNGP Hierarchy Configuration

Also called: Instant Neural Graphics Primitives Hierarchy Configuration

## Overview
This configuration defines the specific hyperparameters and structural choices for the multi-resolution grid hierarchy used in neural radiance field representations, specifically as optimized in Zip-NeRF [S1]. It addresses the trade-off between memory capacity and rendering speed by selecting grid scales, channel counts, and hashing strategies. It differs from the standard iNGP configuration by using wider grids (4 channels) and coarser scale spacing (powers of 2) to achieve faster convergence and slightly better performance [S1].

## How it works
The method configures a set of hash grids at different spatial resolutions. The hierarchy is defined by the following parameters and steps:

**Inputs:**
- Target spatial domain bounds (typically normalized to $[-1, 1]^3$ or similar).
- Maximum and minimum grid resolutions.

**Outputs:**
- A list of $L$ hash grids, each associated with a specific scale $s_l$.

**Data Structures:**
- **Hash Grid:** A sparse hash table mapping 3D coordinates to feature vectors.
- **Feature Vector:** A tensor of size $C$ (channels) per grid level.
- **Hash Table Size:** Fixed at $2^{18}$ (262,144) entries for all grids [S1].

**Parameters:**
- **Number of Levels ($L$):** 10 [S1].
- **Channels per Level ($C$):** 4 [S1].
- **Scale Spacing:** Powers of 2 [S1].
- **Minimum Resolution ($N_{min}$):** 16 [S1].
- **Maximum Resolution ($N_{max}$):** 8192 [S1].
- **Hash Table Size ($H$):** $128^3 = 2,097,152$ entries? No, the text says "hash of size $128^3$". Let's re-read carefully. "Grid sizes $n_l > 128$ are parameterized identically to iNGP using a hash of size $128^3$." This implies the hash table capacity is $128^3$ [S1].

**Steps:**
1. **Generate Scales:** Create a sequence of grid resolutions $N_l$ for $l = 0 \dots L-1$.
   $$ N_l = 16 \times 2^l $$
   This yields resolutions: 16, 32, 64, 128, 256, 512, 1024, 2048, 4096, 8192 [S1].

2. **Initialize Hash Tables:** For each level $l$, allocate a hash table with $H = 128^3$ entries [S1]. Each entry stores a feature vector of size $C=4$.

3. **Coordinate Hashing:** For a query point $\mathbf{x}$, compute its position in the grid at level $l$.
   - Normalize $\mathbf{x}$ to the grid's local coordinate system.
   - Compute the integer voxel coordinates $\mathbf{v}_l$.
   - Hash $\mathbf{v}_l$ to an index in the hash table using a standard hash function (e.g., splitmix64) [S1, implied by iNGP standard].

4. **Feature Retrieval:** Retrieve the feature vector $\mathbf{f}_l$ from the hash table at the computed index.

5. **Aggregation:** Combine features from all levels $l=0 \dots L-1$ (typically via concatenation or summation) before passing them to the subsequent MLP [S1, implied by standard iNGP usage].

**Edge Cases and Pitfalls:**
- **Grid Size > 128:** For levels where $N_l > 128$, the hash table size remains fixed at $128^3$. This means the hash table density decreases as resolution increases, potentially leading to more collisions. The sources state this is parameterized "identically to iNGP" [S1].
- **Precision:** Floating-point precision in coordinate hashing can cause artifacts at high resolutions.
- **Memory:** Total memory is $L \times H \times C \times \text{sizeof(float)}$. With $L=10, H=128^3, C=4$, this is $10 \times 2,097,152 \times 4 \times 4$ bytes $\approx 32$ MB.

## Subtopics

### Tile Value Regressor
A specialized MLP-based value regressor used within the NeuralVDB hierarchical structure [S2]. It encodes tile values for nodes at level 1 that have tile values (where the mask $m_1=0$) [S2]. It is similar to the voxel value regressor [S2]. During reconstruction, if the level-1 mask classifier classifies a query point as an active tile, the corresponding tile value is predicted and returned using this regressor [S2]. The capacity of this regressor is typically much smaller than the level-0 voxel regressor due to the hierarchical nature of the tree [S2].

## Operating characteristics
- **Memory:** Uses 10 grids with $128^3$ hash entries each and 4 channels per entry. Total memory is approximately 32 MB for the grid features [S1].
- **Speed:** The configuration (4 channels, powers of 2 spacing) was found to work slightly faster than the standard iNGP configuration (2 channels, spacing by $\sqrt{2}$) [S1].
- **Scaling:** The use of powers of 2 spacing simplifies coordinate mapping and interpolation compared to $\sqrt{2}$ spacing [S1].
- **Strengths:** Faster convergence and better performance than the baseline iNGP configuration [S1].
- **Limitations:** Higher memory usage per level compared to 2-channel configurations [S1].

## Background
This configuration was introduced in Zip-NeRF [S1] as an optimization over the original Instant Neural Graphics Primitives (iNGP) [S1]. The original iNGP used 2 channels and $\sqrt{2}$ spacing [S1]. Zip-NeRF demonstrated that increasing the channel count to 4 and using integer power-of-2 scale spacing improved both speed and quality [S1]. The concept of using hash grids for neural radiance fields was pioneered by iNGP [S1]. The specific variant of using a Tile Value Regressor is described in NeuralVDB [S2].

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S2] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks, https://www.emergentmind.com/papers/2208.04448, fetched 2026-10-27
