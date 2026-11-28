# Piecewise Linear LOD Transition

Also called: Hash Grid LOD Transition

## Overview
A method to weight hash grid features from multiple LOD levels based on an optimal LOD level $L^*$, allowing smooth transitions between detail levels to reduce rendering time for distant points while preserving high-frequency features for close points [S1]. It sets itself apart from [Progressive Level Transitions](14190-progressive-level-transitions.md) by using a piecewise linear weight function derived from the Nyquist–Shannon sampling theorem rather than blending Gaussian opacity properties, and from [Continuous LOD (CLOD)](16305-continuous-lod-clod.md) by operating on discrete hash grid layers with explicit floor/ceiling logic rather than continuous interpolation of geometry [S1].

## How it works
**Inputs:**
- $f$: Scaling factor for the hash grid levels [S1].
- $s$: Sample spacing along the ray [S1].
- $\hat{r}$: Position of the sample in contracted coordinate space [S1].
- $L_{max}$: Maximum number of LOD levels available in the hash grid [S1].

**Outputs:**
- A set of weights $\{w_L\}$ for each LOD level $L \in [0, L_{max}]$ [S1].

**Data Structures:**
- Hash Grid: A collection of layers indexed by integer LOD level $L$. Each layer $L$ has a voxel size of $(sf^L)^{-1}$ [S1].
- Features: Stored per layer, accessed via hashing [S1].

**Parameters:**
- $f$: Typical value is 1.4 [S1].
- Optional LOD Bias $\Delta L$: Can be added to $L^*$ to adjust sharpness, balancing blurred vs. aliased rendering [S1].

**Steps:**
1. Calculate the optimal continuous LOD level $L^*$ for the current sample using the formula:
   $$L^* = -\log_f(2s\hat{r})$$
   where $L^*$ is the optimal LOD level, $f$ is the scaling factor, $s$ is the sample spacing, and $\hat{r}$ is the position in contracted coordinate space [S1].
   [G] If $L^*$ is negative, clamp it to 0, as level 0 is the base level.

2. Compute the weight $w_L$ for each LOD level $L$ from $0$ to $L_{max}$ using the piecewise linear function:
   $$w_L = \begin{cases} 1 & L \leq \lfloor L^* \rfloor \\ L^* - \lfloor L^* \rfloor & \lfloor L^* \rfloor < L \leq \lceil L^* \rceil \\ 0 & \lceil L^* \rceil < L \end{cases}$$
   where $w_L$ is the weight for LOD level $L$, and $\lfloor \cdot \rfloor$ and $\lceil \cdot \rceil$ are floor and ceiling functions [S1].

3. Query features from the hash grid layers using the computed weights $w_L$ to combine contributions from different LOD levels [S1].
   [G] In implementation, this typically involves summing the feature vectors retrieved from each layer $L$, scaled by $w_L$. If $w_L$ is 0, the query for that layer can be skipped.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_lod_weights(L_max, f, s, r_hat, lod_bias=0.0):
    """
    Computes piecewise linear LOD weights for a hash grid.
    
    Args:
        L_max (int): Maximum number of LOD levels (0 to L_max).
        f (float): Scaling factor for hash grid levels.
        s (float): Sample spacing along the ray.
        r_hat (float): Position of the sample in contracted coordinate space.
        lod_bias (float): Optional bias to adjust LOD sharpness.
        
    Returns:
        list[float]: Weights w_L for each level L from 0 to L_max.
    """
    # Step 1: Calculate optimal LOD level L*
    # L* = -log_f(2 * s * r_hat)
    # Using change of base formula: log_f(x) = ln(x) / ln(f)
    argument = 2.0 * s * r_hat
    
    # Guard against invalid log arguments
    if argument <= 0:
        L_star = 0.0
    else:
        L_star = -math.log(argument) / math.log(f)
        
    # Apply LOD bias
    L_star += lod_bias
    
    # Step 2: Compute weights for each level
    weights = []
    floor_L_star = math.floor(L_star)
    ceil_L_star = math.ceil(L_star)
    fractional_part = L_star - floor_L_star
    
    for L in range(L_max + 1):
        if L <= floor_L_star:
            w_L = 1.0
        elif floor_L_star < L <= ceil_L_star:
            w_L = fractional_part
        else:
            w_L = 0.0
        weights.append(w_L)
        
    return weights
```

**Edge Cases and Pitfalls:**
- **Precision:** Ensure floating-point precision is sufficient for calculating $L^*$, especially when $s\hat{r}$ is very small or very large [S1].
- **Boundaries:** If $L^*$ falls exactly on an integer, the weight for that level is 1, and adjacent levels are 0 [S1].
- **Degenerate Input:** If $s=0$ or $\hat{r}=0$, the argument to the logarithm is 0, which is undefined. Clamp $L^*$ to 0 in this case [G].
- **Memory Access:** Feature sampling of the finest hash grid layers is expensive due to highly incoherent memory access patterns. Skipping these features for distant points (where $w_L \approx 0$) reduces rendering time substantially [S1].

## Operating characteristics
- **Time Complexity:** Depends on the number of non-zero weights. For distant points, only low LOD levels have non-zero weights, reducing the number of hash grid queries [S1].
- **Memory Complexity:** Proportional to the total size of the hash grid across all levels [S1].
- **Measured Performance:** In room-scale scenes with 16 LOD levels, base resolution 128, and scaling factor 1.4, skipping fine layers for distant points substantially reduces rendering time [S1].
- **Strengths:** Reduces rendering time for distant points by avoiding expensive fine-layer sampling; preserves high-frequency features for close points [S1].
- **Limitations:** Feature sampling of fine hash grid layers is expensive due to incoherent memory access patterns if not skipped [S1].
- **When to use:** When rendering scenes with significant depth variation and hash grid-based feature representations [S1].
- **When not to use:** When memory access coherence is not a bottleneck or when a different LOD strategy (e.g., CLOD) is more appropriate for the data structure [S1].

## Background
- **Origin:** Introduced in VR-NeRF: High-Fidelity Virtualized Walkable Spaces [S1].
- **History:** The method is based on the Nyquist–Shannon sampling theorem, dampening features whose size is less than twice the footprint in contracted coordinate space [S1]. It is related to standard mip-mapping concepts, with an optional LOD bias $\Delta L$ that can be viewed as a unifying framework for coarse-to-fine training strategies [S1].
- **Usage:** Used in VR-NeRF for rendering high-fidelity virtualized walkable spaces [S1].

## Sources
[S1] VR-NeRF: High-Fidelity Virtualized Walkable Spaces, https://doi.org/10.1145/3610548.3618139, fetched 2026-10-28
