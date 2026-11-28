# Multiscale Feature Orbit Extraction via Hyper-iso-surface Intersection

Also called: Feature tracking in scale space, Orbit extraction

## Overview
This method extracts the trajectories (orbits) of specific features, such as curvature extrema, across a scale space by computing the intersection of two hyper-iso-surfaces: one representing intensity levels and the other representing scaled differential descriptors [S1, unofficial]. It differs from standard isosurface extraction by operating in $(d+1)$ dimensions (space plus scale) to track feature evolution, and from the parent [Marching Lines Algorithm](942-marching-lines-algorithm.md) by specifically targeting feature orbits in scale space rather than general curve intersections in 3D [S1, unofficial].

## How it works
**Inputs:**
*   A primal 3D image $I(x, y, z)$.
*   Scale space parameters: initial variance $\varepsilon$, number of scale samples $n$, and variance change factor $\tau$ [S1, unofficial].

**Outputs:**
*   A set of feature orbits, represented as sequences of 3D points $(x, y, z)$ indexed by scale level $i$, where each point is an intersection of the intensity hyper-iso-surface and the zero-hyper-surface [S1, unofficial].

**Data Structures:**
*   **Multiscale Extension Image:** A 4D array $M[x, y, z, i]$ where $i \in [0, n-1]$. Each slice $i$ is the primal image convolved with a Gaussian kernel of variance $\sigma_i$.
*   **Scaled Differential Descriptor Image:** A 4D array $D[x, y, z, i]$. Each slice $i$ is the primal image convolved with derivatives of the Gaussian kernel of variance $\sigma_i$.
*   **Scale Axis:** Discrete levels $i=0 \dots n-1$.

**Parameters:**
*   $\varepsilon$: Initial variance (smallest scale) [S1, unofficial].
*   $n$: Number of samples at the scale axis [S1, unofficial].
*   $\tau$: Change of variance factor [S1, unofficial].
*   Variance at level $i$: $\sigma_i = \varepsilon \cdot \tau^i$ [S1, unofficial].

**Steps:**

1.  **Generate Scale Space:**
    For each scale level $i$ from $0$ to $n-1$:
    a. Calculate variance $\sigma_i = \varepsilon \cdot \tau^i$.
    b. Compute the multiscale extension image slice $M[\cdot, \cdot, \cdot, i]$ by convolving the primal image $I$ with a Gaussian kernel $G_{\sigma_i}$ [S1, unofficial].
    c. Compute the scaled differential descriptor image slice $D[\cdot, \cdot, \cdot, i]$ by convolving the primal image $I$ with the appropriate derivatives of the Gaussian kernel $G_{\sigma_i}$ (e.g., for curvature extrema, second-order derivatives) [S1, unofficial].

2.  **Extract Hyper-iso-surfaces:**
    a. **Intensity Hyper-iso-surface:** Extract the surface from $M$ where intensity equals a constant $I_{const}$. This surface represents the evolution of iso-intensity regions across scale [S1, unofficial].
    b. **Zero-hyper-surface:** Extract the surface from $D$ where the differential descriptor equals zero. This surface represents the loci of singularities (e.g., curvature extrema) across scale [S1, unofficial].

3.  **Compute Intersection (Orbits):**
    Compute the intersection of the Intensity Hyper-iso-surface and the Zero-hyper-surface. The resulting curves are the feature orbits [S1, unofficial].

