# Adaptive Sampling (Sparse Grid)
Also called: Sparse Grid Adaptive Sampling
## Overview
Adaptive Sampling (Sparse Grid) is a rendering optimization that exploits coherence in projections by casting a sparse grid of rays (less than one per pixel) and adaptively increasing the number of rays in regions of high image complexity [S1, unofficial]. It generates a sequence of successively more refined images by iteratively adding samples and resampling [S1, unofficial].
## How it works
**Inputs:** Volumetric dataset; display resolution; initial sparse grid configuration [G].
**Outputs:** Final rendered image; intermediate sequence of refined images [G].

**Data Structures:**
*   **Sample Array:** A non-uniform array storing color values at specific ray intersection points. Layout: List of `(x, y, color)` tuples, where `x, y` are pixel coordinates and `color` is the accumulated radiance [S1, unofficial].

**Steps:**
1.  **Initial Sparse Casting:** Cast a sparse grid of rays through the volume, with a density of less than one ray per pixel [S1, unofficial].
2.  **Interpolation:** Form an initial image from the resulting non-uniform array of sample colors by interpolating and resampling to the display resolution [S1, unofficial].
3.  **Complexity Analysis:** Identify regions of high image complexity in the interpolated image [S1, unofficial].
4.  **Adaptive Refinement:** Cast additional rays in the identified high-complexity regions and add the resulting colors to the sample array [S1, unofficial].
5.  **Resampling:** Repeat the interpolation and resampling steps to generate a sequence of successively more refined images [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_sparse_grid_render(volume, display_res, initial_density, max_iterations):
    sample_array = []  # Stores (x, y, color) tuples
    image = [None] * (display_res[0] * display_res[1])
    
    # Step 1: Cast a sparse grid of rays (less than one per pixel)
    step = max(1, int(1.0 / initial_density))
    for y in range(0, display_res[1], step):
        for x in range(0, display_res[0], step):
            color = cast_ray(volume, x, y)
            sample_array.append((x, y, color))
    
    # Step 2: Form image from non-uniform array by interpolation and resampling
    image = resample_to_display(image, sample_array, display_res)
    
    # Steps 3-5: Iteratively refine
    for _ in range(max_iterations):
        # Step 3: Identify high-complexity regions in the interpolated image
        complex_regions = find_high_complexity_regions(image, display_res)
        
        # Step 4: Cast additional rays in complex regions and add to sample array
        for y, x in complex_regions:
            if (x, y) not in sampled_set:
                color = cast_ray(volume, x, y)
                sample_array.append((x, y, color))
                sampled_set.add((x, y))
        
        # Step 5: Repeat interpolation and resampling
        image = resample_to_display(image, sample_array, display_res)
                
    return image

def cast_ray(volume, x, y):
    # Cast ray through volume along pixel coordinates (x, y)
    # Returns accumulated radiance [G]
    return (0.0, 0.0, 0.0)

def resample_to_display(image, samples, res):
    # Interpolate non-uniform sample colors to full display resolution
    # Fills grid using nearest-neighbor or bilinear interpolation [G]
    for x, y, color in samples:
        image[y * res[0] + x] = color
    return image

def find_high_complexity_regions(image, res):
    # Identify regions with high frequency or sharp edges
    # Returns list of (x, y) coordinates requiring refinement [G]
    return [(x, y) for y in range(1, res[1]-1) for x in range(1, res[0]-1)]

sampled_set = set()
```

**Parameters:**
*   `initial_density`: Ratio of rays to pixels. Must be $< 1$ [S1, unofficial].
*   `max_iterations`: Controls the number of refinement cycles [G].

**Edge Cases:**
*   **Small Polygons:** If integrated with adaptive supersampling in a hybrid ray tracer, missing small polygons occurs if the initial sampling rate is too low [S1, unofficial].
*   **Blurring/Coherence:** The method is particularly effective for data acquired from sensing devices where the acquisition process induces considerable blurring, as coherence is higher [S1, unofficial].
## Subtopics

### [Adaptive Point Insertion Strategy](3719-adaptive-point-insertion-strategy.md)
Optimizes point distribution in signed distance field reconstruction by prioritizing topologically critical regions. It sets itself apart from uniform sampling by using quality-driven criteria to strategically add points, directly correlating reconstruction quality with the proportion of adaptively inserted samples. Choose it when topological preservation is critical under a constrained sampling budget.

## Operating characteristics
**Time Complexity:** Grows linearly with image size, logarithmically with ray length, and linearly with depth complexity [S1, unofficial]. Adaptive termination reduces dependence on depth complexity, becoming nearly linear with image size for opaque environments [S1, unofficial].
**Memory Complexity:** Scales linearly with the number of samples stored in the array [G].
**Performance:** Achieves savings of more than two orders of magnitude over brute-force methods [S1, unofficial]. Generates crude images in a few seconds, with high-quality images in less than a minute [S1, unofficial].
**Strengths:** Exploits coherence in projections, particularly effective for data from sensing devices with acquisition-induced blurring [S1, unofficial]. Supports progressive refinement.
**Limitations:** Risk of missing small polygons if the initial sampling rate is too low, especially in hybrid ray tracers with adaptive supersampling [S1, unofficial].
## Background
Introduced by Michael Levoy in 1988 as an optimization for rendering mixtures of geometric and volumetric data [S1, unofficial]. The method was developed to optimize ray tracing by leveraging coherence in projections, particularly for data from sensing devices [S1, unofficial]. It is well-suited for scan-conversion methods where polygons are filtered prior to ray tracing [S1, unofficial].
## Sources
[S1] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
