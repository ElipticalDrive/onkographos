# Multi-Resolution Rendering for Clouds
Also called: Low-resolution first rendering for clouds, Progressive resolution cloud rendering
## Overview
Multi-resolution rendering for clouds limits the total number of volume samples by initially rendering the cloud field at a low resolution to identify regions requiring fewer samples, then accumulating the final image with higher-resolution passes [S1, unofficial].
## How it works
Inputs: A volumetric cloud density field, a camera configuration, and a target resolution $R_{\text{target}}$.
Outputs: The final accumulated cloud image $I$.

Data Structures: Not specified in the sources.

Steps:
1. Render the cloud volume at a lower resolution $R_{\text{low}}$ ($R_{\text{low}} \ll R_{\text{target}}$) to obtain a quick approximation [S1, unofficial].
2. Analyze the low-resolution result to identify pixels or regions where fewer or no additional samples are necessary [S1, unofficial].
3. Render the cloud volume at $R_{\text{target}}$, applying full ray marching only where the analysis indicates it is needed, while skipping or using fewer samples elsewhere [S1, unofficial].
4. Accumulate the low-resolution render with the higher-resolution passes to produce the final image, reducing the overall sample count [S1, unofficial].

Parameters:
- $R_{\text{low}}$: Initial low resolution. Not given in the sources.
- $R_{\text{target}}$: Final target resolution. Not given in the sources.
- Refinement threshold: Not given in the sources.

Edge Cases:
- Temporal accumulation is noted as very difficult to apply to clouds due to velocity vector computation and coherence issues [S1, unofficial].
- Resolution transitions between low and high-resolution regions may cause visible seams if not properly blended [G].

Pseudocode:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_resolution_cloud_rendering(camera, cloud_field, R_low, R_target):
    # Step 1: Low-resolution pass
    I_low = render_cloud_at_resolution(camera, cloud_field, R_low)
    
    # Step 2: Analysis to determine mask
    M_mask = compute_refinement_mask(I_low)
    
    # Step 3: High-resolution pass
    I_high = render_cloud_at_resolution(camera, cloud_field, R_target)
    
    # Step 4: Accumulation
    I_final = blend(I_low, I_high, M_mask)
    
    return I_final

def render_cloud_at_resolution(camera, cloud_field, resolution):
    # [G] Standard ray-marching: iterate over pixel grid at given resolution,
    # step through cloud_field along camera rays, accumulate transmittance and color.
    # Source does not specify implementation.
    pass

def compute_refinement_mask(I_low):
    # [G] Compute gradient magnitude or variance of I_low.
    # Return binary mask where gradient exceeds a threshold.
    # Source does not specify implementation.
    pass

def blend(I_low, I_high, M_mask):
    # [G] Linear interpolation: I_low * (1 - M_mask) + I_high * M_mask
    return I_low * (1 - M_mask) + I_high * M_mask
```
## Subtopics

### [Iteratively Increasing Resolution](9448-iteratively-increasing-resolution.md)
A rendering strategy that reduces computation time by starting at a low resolution and gradually increasing it to the target final resolution. It solves the problem of expensive high-resolution ray-surface intersection tests by deferring them until necessary, using cone-tracing to guarantee that no surface points are skipped during the resolution transition.

## Operating characteristics
Time Complexity: Not given in the sources.
Memory Complexity: Not given in the sources.
Strengths: Not given in the sources.
Limitations: Not given in the sources.
When to use: Not given in the sources.
When not to use: Not given in the sources.
## Background
Origin: The technique was proposed by an author in the three.js forum who considered it for efficient volumetric cloud rendering but did not implement it [S1, unofficial].
History: The author suspected the technique could be very effective but ultimately gave up on temporal accumulation approaches for clouds due to difficulty [S1, unofficial].
Usage: Discussed in the context of optimizing volumetric cloud rendering in web-based 3D graphics [S1, unofficial].
## Sources
[S1] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
