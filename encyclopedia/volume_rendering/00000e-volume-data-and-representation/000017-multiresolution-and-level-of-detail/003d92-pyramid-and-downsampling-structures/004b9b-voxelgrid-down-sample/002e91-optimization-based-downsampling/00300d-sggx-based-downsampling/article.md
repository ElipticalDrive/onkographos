# SGGX-based Downsampling
Also called: Heitz et al. downsampling

## Overview
SGGX-based Downsampling is a method for reducing the resolution of microflakes normal distributions in volume data [S1]. It addresses the challenge of aggregating microgeometry by preserving directional coherence, distinguishing itself from naive averaging approaches that collapse tangents and lose accuracy [S1]. It builds on [Optimization-based Downsampling](5825-optimization-based-downsampling.md) by focusing on the distribution-level aggregation before iterative parameter optimization.

## How it works
**Inputs and Outputs:** Not specified in the sources [S1].
**Data Structures:** Not specified in the sources [S1].
**Method Steps:** Not specified in the sources [S1].
**Formulas:** Not specified in the sources [S1].
**Parameters:** Not specified in the sources [S1].
**Edge Cases and Pitfalls:** Not specified in the sources [S1].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sggx_downsample(distributions, target_count):
    """Downsample SGGX distributions to a target count."""
    # The sources do not provide explicit algorithmic steps, formulas, or data structures for this method [S1].
    # Implementation requires clustering criteria and parameter update rules not detailed in the reference.
    return distributions
```

## Subtopics

### [Hierarchical SGGX Aggregation](7833-hierarchical-sggx-aggregation.md)
A method for progressively reducing the number of SGGX distributions within a Level of Detail block by clustering similar distributions rather than collapsing them, preserving high-frequency orientation details better than naive averaging. It solves the problem of accurately representing complex orientation data and optical density in lower-resolution volumes by evaluating samples of multiple distributions without averaging, preserving anisotropy. Choose this variant when high-frequency orientation details must be retained in lower-resolution volumes, as it outperforms naive aggregation.

## Operating characteristics
*   **Time Complexity:** Not specified in the sources [S1].
*   **Memory Complexity:** Not specified in the sources [S1].
*   **Strengths:** Produces accurate LoDs for complex multi-fiber fabrics [S1].
*   **Limitations:** The optimization-based extension requires enormous amounts of parameters, making it intractable without low-resolution volumes [S1].

## Background
The method was introduced by Heitz et al. [HDCD15] as a solution for downsampling microflakes normal distributions [S1]. It was later built upon by Zhao, Wu et al. [ZWDR16] for downsampling microflakes volumes taking into account shadowing effects through iterative optimization of scattering parameters [S1]. The approach has proven effective for challenging datasets including complex multi-fiber fabrics [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
