# Final Composition

Also called: Density-dependent interpolation, Segment compositing, Volume imaging, Volumetric Compositing

## Overview
Final Composition (also known as Volume Imaging) integrates a 3D volume into a 2D image by assigning voxel values to organ ranges with intermediate blending, then applying lighting to depict surfaces without exact boundary segmentation [S1, unofficial]. It differs from threshold-based segmentation by using the full volume and from pure projection by incorporating illumination [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume dataset, intensity ranges defining organ segments, and lighting parameters [S1, unofficial].
- Output: A 2D rendered image [S1, unofficial].

**Data Structures:**
- Voxel Grid: 3D array of intensity and opacity values [G].
- Segment Definitions: Mappings from intensity ranges to organ labels or material properties [S1, unofficial].
- Convolution Buffers: Precomputed values for low-density segments, stored only for the right-hand column of the previous block and the bottom row of the previous sweep to minimize memory [S2, unofficial].

**Steps:**
1. Define intensity ranges for organ segmentation, allowing intermediate values for blending [S1, unofficial].
2. Compute convolutions for low-density segments, retaining only the required boundary rows/columns for memory efficiency [S2, unofficial].
3. For each pixel, cast a ray through the volume and composite voxel segments front-to-back [S2, unofficial].
4. For each segment on the ray, select the rendering solution: low-density convolution weighted by optical depth, high-density diffusion, or a density-dependent interpolation between them to eliminate aliasing artifacts [S2, unofficial].
5. Apply lighting information to the composited volume data to depict organ surfaces [S1, unofficial].
6. Evaluate the final color per pixel [S2, unofficial].

**Formulas:**
The compositing process follows the standard front-to-back accumulation model [G], adapted with segment-specific contributions:
$$
C_{final} = \sum_{i} T_i \cdot \alpha_i \cdot c_i^{seg}
$$
$$
T_i = \prod_{j=0}^{i-1} (1 - \alpha_j)
$$
Where $c_i^{seg}$ is determined by the low-density convolution, high-density diffusion, or their density-dependent interpolation [S2, unofficial]. $T_i$ is the accumulated transmittance up to sample $i$ [G]. $\alpha_i$ is the sample opacity [G].

**Parameters:**
- Intensity Ranges: User-defined or algorithmically determined thresholds for organ segmentation. These are critical; the result is heavily dependent on them [S1, unofficial].
- Optical Depth: Used to weight the low-density convolution solution [S2, unofficial].

**Edge Cases and Pitfalls:**
- Intensity Range Dependency: The method is heavily dependent on the chosen intensity ranges. Incorrect ranges lead to poor visualization or misrepresentation of organs [S1, unofficial].
- Aliasing: Switching between low-density and high-density solutions can cause aliasing artifacts. Density-dependent interpolation is required to mitigate this [S2, unofficial].
- Surface Only: The method portrays organ surfaces but not interior details, as it relies on lighting and surface depiction rather than full volumetric transparency [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def final_composition(volume, intensity_ranges, convolutions):
    """
    Final Composition rendering loop.
    volume: 3D array of voxel intensities and opacities.
    intensity_ranges: mappings defining organ segments.
    convolutions: precomputed values for low-density segments.
    """
    image = empty_2d_array()
    for each pixel (x, y):
        ray = cast_ray_through_volume(x, y) # Not detailed in sources [G]
        C_acc = vec3(0.0)
        alpha_acc = 0.0
        T = 1.0
        for sample in ray:
            # Determine segment solution based on voxel intensity [S2, unofficial]
            if sample.intensity in low_density_range:
                c_seg = convolutions[sample] * sample.optical_depth
                alpha_seg = sample.opacity
            elif sample.intensity in high_density_range:
                c_seg = compute_diffusion_solution(sample) # Equations (8) and (9) not provided in notes [S2, unofficial]
                alpha_seg = sample.opacity
            else:
                # Density-dependent interpolation to eliminate aliasing [S2, unofficial]
                t = compute_interpolation_factor(sample.intensity) # Not given in sources [G]
                c_low = convolutions[sample] * sample.optical_depth
                c_high = compute_diffusion_solution(sample)
                c_seg = c_low + t * (c_high - c_low)
                alpha_seg = sample.opacity

            # Standard front-to-back compositing update [G]
            C_acc = C_acc + T * alpha_seg * c_seg
            alpha_acc = alpha_acc + T * alpha_seg
            T = T * (1.0 - alpha_seg)

            # Early ray termination [G]
            if alpha_acc >= 1.0:
                break

        # Composite over background [G]
        image[x, y] = C_acc + T * background_color
    return image
```

## Operating characteristics
- Time Complexity: Depends on the number of rays and the complexity of the convolution and diffusion calculations [S2, unofficial].
- Memory Complexity: Requires storage for the volume data and intermediate convolution results (specifically for the right-hand column of the previous block and the bottom row of the previous sweep) [S2, unofficial].
- Strengths: Incorporates information from the entire volume; uses lighting to enhance surface perception [S1, unofficial].
- Limitations: Heavily dependent on user-selected intensity ranges; only portrays organ surfaces, not interior structures; potential for aliasing if interpolation is not handled correctly [S1, unofficial][S2, unofficial].
- When to use: When visualizing organ surfaces in nuclear medicine or similar modalities where relative uptake is important, and exact boundary segmentation is not desired [S1, unofficial].
- When not to use: When interior volume details are required, or when user-defined intensity ranges are not feasible or desirable [S1, unofficial].

## Background
Final Composition, also referred to as Volume Imaging or Volumetric Compositing, was developed as an alternative to methods that rely on exact boundary determination or pure maximum activity projection [S1, unofficial]. It emerged in the context of 3D display of SPECT images, aiming to incorporate information from the entire volume while using lighting to depict surfaces [S1, unofficial]. The technique involves a blend of convolution-based low-density handling and diffusion-based high-density handling, interpolated based on density [S2, unofficial].

## Sources
[S1] "Volume rendering in three-dimensional display of SPECT images", http://jnm.snmjournals.org/cgi/pmidlookup?view=long&pmid=2384811, fetched 2026-10-29 (unofficial)
[S2] Diffusion and multiple anisotropic scattering for global illumination in clouds, https://digital.library.unt.edu/ark:/67531/metadc1410406/?q=multiple scattering rendering, fetched 2026-10-29 (unofficial)
