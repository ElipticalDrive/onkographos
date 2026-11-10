# Adaptive Sampling (Sparse Grid)
Also called: Sparse Grid Adaptive Sampling
## Overview
Adaptive Sampling (Sparse Grid) is a rendering optimization that exploits coherence in projections—often introduced by sensing device blurring—to reduce computational overhead by casting a sparse grid of rays (less than one per pixel) and adaptively increasing ray density in regions of high image complexity [S1, unofficial]. It sets itself apart from uniform sampling by decoupling the initial sampling rate from the display resolution, relying on interpolation and resampling to form the final image, which makes it particularly suitable for scan-conversion methods where polygons are filtered prior to ray tracing [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Volumetric or geometric data; initial sparse grid configuration (typically < 1 ray/pixel); display resolution.
- **Output:** A rendered image at display resolution; optionally, a sequence of successively refined images.

**Data Structures:**
- **Sample Array:** A non-uniform array storing sample colors corresponding to the cast rays. The layout is not fixed to a grid; it contains entries for each ray origin and its resulting color value.
- **Ray Casters:** Functions to trace rays through the volume or against geometry.

**Steps:**
1. **Initial Sparse Casting:** Cast a sparse grid of rays through the scene. The density is intentionally low, often less than one ray per pixel [S1, unofficial].
2. **Color Sampling:** For each ray, compute the color contribution (integration through volume or shading of geometry) and store the result in the Sample Array [S1, unofficial].
3. **Interpolation and Resampling:** Form the image from the non-uniform array of sample colors by interpolating the sparse samples to the full display resolution [S1, unofficial].
4. **Adaptive Refinement (Iterative):**
   - Identify regions of high image complexity (e.g., high variance in color or gradient).
   - Cast additional rays in these regions, adding the resulting colors to the existing Sample Array [S1, unofficial].
   - Repeat the interpolation and resampling steps to generate a sequence of successively more refined images [S1, unofficial].

**Parameters:**
- **Initial Sampling Rate:** Must be low enough to be "sparse" (< 1 ray/pixel) but high enough to avoid missing small features [S1, unofficial].
- **Refinement Threshold:** Criteria for determining "high complexity" regions (not explicitly defined in sources, but implied to be based on image quality metrics) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Missing Small Polygons:** When integrated with adaptive supersampling in a hybrid ray tracer, there is a risk of missing small polygons if the initial sampling rate is too low [S1, unofficial].
- **Blurring Dependency:** The method relies on coherence present in projections, which is particularly true for data acquired from sensing devices that introduce considerable blurring [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_sparse_grid_rendering(scene, display_resolution):
    # Initialize empty sample array
    samples = []
    
    # Step 1: Initial sparse casting
    # Cast rays in a sparse grid pattern (density < 1 ray/pixel)
    initial_rays = generate_sparse_grid_rays(scene, display_resolution, density=0.5)
    
    # Step 2: Color sampling
    for ray in initial_rays:
        color = trace_ray(scene, ray)
        samples.append({'ray': ray, 'color': color})
        
    # Step 3: Interpolation and resampling
    image = interpolate_and_resample(samples, display_resolution)
    
    # Step 4: Adaptive refinement loop
    # Continue refining until convergence or time limit
    while not convergence_criteria_met(image):
        # Identify high-complexity regions
        complex_regions = identify_complex_regions(image, samples)
        
        # Cast more rays in complex regions
        new_rays = generate_rays_in_regions(scene, complex_regions, count=10)
        
        # Add new samples
        for ray in new_rays:
            color = trace_ray(scene, ray)
            samples.append({'ray': ray, 'color': color})
            
        # Re-render
        image = interpolate_and_resample(samples, display_resolution)
        
    return image
```

## Subtopics

### [Adaptive Point Insertion Strategy](3719-adaptive-point-insertion-strategy.md)
A strategy that intelligently prioritizes topologically critical regions to optimize computational resources, improving topological fidelity compared to uniform sampling. It sets itself apart by explicitly combining sample points with their surface projections to enhance detail preservation in high-genus or complex geometric models, where standard uniform or even progressive refinement methods often fail to capture intricate features like chair backs or surface cavities.

## Operating characteristics
**Time and Memory Complexity:**
- The cost of ray tracing volumetric data using hierarchical enumeration grows linearly with the size of the image, logarithmically with the length of each ray, and linearly with the depth complexity of the data [S1, unofficial].
- Adaptive termination of ray tracing reduces the dependence on depth complexity, and for opaque objects, eliminates this dependence entirely, causing the cost to grow nearly linearly with the size of the image [S1, unofficial].
- Savings of more than two orders of magnitude over brute-force rendering methods have been obtained for many datasets [S1, unofficial].

**Measured Performance:**
- Implemented on a Sun 4/280 with 32MB of main memory [S1, unofficial].
- Crude images can be obtained in a few seconds, followed by gradually better images at intervals of a few seconds each, culminating in a high-quality image in less than a minute [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Significant performance gains for coherent data; allows progressive refinement; well-suited for scan-conversion methods [S1, unofficial].
- **Limitations:** Risk of missing small polygons if initial sampling is too low; relies on projection coherence (blurring) [S1, unofficial].

**When to Use:**
- Use when data exhibits coherence in projections (e.g., from sensing devices) and when progressive refinement is desired [S1, unofficial].
- Avoid when the scene contains small, high-frequency geometric details that may be missed by sparse initial sampling [S1, unofficial].

## Background
Originating from Michael Levoy's work in 1988, this method was introduced as an optimization to take advantage of coherence in projections [S1, unofficial]. It was part of a suite of optimizations for rendering mixtures of geometric and volumetric data, aiming to reduce the computational cost of ray tracing volumetric data [S1, unofficial].

## Sources
[S1] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
