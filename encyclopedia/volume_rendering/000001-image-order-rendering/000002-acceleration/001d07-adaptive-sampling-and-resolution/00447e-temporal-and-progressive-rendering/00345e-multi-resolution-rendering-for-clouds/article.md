# Multi-Resolution Rendering for Clouds
Also called: Low-resolution first rendering for clouds, Progressive resolution cloud rendering
## Overview
Multi-resolution rendering for clouds limits the total number of volume samples by initially rendering the cloud field at a low resolution to identify regions requiring fewer samples, then accumulating the final image with higher-resolution passes [S1, unofficial]. It distinguishes itself from sibling methods like [Temporal and Progressive Rendering](11267-temporal-and-progressive-rendering.md) by varying the spatial resolution of the rendering pass itself rather than just the sampling density or temporal reuse.
## How it works
Inputs: A volumetric cloud density field, a camera configuration, and a target resolution $R_{\text{target}}$.
Outputs: The final accumulated cloud image $I$.

Data Structures: Not specified in the sources.

Steps:
1. **Low-Resolution Pass:** Render the cloud volume at a lower resolution $R_{\text{low}}$ ($R_{\text{low}} \ll R_{\text{target}}$) to obtain a quick approximation [S1, unofficial].
2. **Analysis:** Use the low-resolution result to identify pixels or regions where fewer or no additional samples are necessary [S1, unofficial].
3. **High-Resolution Pass:** Render the cloud volume at $R_{\text{target}}$, applying full ray marching only where the analysis indicates it is needed, while skipping or using fewer samples elsewhere [S1, unofficial].
4. **Accumulation:** Combine the low-resolution render with the high-resolution passes to produce the final image, reducing the overall sample count [S1, unofficial].

Parameters:
- $R_{\text{low}}$: Initial low resolution. Not given in the sources.
- $R_{\text{target}}$: Final target resolution. Not given in the sources.
- Refinement threshold: Not given in the sources.

Edge Cases:
- **Temporal Coherence:** The author noted that applying temporal accumulation to clouds is very difficult [S1, unofficial].
- **Resolution Transitions:** Sharp boundaries between low and high-resolution regions may cause visible seams. Not given in the sources.

Pseudocode:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_resolution_cloud_rendering(camera, cloud_field, R_low, R_target):
    # Step 1: Low-resolution pass
    I_low = render_cloud_at_resolution(camera, cloud_field, R_low, samples_per_ray=coarse)
    
    # Step 2: Analysis to determine mask
    # Metric based on density or gradient from I_low
    M_mask = compute_refinement_mask(I_low)
    
    # Step 3: High-resolution pass
    # Render at target resolution, but only fully sample where needed
    I_high = render_cloud_at_resolution(camera, cloud_field, R_target, samples_per_ray=full)
    
    # Step 4: Accumulation
    # Blend I_low and I_high based on M_mask
    # Note: Exact blending formula not specified in sources
    I_final = blend(I_low, I_high, M_mask)
    
    return I_final
```
## Subtopics
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