**Alternative Efficient Method:**
To avoid pre-calculating all scaled images and extensive hyper-surface extraction, use the following approach [S1, unofficial]:
1.  Pre-calculate only the first two levels of scaled images ($i=0, 1$).
2.  Find starting line segments by intersecting hyper-iso-surfaces at these initial levels.
3.  Extract the rest of the segments using the orientation of the previous segments, processing only voxels in the reached region. This leverages the simplification property of scale space where sharp curvatures are attenuated [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_feature_orbits(primal_image, epsilon, n, tau, intensity_const):
    """
    Extract feature orbits using the alternative efficient method.
    """
    # Step 1: Pre-calculate first two levels
    sigma_0 = epsilon
    sigma_1 = epsilon * tau
    
    # Compute Gaussian and its derivatives for level 0
    M_0 = convolve_gaussian(primal_image, sigma_0)
    D_0 = convolve_gaussian_derivatives(primal_image, sigma_0)
    
    # Compute Gaussian and its derivatives for level 1
    M_1 = convolve_gaussian(primal_image, sigma_1)
    D_1 = convolve_gaussian_derivatives(primal_image, sigma_1)
    
    # Step 2: Find starting line segments at level 0 and 1
    # Intersect intensity hyper-iso-surface (M = intensity_const) 
    # with zero-hyper-surface (D = 0)
    starting_segments = intersect_hyper_isosurfaces(M_0, D_0, intensity_const, 0)
    # Extend to level 1 if necessary for continuity
    segments = extend_segments_to_level(starting_segments, M_1, D_1, intensity_const, 1)
    
    # Step 3: Extract rest of segments using orientation
    for i in range(2, n):
        sigma_i = epsilon * (tau ** i)
        M_i = convolve_gaussian(primal_image, sigma_i)
        D_i = convolve_gaussian_derivatives(primal_image, sigma_i)
        
        # Use orientation from previous level to guide search in reached region
        segments = propagate_segments(segments, M_i, D_i, intensity_const, i)
        
    return segments

def convolve_gaussian(image, sigma):
    """Convolve image with Gaussian kernel of variance sigma."""
    # Implementation detail: standard Gaussian blur
    kernel = create_gaussian_kernel(sigma)
    return apply_convolution(image, kernel)

def convolve_gaussian_derivatives(image, sigma):
    """Convolve image with derivatives of Gaussian kernel of variance sigma."""
    # Implementation detail: e.g., Laplacian for curvature extrema
    kernel = create_gaussian_derivative_kernel(sigma)
    return apply_convolution(image, kernel)

def intersect_hyper_isosurfaces(M, D, intensity_const, level):
    """
    Find intersection points/segments where M == intensity_const and D == 0.
    Returns list of segments (list of points) at the given level.
    """
    # Implementation detail: Use Marching Lines or similar 3D curve extraction
    # on the 4D data (x,y,z,level) restricted to the current level slice
    # or between levels if tracking continuity.
    # For a single level, find voxels where M approx intensity_const and D approx 0.
    # Refine to exact intersection using linear interpolation.
    segments = []
    # ... (Algorithm to find 1D curves in 3D grid satisfying two constraints)
    return segments

def extend_segments_to_level(segments, M, D, intensity_const, level):
    """Extend existing segments to the new level using orientation."""
    # Implementation detail: Predict next position based on previous segment direction
    # and refine using local intersection in M and D at the new level.
    extended = []
    for seg in segments:
        next_point = predict_next_point(seg, level)
        refined_point = refine_intersection(M, D, next_point, intensity_const)
        extended.append([seg[-1], refined_point])
    return extended

def propagate_segments(segments, M, D, intensity_const, level):
    """Propagate segments to the next level using orientation and reached region."""
    # Implementation detail: Only process voxels near the end of existing segments
    # to leverage scale space simplification.
    propagated = []
    for seg in segments:
        last_point = seg[-1]
        # Search neighborhood of last_point in M and D
        candidates = find_candidates_in_neighborhood(M, D, last_point, intensity_const)
        for cand in candidates:
            propagated.append(seg + [cand])
    return propagated
```

**Edge Cases and Pitfalls:**
*   **Bifurcations:** As resolution increases (scale decreases), a simple feature can turn into several features. The intersection method must handle branching orbits [S1, unofficial].
*   **Precision:** Intersecting two surfaces in 3D space at each scale level requires precise numerical methods to avoid drift or gaps in orbits [S1, unofficial].
*   **Boundary Conditions:** Features may enter or leave the domain at different scales. The algorithm must handle boundary voxels correctly [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the method. The naive approach (pre-calculate all scales, extract all hyper-surfaces, intersect) is expensive. The alternative method reduces complexity by leveraging scale space simplification and only processing reached regions [S1, unofficial].
*   **Memory Complexity:** The naive approach requires storing all multiscale images and differential descriptors ($O(n \cdot W \cdot H \cdot D)$). The alternative method reduces this by only storing current and previous levels or recomputing as needed [S1, unofficial].
*   **Strengths:** Explicitly tracks feature evolution across scale; handles bifurcations; leverages scale space properties for efficiency [S1, unofficial].
*   **Limitations:** Requires careful parameter tuning ($\varepsilon, \tau, n$); sensitive to noise in differential descriptors; complex implementation of intersection in 3D [S1, unofficial].
*   **When to use:** When analyzing the evolution of specific features (e.g., curvature extrema) across different resolutions is required [S1, unofficial].
*   **When not to use:** When only static isosurfaces at a single scale are needed; when computational resources are extremely limited and the naive approach is too slow [S1, unofficial].

## Background
This method was proposed in the context of iso-surface extraction in $n$-dimensions, specifically applied to tracking feature curves across scale [S1, unofficial]. It builds on the concept that iso-surfaces at different scales are equivalent to hyper-iso-surfaces in a multiscale image [S1, unofficial]. The work highlights the limitations of separate computation of multiscale images and hyper-iso-surfaces when only feature orbits are needed, proposing the alternative efficient method to reduce unnecessary effort [S1, unofficial].

## Sources
[S1] Iso-surface Extraction in n-D applied to Tracking Feature Curves across Scale, https://inria.hal.science/inria-00615083v1, fetched 2026-10-27 (unofficial)
