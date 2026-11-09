# Continuous (Progressive) Levels of Detail

Also called: Hierarchical Level-of-Detail Representation, LOD representation, Multi-resolution hierarchies, Progressive Levels of Detail, View-Dependent Levels of Detail

## Overview
Continuous (Progressive) Levels of Detail (LoD) is a technique for generating multi-resolution hierarchies for 3D data volumes or scenes by creating a continuous-resolution representation that allows smooth transitions between approximations [S1, S3, unofficial][S2, unofficial]. Unlike discrete LoD systems that switch abruptly between fixed resolution steps, this method enables fine-grained control over detail levels, eliminating popping artifacts and allowing the renderer to fall back to lower resolutions if rendering time exceeds allotted limits [S1, unofficial][S2, unofficial]. It is distinguished from its sibling [Scene Contraction Function](8533-scene-contraction-function.md) by its focus on precomputing a continuous spectrum of detail rather than geometric simplification via contraction.

## How it works
The method constructs a hierarchy where every level of detail is derived directly from the original high-resolution dataset, preventing the error accumulation seen in recursive approaches like 3D mipmaps [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** Original high-resolution dataset (volume or scene representation).
*   **Output:** A continuous-resolution representation (hierarchy) allowing interpolation of detail levels.

**Data Structures:**
The hierarchy is typically structured as a set of resolutions or a continuous parameter space. In Gaussian Splatting contexts, this may involve a continuous budget parameter $c$ that sweeps a memory-quality curve [S4, unofficial]. The data structure must support querying the appropriate resolution or Gaussian count based on a continuous LoD value.

**Steps:**
1.  **Low-Pass Filtering:** The input dataset is sampled and low-pass filtered to create lower-resolution versions [S1, unofficial].
2.  **Filter Support Mapping:** Establish a direct correspondence between the size of the low-pass filter support and the resulting resolution [S1, unofficial].
3.  **Hierarchy Construction:** Create every detail in the hierarchy by convolving the *original* dataset with a low-pass filter of the appropriate support [S1, unofficial]. This ensures that each level is an independent approximation of the original, not of the previous level.
4.  **Continuous Interpolation:** At runtime, interpolate between precomputed levels or adjust properties (e.g., Gaussian count) continuously based on the view-dependent or time-budget LoD parameter [S2, unofficial][S4, unofficial].

**Formulas and Parameters:**
*   **Filtering:** $D_{res} = \text{LowPassFilter}(D_{original}, \sigma)$, where $\sigma$ corresponds to the desired resolution [S1, unofficial].
*   **Error Propagation:** Errors from non-ideal filters do not accumulate because $D_{level_i}$ is computed from $D_{original}$, not $D_{level_{i-1}}$ [S1, unofficial].
*   **Continuous Budget ($c$):** In specific implementations like Budgeted-GS, a parameter $c$ controls the trade-off between memory and quality [S4, unofficial]. For example, $c=4$ may be lossless, while lower values reduce memory usage significantly (e.g., 110.6 MB vs 1926 MB at ~20.8 dB) [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_continuous_lod(original_dataset):
    """
    Builds a continuous LoD hierarchy by convolving the original dataset
    with low-pass filters of varying support sizes.
    """
    hierarchy = {}
    
    # Define a range of filter supports corresponding to desired resolutions
    # Note: Resolutions need not be powers of two [S1, unofficial]
    filter_supports = [1.0, 1.5, 2.0, 3.0, 4.0] 
    
    for support in filter_supports:
        # Convolve the ORIGINAL dataset, not the previous level
        # This prevents error accumulation [S1, unofficial]
        filtered_dataset = low_pass_filter(original_dataset, support)
        
        # Store with the support/resolution as key
        hierarchy[support] = filtered_dataset
        
    return hierarchy

def render_continuous_lod(hierarchy, lod_parameter):
    """
    Renders using continuous interpolation between precomputed levels.
    lod_parameter: A continuous value (e.g., memory budget or distance)
    mapping to a specific resolution in the hierarchy.
    """
    # Find the two bounding filter supports in the hierarchy
    # Interpolate between them to get the final dataset
    # Implementation detail: Specific interpolation method (e.g., linear)
    # depends on the data type (volume values, Gaussian properties) [S2, unofficial]
    
    # Example: Linear interpolation of Gaussian counts or volumes
    # This avoids popping artifacts [S2, unofficial]
    dataset = interpolate_datasets(hierarchy, lod_parameter)
    
    return render(dataset)
```

**Edge Cases and Pitfalls:**
*   **Filter Non-Ideality:** While errors do not accumulate, a non-ideal filter can still introduce artifacts at individual levels [S1, unofficial].
*   **Resolution Granularity:** The resolution difference between adjacent levels should be small to ensure smooth quality changes [S1, unofficial].
*   **Runtime Overhead:** Continuous LoD has higher runtime overhead and computation time compared to static LODs [S3, unofficial].

## Subtopics

### Continuous LOD Parameter (c)
A specific implementation detail found in Budgeted-GS where a continuous budget parameter $c$ is used to sweep a memory-quality curve [S4, unofficial]. This allows the same model to be rendered at different LOD levels without retraining [S4, unofficial]. Across the full ladder, a single model sweeps a continuous memory-quality curve; re-rendering through the GPS renderer reproduces it within 0.5 dB [S4, unofficial]. At an operating point of ~20.8 dB, the method costs 110.6 MB against GPS's full 1926 MB (17x reduction) [S4, unofficial]. The value $c=4$ is lossless under both renderers [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Higher computation time for precomputing the hierarchy compared to static LODs, as each level is derived from the original via convolution [S1, unofficial][S3, unofficial].
*   **Memory Complexity:** Depends on the number of precomputed levels or the continuous parameter range. Can be optimized via caching (e.g., reusing Gaussians from previous frames) [S2, unofficial].
*   **Performance:** Produces images with higher fidelity for a given polygon budget compared to static LODs [S3, unofficial]. Eliminates popping artifacts associated with discrete switching [S2, unofficial].
*   **Strengths:** Smooth quality transitions, no error accumulation between levels, flexible resolution (not limited to powers of two) [S1, unofficial][S2, unofficial].
*   **Limitations:** Higher runtime overhead, more complex to implement than discrete LOD, requires careful tuning of filter supports and interpolation mechanisms [S1, unofficial][S3, unofficial].
*   **When to Use:** When smooth visual transitions are critical, or when rendering time is variable and needs to be adaptively managed [S1, unofficial].
*   **When Not to Use:** When memory or precomputation time is extremely constrained, or when discrete, predictable LOD switching is sufficient [S3, unofficial].

## Background
The concept of hierarchical level-of-detail representation for 3D volumes was introduced in the context of time-critical volume rendering, where gradual elimination of high-frequency features allows fallback to lower resolutions if rendering exceeds allotted time [S1, unofficial]. The approach was highlighted for its robustness and applicability, particularly in avoiding the error propagation issues of 3D mipmaps [S1, unofficial]. Later, the concept was extended to continuous-resolution representations in out-of-core visualization systems [S3, unofficial]. In recent years, continuous LoD has been adapted for Gaussian Splatting, where it enables fine-grained control over Gaussian counts and memory usage without retraining [S2, unofficial][S4, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19990040407: Time-Critical Volume Rendering, https://archive.org/details/NASA_NTRS_Archive_19990040407, fetched 2026-10-26 (unofficial)
[S2] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S3] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
[S4] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
