# BRDF Evaluation Kernel


## Overview
The BRDF Evaluation Kernel evaluates a Bidirectional Reflectance Distribution Function (BRDF) for a batch of rays during inference ray marching and integration [S1, unofficial]. It transforms geometric and material inputs into final pixel colors [S1, unofficial]. It is distinct from compositing operations, focusing solely on the local reflection model evaluation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- `rgba`: Input RGBA tensor (material color/opacity data) [S1, unofficial].
- `position`: Input position tensor (3D coordinates) [S1, unofficial].
- `gradient`: Input gradient tensor (normals or density gradients) [S1, unofficial].
- `rayDir`: Input ray direction tensor [S1, unofficial].
- `virtual_size`: Dimensions of the virtual batch grid [S1, unofficial].
- `rgbaOutput`: Output RGBA tensor containing the computed color [S1, unofficial].

**Data Structures:**
The kernel operates on flat tensors indexed by a linear batch index `b` [S1, unofficial]. The layout assumes contiguous memory access for the input tensors and the output tensor [S1, unofficial].

**Steps:**
1. Initialize the loop over the batch using `KERNEL_1D_LOOP` with bounds determined by `virtual_size.x * virtual_size.y * virtual_size.z` [S1, unofficial].
2. For each batch index `b`:
   a. Fetch `rgba`, `position`, `gradient`, and `rayDir` from their respective input tensors at index `b` [S1, unofficial].
   b. Set `batch = 0` [S1, unofficial].
   c. Call `brdf.eval(rgba, position, gradient, rayDir, batch)` to compute the reflected color [S1, unofficial].
   d. Write the resulting color to `rgbaOutput` at index `b` [S1, unofficial].

**Formulas:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def brdf_eval_kernel(rgba, position, gradient, rayDir, virtual_size, rgbaOutput):
    total_elements = virtual_size[0] * virtual_size[1] * virtual_size[2]
    for b in range(total_elements):
        current_rgba = rgba[b]
        current_position = position[b]
        current_gradient = gradient[b]
        current_rayDir = rayDir[b]
        batch = 0
        # The source does not provide the internal BRDF formula [S1, unofficial].
        color = brdf_eval(current_rgba, current_position, current_gradient, current_rayDir, batch)
        rgbaOutput[b] = color

def brdf_eval(rgba, position, gradient, rayDir, batch):
    # The source does not provide the internal BRDF formula [S1, unofficial].
    return rgba
```

**Parameters:**
- `virtual_size`: Defines the batch dimension [S1, unofficial]. Typical values depend on the resolution of the volume and the number of rays processed per frame. Not given in the sources.

**Edge Cases and Pitfalls:**
- The kernel assumes valid inputs for all tensors [S1, unofficial]. Degenerate inputs (e.g., zero-length gradients) are not explicitly handled in the source description [S1, unofficial].
- Precision issues may arise if the evaluation function involves complex trigonometric or normalization operations on the gradient and ray direction vectors [G].

## Subtopics

### EPBRR
EPBRR is a variant proposed by Poirier-Ginter et al. [2025] as an improvement to 3DGRT's ray tracer [S2]. It introduces various optimizations to the ray tracing process [S2]. However, it used a crude custom implementation of Depth Importance (DI), summarized as "binning the depth maps" [S2]. Experiments in EDGS argued against this specific DI approach [S2]. Furthermore, EPBRR techniques were not tested against proper dense initialization and were not included in a full ray-tracing solution [S2]. They offered only limited comparisons to 3DGRT, which excluded DI and trained at reduced iteration counts without Spherical Harmonics (SH) [S2]. The method was not validated individually [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.

## Background
The EPBRR variant was proposed by Poirier-Ginter et al. [2025] as an improvement to 3DGRT [S2]. No further background details are provided in the sources.

## Sources
[S1] shamanDevel/fV-SRN, https://github.com/shamanDevel/fV-SRN, fetched 2026-10-26 (unofficial)
[S2] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
