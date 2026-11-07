# Dynamic Sampling Strategy
Also called: Shading-error-driven Dynamic Sampling
## Overview
A procedure to dynamically determine the number and locations of sample points in a participating medium to accurately reproduce shading variations and minimize rendering errors [S1]. It adapts sample locations based on shading error, unlike prior methods that sample based on density distribution [S1]. It builds on [Gradient-driven Sampling](5281-gradient-driven-sampling.md) by using a specific gradient-based error metric to drive a recursive splitting process [S1].
## How it works
**Inputs:**
- Volume geometry and optical properties (scattering/absorption coefficients).
- Light source positions and intensities.
- Camera parameters.
- Initial coarse set of sample points (or empty set for first iteration).

**Outputs:**
- Final set of sample points $\{x_j\}$ with associated source radiance $L_{x_j}$ and gradient $\nabla L_{x_j}$.
- Rendered image via ray marching using interpolated radiance.

**Data Structures:**
- Sample Point: A 3D position $x_j$, source radiance scalar $L_{x_j}$, and source radiance gradient vector $\nabla L_{x_j}$.
- Error Metric: A scalar value per sample or region indicating shading error magnitude.

**Steps:**
1. **Initial Sampling:** Generate a small initial set of sample points $\{x_j\}$ distributed in the medium. The distribution may be uniform or based on a coarse heuristic [S1].
2. **Radiance Evaluation:** At each sample point $x_j$, numerically evaluate the source radiance $L_{x_j}$ and its gradient $\nabla L_{x_j}$ using a volume ray tracer [S1].
3. **Interpolation:** Reconstruct source radiance $L_x$ at arbitrary points $x$ in the volume using gradient-based interpolation from the sample points [S1]. This interpolation is efficiently applied via sample splatting [S1].
4. **Error Estimation:** Compute the shading error for regions in the medium. Areas where the reconstructed source radiance results in significant shading errors are identified [S1].
5. **Dynamic Adjustment (Recursive Splitting):**
   - Identify sample points or regions with high shading error.
   - Recursively split these high-error samples by placing additional sample points in those areas [S1].
   - The splitting process replaces high-error samples with finer grid vertices or new sample points [S1].
   - Repeat steps 2-5 until the shading error is minimized or a computational budget is reached [S1].
6. **Rendering:** Perform volume ray marching to composite the final radiance along view rays using the interpolated source radiance from the final set of sample points [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_sampling_strategy(volume, lights, camera, max_iterations=5):
    # Initial coarse sampling
    samples = generate_initial_samples(volume)
    
    for iteration in range(max_iterations):
        # Evaluate radiance and gradient at current samples
        for sample in samples:
            sample.L = evaluate_source_radiance(sample.position, volume, lights)
            sample.grad_L = evaluate_gradient(sample.position, volume, lights)
            
        # Interpolate radiance across the volume
        # This step is implicit in the error calculation below
        # Interpolation uses gradient-based scheme (e.g., splatting)
        
        # Estimate shading error and identify high-error regions
        high_error_regions = estimate_shading_error(samples, volume, lights)
        
        # If no significant error, stop
        if not high_error_regions:
            break
            
        # Recursive sample splitting: add new samples in high-error areas
        new_samples = []
        for region in high_error_regions:
            # Split the region by adding new sample points
            # The exact splitting logic (e.g., grid refinement) is implementation-specific
            # but aims to minimize shading error [S1]
            new_samples.extend(split_region(region, samples))
            
        samples.extend(new_samples)
        
    return samples

def evaluate_source_radiance(position, volume, lights):
    # Numerical evaluation via volume ray tracing
    # Returns scalar radiance value
    pass

def evaluate_gradient(position, volume, lights):
    # Numerical evaluation of gradient of source radiance
    # Returns vector gradient
    pass

def estimate_shading_error(samples, volume, lights):
    # Compute error based on interpolation accuracy
    # Returns regions with error above threshold
    pass

def split_region(region, existing_samples):
    # Generate new sample points within the high-error region
    # Details of splitting (e.g., recursive grid subdivision) are not fully specified
    # but must adaptively place samples to reduce error [S1]
    pass
```

**Parameters:**
- `max_iterations`: Maximum number of refinement iterations. Not given in the sources.
- Error threshold: Threshold for determining "significant shading errors". Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Numerical evaluation of radiance and gradient must be accurate to avoid propagating errors into the interpolation [S1].
- **Boundaries:** Handling of volume boundaries during sample generation and splitting is not explicitly detailed [S1].
- **Degenerate Input:** If the medium is homogeneous, shading error may be low everywhere, leading to minimal sampling [S1].

## Subtopics

### [Dynamic Sampling with Recursive Sample Splitting](9091-dynamic-sampling-with-recursive-sample-splitting.md)
Uses a specific gradient-based error metric to drive a recursive splitting process that replaces high-error samples with finer grid vertices. Choose when accurate modeling of sharp shading variations (e.g., light shafts) is required.

## Operating characteristics
- **Time Complexity:** Depends on the number of sample points and iterations. Real-time performance is claimed for dynamic scenes [S1].
- **Memory Complexity:** Proportional to the number of sample points stored.
- **Performance:** Comparable to ray tracing in accuracy but faster due to adaptive sampling and interpolation [S1].
- **Strengths:** Captures all-frequency shading effects (glows, shadows, light shafts) in real-time without precomputation [S1].
- **Limitations:** Requires numerical evaluation of radiance and gradient at each sample point, which can be costly if not optimized [S1].
- **When to Use:** Real-time rendering of inhomogeneous, single-scattering media with sharp shading variations [S1].
- **When Not to Use:** When precomputed light transport is available or when media are homogeneous [S1].
## Background
Originated in the paper "Gradient-based Interpolation and Sampling for Real-time Rendering of Inhomogeneous, Single-scattering Media" (2008) [S1]. It was developed to address artifacts in previous methods that sampled based on density distribution, aiming for real-time rendering of dynamic lighting and scenes [S1].
## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
