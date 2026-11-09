# Level of Detail Calculation

Also called: ComputeVoxelSize, LoD Level Calculation, Voxel Pre-integration, Cone Stepping and Mip Level Selection, Brick LOD Selection

## Overview
Level of Detail Calculation determines the specific resolution level $L$ to sample from a multiresolution hierarchy for a given geometric primitive or view ray. It solves the problem of mapping continuous spatial or view-dependent metrics to discrete integer levels in a mip-chain or voxel hierarchy. It sets itself apart from sibling methods by providing the fundamental mathematical mapping function that converts a raw metric (such as projected width) into a fractional level index, which is then split into integer and fractional components for interpolation or selection [S1].

## How it works
**Inputs and Outputs:**
*   Input: $M_{axLevels}$, the total number of available resolution levels in the hierarchy (integer $\ge 1$).
*   Input: $width$, the metric determining the required detail (e.g., projected width of a voxel or primitive in pixels, or a normalized distance metric).
*   Output: $level$, the fractional target level index.
*   Output: $level0$, the lower integer level index.
*   Output: $plevel0$, the fractional weight for the lower level.
*   Output: $level1$, the upper integer level index.
*   Output: $plevel1$, the fractional weight for the upper level.

**Data Structures:**
No complex data structures are required for the calculation itself. The inputs are scalar values. The outputs are typically used to index into a texture array, voxel grid, or brick hierarchy.

**Steps:**
1.  Calculate the raw fractional level index using the logarithmic relationship between the maximum levels and the input width.
2.  Compute the lower integer level and its corresponding fractional weight.
3.  Compute the upper integer level and its corresponding fractional weight.

**Formulas:**
The base calculation for the fractional level is:
$$ level = M_{axLevels} - 1 + \log_2(width) \quad (19) $$

The lower integer level $level0$ and its weight $plevel0$ are:
$$ level0 = \lceil level \rceil $$
$$ plevel0 = level - level0 \quad (20) $$

The upper integer level $level1$ and its weight $plevel1$ are:
$$ level1 = \lfloor level \rfloor $$
$$ plevel1 = level1 - level \quad (21) $$

**Parameters:**
*   $M_{axLevels}$: Typically determined by the resolution of the finest voxel grid relative to the coarsest. For example, if the finest grid is $256^3$ and the coarsest is $16^3$, $M_{axLevels} = \log_2(256/16) + 1 = 5$.
*   $width$: Must be positive. If $width$ represents a projected size in pixels, it is often clamped to $[1, \text{screen\_width}]$ to prevent out-of-bounds level indices.

**Edge Cases and Pitfalls:**
*   **Negative or Zero Width:** $\log_2(width)$ is undefined for $width \le 0$. Implementers must clamp $width$ to a minimum positive value (e.g., $10^{-6}$ or $1.0$) [G].
*   **Index Bounds:** The resulting $level0$ and $level1$ may exceed $[0, M_{axLevels}-1]$ if $width$ is outside the expected range. Clamping the final integer indices to the valid hierarchy range is required [G].
*   **Precision:** Floating-point precision in $\log_2$ can cause jitter in level selection for very small changes in $width$. Using integer arithmetic for level selection or smoothing functions can mitigate this [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_lod(max_levels: int, width: float) -> tuple:
    """
    Calculate Level of Detail indices and weights.
    
    Args:
        max_levels: Total number of levels in the hierarchy (integer).
        width: The metric determining detail (float, must be > 0).
        
    Returns:
        A tuple (level0, plevel0, level1, plevel1) where:
        - level0: Lower integer level index
        - plevel0: Weight for level0
        - level1: Upper integer level index
        - plevel1: Weight for level1
    """
    # Clamp width to avoid log(0) or log(negative)
    # [G] Minimum width of 1.0 is typical for pixel-based metrics
    if width <= 0.0:
        width = 1.0
        
    # Formula 19
    level = (max_levels - 1) + math.log2(width)
    
    # Formula 20
    level0 = math.ceil(level)
    plevel0 = level - level0
    
    # Formula 21
    level1 = math.floor(level)
    plevel1 = level1 - level
    
    return level0, plevel0, level1, plevel1
```

## Subtopics

### [Multiresolution Terrain Quantization and Prediction](9998-multiresolution-terrain-quantization-and-prediction.md)
This method assigns quantization precision to terrain vertices in a multiresolution hierarchy to minimize bitstream size while maintaining visual fidelity. It solves the problem of efficient compression for multiresolution terrain data by using specific shift values for different vertex types (Corner, Leg, Hypotenuse, Interior). It sets itself apart from generic quantization methods by leveraging the geometric connectivity of patch neighbors (specifically cousin patches and parent-child relationships) to derive shifts for boundary vertices from their children, thereby reducing the number of stored parameters.

## Operating characteristics
**Time and Memory Complexity:**
The calculation is $O(1)$ time complexity and $O(1)$ memory complexity. It involves a constant number of arithmetic operations and a logarithm.

**Measured Performance:**
Performance is negligible compared to rendering costs. The raster approach for LOD generation is noted to be faster in some contexts, but the calculation itself is hardware-independent [S1].

**Strengths and Limitations:**
*   **Strengths:** Simple, fast, and provides smooth interpolation between levels via fractional weights.
*   **Limitations:** Requires careful clamping of inputs to avoid invalid indices. The logarithmic mapping assumes a geometric progression of resolutions.

**When to Use:**
Use when a continuous metric (like projected size) needs to be mapped to a discrete multiresolution hierarchy with interpolation support.

## Background
The method originates from work on fast voxelization and level of detail for microgeometry rendering, specifically in the context of SGGX (Smooth Geometry Gradient X) models [S1]. It was developed to handle anisotropic steel tables and other microgeometries where naive averaging at coarse resolutions produces poor visual results [S1].

## Sources
[S1] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
