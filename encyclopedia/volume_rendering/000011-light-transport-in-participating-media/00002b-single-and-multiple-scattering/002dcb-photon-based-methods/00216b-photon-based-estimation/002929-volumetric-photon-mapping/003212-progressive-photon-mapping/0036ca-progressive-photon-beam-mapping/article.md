# Progressive Photon Beam Mapping
## Overview
Progressive Photon Beam Mapping is a memory-efficient volumetric light transport method that replaces the large, static photon map with a series of small, iteratively generated photon maps [S1]. It addresses the high memory costs associated with storing huge volumetric photon maps in participating media [S1]. Unlike standard photon mapping which requires storing all photons, this method discards the photon map after each iteration, significantly improving practicality for large scenes [S1].
## How it works
**Inputs:** Not given in the sources.
**Outputs:** Not given in the sources.

**Data Structures:**
- `PhotonMap`: A temporary structure storing photons for the current iteration. The exact layout is not specified in the sources.
- `ShadingPoints`: A set of points in the volume where radiance is being estimated.

**Steps:**
1. Iteratively generate a small photon map [S1].
2. Use the small photon map to update the contribution of the shading points [S1].
3. Discard the small photon map [S1].

**Parameters:**
- Number of iterations: Not given in the sources.
- Number of photons per iteration: Not given in the sources.

**Edge Cases and Pitfalls:**
- Convergence speed may vary depending on the media type, as the method converges faster for some media like fog [S1].
- Precision issues may arise if the number of iterations is insufficient or if the photon density is too low [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_photon_beam_mapping(shading_points, num_iterations):
    # Initialize accumulated radiance for each shading point
    radiance = [0.0] * len(shading_points)
    
    for _ in range(num_iterations):
        # Generate a small photon map by tracing photon beams from light sources into the participating media and storing their interactions [S1]
        photon_map = []
        
        # Use the small photon map to update the contribution of the shading points; the estimator is beam-to-beam [S1]
        for i in range(len(shading_points)):
            # Beam-to-beam density estimation step not detailed in the sources
            radiance[i] += 0.0
            
        # Discard the small photon map to free memory [S1]
        del photon_map
        
    return radiance
```
## Subtopics

### [Transient Progressive Photon Beams](8973-transient-progressive-photon-beams.md)
A volumetric light transport simulation method that estimates transient radiance in participating media. It builds on Progressive Photon Beam Mapping by optimizing the convergence rate of the estimator through analytical minimization of the Asymptotic Mean Squared Error (AMSE). Choose this variant when simulating time-resolved light transport in participating media.

## Operating characteristics
- **Memory Complexity:** Significantly reduced compared to standard photon mapping, as only a small photon map is stored at any time [S1].
- **Time Complexity:** Depends on the number of iterations and photons per iteration. Not given in the sources.
- **Performance:** Improves practicality by eliminating memory limitations [S1]. Convergence can be faster for certain media like fog [S1].
- **Strengths:** Memory efficiency and suitability for large scenes [S1].
- **Limitations:** Requires multiple passes and convergence speed may vary depending on the media type [S1].
## Background
Progressive Photon Beam Mapping was proposed to address the memory limitations of previous photon-based methods that relied on huge volumetric photon maps [S1]. It builds on the observation that different representations (photons, beams, paths) are suitable for different types of participating media, with beams being particularly suitable for low-order scattering objects [S1]. The method is part of a broader effort to create unified solutions for light transport in participating media, such as the Unified Photon Beam Path (UPBP) method which combines volumetric density estimation and Monte Carlo path tracing through multiple importance sampling [S1].
## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
