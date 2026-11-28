# Light Ray Extinction Optimization


## Overview
Light Ray Extinction Optimization accelerates direct illumination computation in Monte Carlo volume rendering by estimating extinction values between a scatter vertex $y$ and a point on the light source [S1]. While Optimized Ray Casting focuses on visibility along camera rays, this method extends the optimization to light transport paths, reducing variance in shadowed or complex volumetric regions [S1]. It differs from camera-ray optimization because stochastic scattering locations prevent data structure reuse across frames, requiring a full rebuild for each set of light rays [S1].

## How it works
**Inputs:** scattering vertex $y$, light source position, volume density field [G], and the visibility optimization data structure [S1].
**Outputs:** optimized extinction coefficient $\mu'_*(y)$ and transmittance estimates between the scattering vertex and the light source [S1].
**Data Structures:** The method reuses the hierarchical data structure from camera-ray visibility optimization [S1]. This structure maintains segments representing extinction intervals and supports `insert`, `merge`, and `evaluate` operations [S1].

**Algorithm Steps:**
1. For each scattering vertex $y$, cast a ray toward the light source [S1].
2. Draw a fixed number of stratified samples along the light ray segment [S1].
3. Insert the stratified samples into the data structure where merging operations take place [S1]. The structure is built from scratch for each set of rays because scattering locations change stochastically [S1]. Splitting segments or discarding samples is only required for progressive updates and can be omitted for standard per-pass builds [S1].
4. During transmittance estimation, evaluate the data structure to compute the optimized extinction coefficient $\mu'_*(y)$ at the current position [S1].
5. Use $\mu'_*(y)$ to compute decision probabilities for the delta tracking algorithm along the light ray [S1].
6. If multiple scattering is supported, sample additional rays at new scattering locations according to the phase function and apply the same optimization [S1].
7. In implementations using ratio tracking [NSJ14], multiple vertices are produced during transmittance estimation, each requiring an optimized extinction query [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_light_ray_structure(scattering_vertex, light_source_pos, num_samples=200):
    # 1. Define ray from scattering vertex to light source
    ray_dir = normalize(light_source_pos - scattering_vertex)
    ray_len = distance(scattering_vertex, light_source_pos)
    
    # 2. Generate stratified samples along the ray [G]
    samples = [scattering_vertex + ray_dir * (i + 0.5) * ray_len / num_samples for i in range(num_samples)]
    
    # 3. Evaluate raw extinction at sample positions [G]
    # Standard volume density query at each sample position
    
    # 4. Build data structure from scratch [S1]
    # Note: Splitting or discarding samples is only required for progressive updates [S1]
    # Hierarchical segment structure initialization [G]
    structure = []
    for pos in samples:
        structure.append((pos, get_density(pos)))
    # Merge adjacent segments [G]
    structure = merge_segments(structure)
    return structure

def query_optimized_extinction(position, structure):
    # Evaluate structure to compute optimized extinction coefficient mu'_*(y) [S1]
    # Standard segment lookup [G]
    return lookup_extinction(position, structure)
```

**Parameters:**
*   **Number of samples:** 200 samples per light ray is recommended as a good trade-off between accuracy and speed [S1].
*   **Progressive Updates:** Splitting of segments or discarding of samples is optional and only required for progressive rendering updates [S1].

**Edge Cases and Pitfalls:**
*   **Stochastic Scattering Locations:** Because scattering vertices are determined stochastically, the data structure cannot be reused across frames and must be rebuilt from scratch for each set of light rays [S1].
*   **Long Transport Paths:** Optimizing extinction along very long transport paths (e.g., multiple scattering events far from the camera) is computationally costly and yields no visual improvement [S1].
*   **Memory Allocation:** The method requires a static allocation for light ray optimization independent of resolution (e.g., 156MB) plus per-pixel allocations for structure sizes and offsets [S1].

## Subtopics

### Sample Reduction via Visibility Observations
A method to reduce sampling operations by applying two observations: only visible samples receive illumination, and only samples visible from the light source need to be illuminated [S2, unofficial]. It determines the interval of samples for illumination computation defined by $x_l$ (where visibility becomes zero) and $x_f$ (first point hit in camera space) [S2, unofficial]. It connects $x_f$ and $x_l$ through line segments and projects them into light space to determine the minimal set of entry points for light view rays [S2, unofficial]. It casts light rays only from these optimized entry points to ensure only samples visible from the camera are illuminated [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The cost is dominated by building the data structure for each light ray [S1]. On the HEPTANE dataset with 5000 samples per pixel, render times dropped from 51.7s for the reference to 21.47s, 27.17s, 28.2s, and 37s for varying numbers of optimization samples [S1].
*   **Memory Complexity:** Requires a static allocation independent of resolution (e.g., 156MB) plus per-pixel allocations for the data structure [S1].
*   **Strengths:** Significantly reduces variance in direct illumination, especially in shadowed regions or complex volumetric scenes [S1].
*   **Limitations:** Cannot be easily integrated into progressive rendering due to the stochastic nature of scattering locations, and optimizing for multiple scattering events far from the camera is inefficient [S1].
*   **When to Use:** When direct illumination variance is high and the scene contains complex shadowing or participating media [S1].
*   **When Not to Use:** When memory is extremely constrained, or when the scene is optically thin and variance is already low [S1].

## Background
Light Ray Extinction Optimization was introduced in the context of view-dependent visibility optimization for Monte Carlo volume visualization [S1]. It builds upon the idea of optimizing camera-ray visibility by extending the same hierarchical data structure techniques to light rays [S1]. The method addresses the challenge of efficiently computing transmittance in participating media, where standard delta tracking can be inefficient due to high variance in dense regions [S1]. By optimizing the extinction coefficient estimates, the method allows for more efficient sampling of light transport paths [S1].

## Sources
[S1] View‐Dependent Visibility Optimization for Monte Carlo Volume Visualization, https://doi.org/10.1111/cgf.70064, fetched 2026-11-05
[S2] Multimodal Volume Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2015/sunden15multimodal.pdf, fetched 2026-10-29 (unofficial)
