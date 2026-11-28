# Progressive Photon Beam Mapping


## Overview
Progressive Photon Beam Mapping is a volumetric light transport method that eliminates the memory limitations of traditional photon mapping by replacing the single huge volumetric photon map with many small, iteratively generated photon maps [S1]. It builds on [Progressive Photon Mapping](6672-progressive-photon-mapping.md) by applying a multipass strategy where each small photon map is used to update shading point contributions and then discarded, thereby improving practicality for participating media [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Light source parameters, participating media properties, and scene geometry [G].
*   **Outputs:** Final radiance estimate at shading points [G].

**Data Structures:**
*   **Small Photon Map:** A temporary data structure storing photon beam data (origin, direction, power, interaction points) generated in the current iteration [G].
*   **Shading Point Accumulator:** Stores the accumulated radiance contribution for each shading point across iterations [G].

**Steps:**
1.  **Initialize:** Set up the scene and shading points. Initialize accumulators for radiance to zero [G].
2.  **Iterative Loop:** For each iteration $i = 1, 2, \dots, N$:
    a.  **Generate Small Photon Map:** Iteratively generate a small photon map [S1].
    b.  **Update Contributions:** Use the small photon map to update the contribution of the shading points [S1]. The estimator operates on a beam-to-beam basis [S1].
    c.  **Discard:** Discard the small photon map [S1].
3.  **Finalize:** Return the accumulated radiance values at the shading points [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_photon_beam_mapping(scene, media, light_params, shading_points, N):
    # Initialize accumulators for radiance to zero [G]
    radiance_accumulators = {p: 0.0 for p in shading_points}
    for i in range(1, N + 1):
        # Iteratively generate a small photon map [S1]
        # Not given in the sources: photon emission and beam tracing logic
        small_photon_map = [] 
        # Use the small photon map to update the contribution of the shading points [S1]
        # Not given in the sources: beam-to-beam estimator implementation
        for point in shading_points:
            radiance_accumulators[point] += 0.0 
        # Discard the small photon map [S1]
        del small_photon_map
    return radiance_accumulators
```

**Formulas:**
The specific mathematical formula for the beam-to-beam estimator is not provided in the sources [S1]. The method combines volumetric density estimation with Monte Carlo path tracing via Multiple Importance Sampling (MIS) in the broader context of Unified Photon Beam Path (UPBP) [S1].

**Parameters:**
*   **Number of Iterations ($N$):** Not given in the sources [S1].
*   **Photons per Iteration:** Not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Memory:** The primary advantage is reduced memory usage, but implementation must ensure strict discarding of the small photon map to prevent leaks [S1].
*   **Convergence:** Convergence speed may vary depending on the media type, such as fog [S1].

## Subtopics

### [Transient Progressive Photon Beams](8973-transient-progressive-photon-beams.md)
A volumetric light transport simulation method that uses photon beams to estimate transient radiance in participating media. It sets itself apart from its sibling by optimizing the convergence rate of the estimator through analytical minimization of the Asymptotic Mean Squared Error (AMSE). Choose it when transient radiance estimation with optimized convergence is required.

### [Transient Progressive Photon Beams Convergence Analysis](12749-transient-progressive-photon-beams-convergence-analysis.md)
Provides the theoretical foundation for optimizing the convergence rate of the radiance estimator in Transient Progressive Photon Beams. It sets itself apart from its sibling by providing the mathematical proof for why specific parameter schedules ($\alpha=2/3, \beta_T=1/2$) are chosen, rather than just describing the algorithmic steps.

## Operating characteristics
*   **Memory Complexity:** Significantly reduced compared to standard photon mapping, as it does not store a single huge volumetric photon map [S1].
*   **Time Complexity:** Not given in the sources [S1].
*   **Strengths:** Improves practicality by eliminating memory bottlenecks; suitable for participating media like fog [S1].
*   **Limitations:** Specific performance metrics and scaling behavior are not provided in the sources [S1].

## Background
The method was proposed to address the memory costs associated with huge volumetric photon maps used in previous photon-based methods [S1]. It builds on observations by Křivánek et al. that different representations (photons, beams, paths) are suitable for different media, leading to unified solutions like UPBP [S1].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
