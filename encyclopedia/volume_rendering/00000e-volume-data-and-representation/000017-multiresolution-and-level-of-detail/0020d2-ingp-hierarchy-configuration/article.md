# iNGP Hierarchy Configuration

Also called: Instant Neural Graphics Primitives Hierarchy Configuration

## Overview
This configuration defines the specific hyperparameters and structural choices for the multi-resolution hash grid hierarchy used in neural radiance field representations, as optimized in Zip-NeRF [S1]. It differs from the original Instant Neural Graphics Primitives (iNGP) by using 4 channels per level and scale spacing by powers of 2 instead of $\sqrt{2}$ [S1]. This adjustment yields slightly faster convergence and better rendering performance [S1].

## How it works
The method configures a set of hash grids at different spatial resolutions to accelerate neural rendering [S1]. The hierarchy is defined by the following parameters and steps:

**Inputs:**
- Minimum grid resolution $N_{min} = 16$ [S1].
- Maximum grid resolution $N_{max} = 8192$ [S1].
- Number of levels $L = 10$ [S1].
- Channels per level $C = 4$ [S1].
- Hash capacity for high-resolution levels $H_{high} = 128^3$ [S1].

**Outputs:**
- A list of $L$ hash grids, each associated with a specific spatial scale $s_l$ [S1].

**Data Structures:**
- **Hash Grid:** A sparse hash table mapping 3D coordinates to feature vectors [G].
- **Feature Vector:** A tensor of size $C=4$ per grid level [S1].
- **Hash Table Size:** For levels where resolution $N_l > 128$, the hash table capacity is fixed at $H = 128^3$ entries [S1]. For $N_l \le 128$, the source does not specify the hash capacity [S1].

**Parameters:**
- Scale spacing: Powers of 2 [S1].
- Original iNGP baseline: 2 channels per level, spacing by $\sqrt{2}$ [S1].

**Steps:**
1. **Generate Scales:** Create a sequence of grid resolutions $N_l$ for $l = 0 \dots L-1$.
   $$ N_l = 16 \times 2^l $$
   This yields resolutions: 16, 32, 64, 128, 256, 512, 1024, 2048, 4096, 8192 [S1].
2. **Initialize Hash Tables:** For each level $l$, allocate a hash table [G]. If $N_l > 128$, allocate $H = 128^3$ entries, each storing a feature vector of size $C=4$ [S1].
3. **Coordinate Hashing:** For a query point $\mathbf{x}$ at level $l$:
   - Map $\mathbf{x}$ to the local coordinate system of the grid at resolution $N_l$ [G].
   - Compute integer voxel coordinates $\mathbf{v}_l$ [G].
   - Hash $\mathbf{v}_l$ to an index in the hash table using a standard hash function [G].
4. **Feature Retrieval:** Retrieve the feature vector $\mathbf{f}_l$ from the hash table at the computed index [G].
5. **Aggregation:** Combine features from all levels $l=0 \dots L-1$ before passing them to the subsequent MLP [G]. (Note: The specific aggregation method is not detailed in the provided sources [S1].)

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_hierarchy(L=10, C=4, H_high=128**3):
    grids = []
    for l in range(L):
        N_l = 16 * (2 ** l)
        if N_l > 128:
            capacity = H_high
        else:
            capacity = "Not given in the sources."
        grids.append({"level": l, "resolution": N_l, "capacity": capacity, "channels": C})
    return grids

def query_hierarchy(grids, x):
    features = []
    for grid in grids:
        # Map x to local coordinates for resolution grid["resolution"]
        v_l = compute_voxel_coords(x, grid["resolution"])
        idx = hash_function(v_l)
        f_l = grid["hash_table"][idx]
        features.append(f_l)
    # Aggregate features (method not specified in sources)
    return aggregate(features)
```

**Edge Cases and Pitfalls:**
- **Grid Size > 128:** For levels where $N_l > 128$, the hash table size remains fixed at $128^3$ [S1]. This reduces hash table density as resolution increases, potentially increasing collisions [S1].
- **Memory:** Total memory for grid features is $L \times H \times C \times 4$ bytes [S1]. With $L=10, H=128^3, C=4$, this yields approximately 32 MB according to the source [S1].
- **Precision:** Floating-point precision in coordinate hashing can cause artifacts at high resolutions [G].

## Subtopics

### Tile Value Regressor
A specialized MLP-based value regressor used within the NeuralVDB hierarchical structure [S2]. It encodes tile values for nodes at level 1 that have tile values (where the mask $m_1=0$) [S2]. It is structurally similar to the voxel value regressor [S2]. During reconstruction, if the level-1 mask classifier classifies a query point as an active tile, the corresponding tile value is predicted and returned using this regressor [S2]. The capacity of this regressor is typically much smaller than the level-0 voxel regressor due to the hierarchical nature of the tree [S2].

## Operating characteristics
- **Memory:** Uses 10 grids with $128^3$ hash entries each and 4 channels per entry [S1]. Total memory is approximately 32 MB for the grid features [S1].
- **Speed:** The configuration (4 channels, powers of 2 spacing) was found to work slightly faster than the standard iNGP configuration (2 channels, spacing by $\sqrt{2}$) [S1].
- **Scaling:** The use of powers of 2 spacing simplifies coordinate mapping and interpolation compared to $\sqrt{2}$ spacing [S1].
- **Strengths:** Faster convergence and better performance than the baseline iNGP configuration [S1].
- **Limitations:** Higher memory usage per level compared to 2-channel configurations [S1].

## Background
This configuration was introduced in Zip-NeRF [S1] as an optimization over the original Instant Neural Graphics Primitives (iNGP) [S1]. The original iNGP used 2 channels and $\sqrt{2}$ spacing [S1]. Zip-NeRF demonstrated that increasing the channel count to 4 and using integer power-of-2 scale spacing improved both speed and quality [S1]. The concept of using hash grids for neural radiance fields was pioneered by iNGP [S1]. The specific variant of using a Tile Value Regressor is described in NeuralVDB [S2].

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S2] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks, https://www.emergentmind.com/papers/2208.04448, fetched 2026-10-27
