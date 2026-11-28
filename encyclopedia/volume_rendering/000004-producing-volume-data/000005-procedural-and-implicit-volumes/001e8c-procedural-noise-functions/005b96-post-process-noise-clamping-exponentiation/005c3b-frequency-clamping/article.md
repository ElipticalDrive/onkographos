# Frequency Clamping

Also called: Band-pass Noise Filtering, High-Frequency Noise Cancellation

## Overview
Frequency clamping is a noise filtering technique that cancels the contribution of noise bands whose frequency exceeds a specified threshold, exploiting spectral control to generate filtered noise directly [S1, S2, S3]. It distinguishes itself from explicit post-generation filtering (such as convolution) by operating on the constituent band-pass components of the noise function itself [S1, S2, S3]. This method is most effective when the underlying noise is narrowly band-pass, though it faces limitations with isotropic noise sources like Perlin noise [S1, S2, S3].

## How it works
Frequency clamping operates on noise patterns constructed as a weighted sum of band-pass noise functions. The method identifies frequency bands that are too high for the target resolution or visual intent and nullifies their contribution to the final output [S1, S2, S3].

### Inputs and Outputs
*   **Input**: A composite noise function $N(\mathbf{x})$ defined as a weighted sum of individual band-pass noise components $n_i(\mathbf{x})$, where each $n_i$ has a specific frequency support.
*   **Output**: A filtered noise value $N_{filtered}(\mathbf{x})$ with high-frequency components removed.

### Data Structures
The implementation requires a representation of the noise composition. Typically, this involves a list or array of noise components, where each component stores:
1.  **Weight ($w_i$)**: The scalar multiplier for the band.
2.  **Frequency Support ($\Omega_i$)**: The range of frequencies (e.g., minimum and maximum wavenumbers) associated with the band.
3.  **Noise Function ($f_i$)**: The underlying procedural noise generator for that band.

### Steps
1.  **Decompose**: Identify the constituent band-pass noise components $n_i(\mathbf{x})$ and their respective weights $w_i$ in the composite noise function.
2.  **Evaluate Threshold**: Determine the cutoff frequency $f_c$ based on the target resolution or desired visual smoothness.
3.  **Filter Components**: For each component $i$:
    *   Check if the frequency support $\Omega_i$ of $n_i$ exceeds $f_c$.
    *   If the component's frequencies are entirely above $f_c$, set its effective weight $w'_i = 0$.
    *   If the component's frequencies are entirely below $f_c$, set $w'_i = w_i$.
    *   If the component's frequencies straddle $f_c$, the method as described in the sources does not specify a partial attenuation; it implies a hard cutoff or requires the noise to be narrowly band-pass to avoid this ambiguity [S1, S2, S3].
4.  **Reconstruct**: Compute the final noise value using the filtered weights:
    $$ N_{filtered}(\mathbf{x}) = \sum_{i} w'_i \cdot n_i(\mathbf{x}) $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def frequency_clamp(composite_noise, cutoff_frequency):
    """
    Applies frequency clamping to a composite noise function.
    
    Args:
        composite_noise: An object or list containing band-pass components.
                         Each component has attributes:
                         - weight: float
                         - max_frequency: float (upper bound of its spectral support)
                         - evaluate(point): function returning noise value
        cutoff_frequency: float, the maximum frequency to retain.
    
    Returns:
        float: The filtered noise value at the current evaluation point.
    """
    total_value = 0.0
    
    for component in composite_noise.components:
        # Step 3: Filter components based on frequency support
        # If the component's highest frequency is below the cutoff, keep it.
        # If it exceeds the cutoff, its contribution is cancelled.
        if component.max_frequency <= cutoff_frequency:
            effective_weight = component.weight
        else:
            effective_weight = 0.0
            
        # Step 4: Reconstruct
        total_value += effective_weight * component.evaluate(current_point)
        
    return total_value
```

### Edge Cases and Pitfalls
*   **Weakly Band-Pass Noise**: Frequency clamping works best if the noise is narrowly band-pass, meaning the spectral ring is thin and well-defined [S1, S2, S3]. Perlin noise is only weakly band-pass, making frequency clamping difficult to tune effectively [S1, S2, S3].
*   **Isotropy Limitation**: Standard frequency clamping is isotropic. It cannot account for the anisotropy of the filter on tilted surfaces, leading to a compromise between over-blurring or residual aliasing [S1, S2, S3].
*   **Straddling Frequencies**: If a noise band's frequency support straddles the cutoff threshold, a hard clamp may introduce artifacts. The sources suggest that using noise with better-defined bandlimits (e.g., Cook and DeRose noise) alleviates tuning issues [S1, S2, S3].

## Subtopics

### Oriented Band-Pass Noise
A variation where pre-computed noise bands are oriented, allowing each band to correspond to a noise pattern with limited frequency content along a given orientation. This addresses the isotropy limitation of standard frequency clamping [S1, S2, S3]. See [Oriented Noise Filtering](link-to-orientated-noise.md).

## Operating characteristics
*   **Strengths**: Directly generates noise with the filtered power spectrum, avoiding the computational cost of explicit filtering on unfiltered noise [S1, S2, S3].
*   **Limitations**: Performance and quality depend heavily on the spectral properties of the base noise. It is ineffective for noise with broad or undefined spectral support [S1, S2, S3].
*   **When to use**: When the noise function is explicitly constructed from narrow band-pass components and high-frequency aliasing needs to be removed at the source [S1, S2, S3].
*   **When not to use**: With standard Perlin noise due to its weak band-pass characteristics, or when anisotropic filtering is required for tilted surfaces [S1, S2, S3].

## Background
Frequency clamping was introduced as a method to handle high-frequency noise in procedural generation [S1, S2, S3]. It is often referred to by the citation [NRS82] in the context of canceling high-frequency bands [S1, S2, S3]. The technique is discussed in contrast to explicit filtering methods, which apply a filter to unfiltered noise after generation [S1, S2, S3]. Improvements to the underlying noise functions, such as those by Cook and DeRose [CD05] and Goldberg et al. [GZD08], have been developed to make frequency clamping more effective by providing better bandlimits and oriented spectral content [S1, S2, S3].

## Sources
[S1] A Survey of Procedural Noise Functions, https://doi.org/10.1111/j.1467-8659.2010.01827.x, fetched 2026-10-27
[S2] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S3] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
