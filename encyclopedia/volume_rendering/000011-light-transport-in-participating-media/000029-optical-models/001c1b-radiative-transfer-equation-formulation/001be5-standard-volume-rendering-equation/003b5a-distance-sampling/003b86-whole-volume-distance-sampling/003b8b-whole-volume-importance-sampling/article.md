# Whole Volume Importance Sampling

Also called: Geometry-Dependent Transmittance Sampling, Importance Sampling Scheme

## Overview
Whole Volume Importance Sampling is a Monte Carlo strategy for volume rendering that samples the integration domain according to the geometry-dependent transmittance of the entire participating medium, rather than limiting sampling to the transmittance along a single ray. It solves the high-variance noise inherent in standard distance sampling by incorporating both camera and light ray geometries into the probability density function (PDF), resulting in superior convergence for finite media [S1]. It differs from classic techniques by accounting for the global geometry of the medium, not just local transmittance [S1].

## How it works
The method constructs a sampling distribution that matches the geometry-dependent transmittance along a camera ray. It is applicable in path tracing frameworks where multiple sampling strategies can be combined [S1].

**Inputs:**
*   Camera ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   Light source geometry and radiance.
*   Medium geometry (typically a convex bounding box or defined volume).
*   Medium optical properties (extinction coefficient $\sigma_t$).

**Outputs:**
*   Sampled positions and weights for Monte Carlo integration of the radiative transfer equation.

**Data Structures:**
The medium geometry is decomposed into trapezoid-shaped segments. These segments are defined by the intersection of the camera ray and the light ray with the medium boundaries. To avoid repeated traversal, variables describing each segment are cached [S1].

**Steps:**
1.  **Geometry Traversal:** Traverse the edges of the medium bounding box from corner to corner to identify incident and outgoing exit points for both camera and light rays. This decomposes the medium into trapezoid-shaped segments [S1].
2.  **Segment Selection:** Select a segment based on a probability distribution proportional to the geometry-dependent transmittance of that segment. The transmittance is calculated based on the path length through the segment and the extinction coefficient [S1].
3.  **Sample Generation:** Generate a sample position within the selected segment. The position is sampled according to the local transmittance distribution within the segment [S1].
4.  **Weight Calculation:** Compute the importance sampling weight as the ratio of the integrand (geometry-dependent transmittance) to the PDF of the selected segment and position [S1].

**Formulas:**
The geometry-dependent transmittance $T(\mathbf{x})$ along a ray is defined as:
$$ T(\mathbf{x}) = \exp\left(-\int_{\mathbf{o}}^{\mathbf{x}} \sigma_t(\mathbf{u}) d\mathbf{u}\right) $$
where $\sigma_t$ is the extinction coefficient [S1].

The PDF $p(\mathbf{x})$ for sampling a point $\mathbf{x}$ is proportional to the geometry-dependent transmittance:
$$ p(\mathbf{x}) \propto T(\mathbf{x}) \cdot J(\mathbf{x}) $$
where $J(\mathbf{x})$ represents the geometric Jacobian or solid angle factor depending on the specific integrator formulation [S1].

**Parameters:**
*   $\sigma_t$: Extinction coefficient. Typical values depend on the medium density.
*   Number of segments: Determined by the complexity of the medium geometry.

**Edge Cases and Pitfalls:**
*   **Degenerate Geometry:** If the camera or light ray does not intersect the medium, the transmittance is 1, and no scattering occurs.
*   **Precision:** High extinction coefficients can lead to underflow in transmittance calculations.
*   **Caching:** Failure to cache segment variables leads to repeated traversal, increasing computational cost [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def whole_volume_importance_sampling(camera_ray, light_ray, medium, sigma_t):
    # Step 1: Geometry Traversal
    # Decompose medium into trapezoid-shaped segments by traversing edges
    # Connect incident and outgoing exit points from the medium
    segments = decompose_medium_into_trapezoids(camera_ray, light_ray, medium)
    
    # Cache segment variables to avoid repeated traversal
    cached_segments = cache_segment_variables(segments)
    
    # Step 2: Segment Selection
    # Calculate geometry-dependent transmittance for each segment
    segment_transmittances = []
    for seg in cached_segments:
        # Calculate transmittance based on path length and sigma_t
        path_length = seg.length()
        transmittance = exp(-sigma_t * path_length)
        segment_transmittances.append(transmittance)
    
    # Normalize transmittances to form a PDF
    total_transmittance = sum(segment_transmittances)
    if total_transmittance == 0:
        return None # No intersection or zero transmittance
    
    pdfs = [t / total_transmittance for t in segment_transmittances]
    
    # Select a segment based on the PDF
    segment_index = sample_categorical(pdfs)
    selected_segment = cached_segments[segment_index]
    
    # Step 3: Sample Generation
    # Sample a position within the selected segment
    # The position is sampled according to the local transmittance distribution
    sample_position = sample_within_segment(selected_segment, sigma_t)
    
    # Step 4: Weight Calculation
    # Compute the importance sampling weight
    # Weight = Integrand / PDF
    integrand = segment_transmittances[segment_index] * local_transmittance_factor(sample_position, sigma_t)
    pdf_value = pdfs[segment_index] * local_pdf_factor(sample_position, selected_segment)
    
    weight = integrand / pdf_value
    
    return sample_position, weight
```

## Subtopics

### [Importance Sampling for Forward Scattering Propagator](10151-importance-sampling-for-forward-scattering-propagator.md)
This method provides a Monte Carlo path tracing strategy to evaluate the rendering equation for the forward scattering propagator $G_f^b$, specifically addressing the transport of light through participating media where forward scattering dominates. It differs from standard local importance sampling (which samples phase functions and exponential depth independently at each vertex) by constructing a 7-dimensional global sampling strategy that incorporates the analytical structure of the propagator, resulting in faster convergence for infinite-dimensional scattering problems.

## Operating characteristics
*   **Time Complexity:** Depends on the number of segments and the cost of geometry traversal. Caching segment variables reduces repeated traversal costs [S1].
*   **Memory Complexity:** Proportional to the number of segments stored in the cache [S1].
*   **Measured Performance:** The method yields visible noise reduction and superior convergence compared to independent sampling with existing algorithms and ray marching/distance sampling methods [S1]. It requires only a single sample in unoccluded areas to compute the whole integral solution [S1].
*   **Strengths:** Lower variance images, superior convergence, applicable to multiple light sources, directly applicable to indoor scene rendering in room-scale interactive experiences [S1].
*   **Limitations:** Requires decomposition of medium geometry into segments, which may be complex for non-convex or highly detailed media [S1].
*   **When to use:** Path tracing frameworks where existing approaches enable combining multiple sampling strategies, especially for finite media with significant geometry-dependent transmittance variations [S1].
*   **When not to use:** Simple homogeneous media where standard distance sampling is sufficient, or when real-time performance is critical and caching overhead is prohibitive [S1].

## Background
The method was introduced in the paper "Collimated Whole Volume Light Scattering in Homogeneous Finite Media" [S1]. It is part of a broader effort to optimize Monte Carlo integration for volume rendering, specifically addressing the limitations of classic techniques that limit sampling to transmittance along a single ray [S1]. The technique generalizes to multiple scattering and progressive rendering [S1].

## Sources
[S1] Collimated Whole Volume Light Scattering in Homogeneous Finite Media, https://doi.org/10.1109/tvcg.2021.3135764, fetched 2026-10-27
