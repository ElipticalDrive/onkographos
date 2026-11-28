# Multi-scale Coordinate Transformation

Also called: PR coordinate transformation

## Overview
Multi-scale Coordinate Transformation enhances neural field representational capacity by applying a single coordinate transformation function $\gamma_i$ at multiple spatial resolutions [S1, unofficial]. It solves the problem of modeling signals containing both low-frequency smooth components and high-frequency fine details by decomposing the signal into spatially varying coefficient maps and multi-level basis maps [S1, unofficial]. It sets itself apart from single-scale transformations by explicitly encoding frequency information through scaled inputs, allowing the network to capture a broader spectrum of signal characteristics [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Scene coordinate $\mathbf{x}$, number of levels $L$, frequency vector $\mathbf{f} = [f_1, \dots, f_L]$, and a base coordinate transformation function $\gamma_i$ (e.g., sawtooth, triangular, sinusoidal, hashing, or orthogonal) [S1, unofficial].
- Output: Transformed coordinate vector $\gamma_i^{\text{R}}(\mathbf{x})$ [S1, unofficial].

**Data Structures:**
- Frequency vector $\mathbf{f}$: A list or array of $L$ floating-point numbers representing the frequency at each level [S1, unofficial].
- Coordinate vector $\mathbf{x}$: A coordinate vector representing the position in space, applicable to multi-dimensional signals [S1, unofficial].

**Parameters:**
- $L$: The number of scales or levels. Not given in the sources.
- $f_l$: The frequency at level $l$, determining the scale. A large scale basis (level 1) has a low frequency and covers a large region; a small scale basis (level $L$) has a large frequency $f_L$ covering a small region [S1, unofficial].
- Scale: Given a bounding box $[u, v]$ along one dimension, the scale at level $l$ is $(v - u)/f_l$ [S1, unofficial].

**Steps:**
1. For each level $l$ from $1$ to $L$:
   a. Scale the input coordinate $\mathbf{x}$ by the frequency $f_l$ to get $\mathbf{x}_l = \mathbf{x} \cdot f_l$ [S1, unofficial].
   b. Apply the base coordinate transformation $\gamma_i$ to $\mathbf{x}_l$ to get $\mathbf{y}_l = \gamma_i(\mathbf{x}_l)$ [S1, unofficial].
2. Concatenate the results $\mathbf{y}_1, \dots, \mathbf{y}_L$ to form the final multi-scale coordinate vector $\gamma_i^{\text{R}}(\mathbf{x})$ [S1, unofficial].

**Formula:**
The multi-scale coordinate transformation is defined as:
$$
\gamma_i^{\text{R}}(\mathbf{x}) = (\gamma_i(\mathbf{x} f_1), \gamma_i(\mathbf{x} f_2), \dots, \gamma_i(\mathbf{x} f_L))
$$
where $\mathbf{x}$ is the scene coordinate, $f_l$ is the frequency at level $l$, and $\gamma_i$ is the base coordinate transformation function [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_scale_coordinate_transformation(x, frequencies, gamma_i):
    """
    Applies a coordinate transformation at multiple scales.
    
    Args:
        x (list/tuple): Coordinate vector.
        frequencies (list): List of L frequencies [f1, f2, ..., fL].
        gamma_i (callable): Base coordinate transformation function.
        
    Returns:
        list: Concatenated transformed coordinates from all levels.
    """
    transformed_coords = []
    for f in frequencies:
        scaled_x = [coord * f for coord in x]
        transformed = gamma_i(scaled_x)
        transformed_coords.extend(transformed)
    return transformed_coords
```

**Edge Cases and Pitfalls:**
- Precision: High frequencies may lead to numerical instability if the coordinate values become very large or if the transformation function is sensitive to input magnitude [G].
- Boundary: Not given in the sources.
- Degenerate Input: If $f_l = 0$ for any level, the scaled coordinate becomes zero, which may lead to loss of information if $\gamma_i$ is not designed to handle zero inputs robustly [G].

## Subtopics

### [Sinusoidal Coordinate Transformation](18420-sinusoidal-coordinate-transformation.md)
Uses sine functions as the basis. Sets it apart by providing smooth, periodic transformations. Choose when smoothness is desired.

### [Hashing Coordinate Transformation](18421-hashing-coordinate-transformation.md)
Uses hash functions for efficient lookup. Sets it apart by enabling sparse, high-dimensional representations. Choose for memory efficiency.

### [Orthogonal Coordinate Transformation](18422-orthogonal-coordinate-transformation.md)
Uses orthogonal basis functions. Sets it apart by ensuring basis functions are uncorrelated. Choose for stable decomposition.

### [Sawtooth Coordinate Transformation](18423-sawtooth-coordinate-transformation.md)
Uses sawtooth waves. Sets it apart by providing piecewise linear periodicity. Choose for simple, efficient periodic modeling.

### [Triangular Coordinate Transformation](18424-triangular-coordinate-transformation.md)
Uses triangular waves. Sets it apart by providing continuous, piecewise linear periodicity. Choose for smoother alternatives to sawtooth.

## Operating characteristics
- Time Complexity: Not given in the sources.
- Memory Complexity: Not given in the sources.
- Measured Performance: Not given in the sources.
- Scaling: Not given in the sources.
- Strengths: Captures both low and high-frequency signal components; flexible with different base transformations [S1, unofficial].
- Limitations: Increased computational cost with more levels; potential numerical instability with high frequencies [G].
- When to use: When the signal contains a wide range of spatial frequencies and a single-scale transformation is insufficient [S1, unofficial].
- When not to use: When computational resources are limited and the signal is predominantly low-frequency [G].

## Background
Originated in the paper "Factor Fields: A Unified Framework for Neural Fields and Beyond" [S1, unofficial]. The method is part of a unified framework that includes various coordinate transformations and projection functions to represent multi-dimensional signals [S1, unofficial]. It is used in neural field representations to enhance the modeling of complex signals [S1, unofficial].

## Sources
[S1] Factor Fields: A Unified Framework for Neural Fields and Beyond, https://doi.org/10.48550/arxiv.2302.01226, fetched 2026-10-26 (unofficial)
