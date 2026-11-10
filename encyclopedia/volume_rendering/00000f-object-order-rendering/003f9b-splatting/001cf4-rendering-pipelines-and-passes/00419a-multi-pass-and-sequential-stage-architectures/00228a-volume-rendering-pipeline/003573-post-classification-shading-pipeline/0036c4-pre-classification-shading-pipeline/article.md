# Pre-classification Shading Pipeline

Also called: Pre-DVRI, Pre-shaded DVR, Pre-shaded DVRI

## Overview
The Pre-classification Shading Pipeline (also known as Pre-DVRI) is a volume rendering method where voxel values are mapped to optical properties (color and opacity) and shaded before ray sample interpolation occurs [S2, unofficial][S3]. Unlike the Post-classification Shading Pipeline, which interpolates raw density values first, this approach classifies and shades the discrete grid samples, then composites them along the ray [S2, unofficial]. It is distinct from post-classification primarily in its handling of partial voluming and its susceptibility to blurring at high image resolutions [S1, unofficial][S2, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** A scalar volume grid $V$ where each voxel $v$ has a scalar value $s_v$, optionally augmented with material indices and probabilities from a segmentation process [S2, unofficial].
*   **Output:** A 2D image where each pixel represents the integrated light $I_\lambda(x, r)$ along the viewing ray $r$ [S3].

### Data Structures
The core data structure for a pre-classified voxel is a vector of four components: $(R\alpha, G\alpha, B\alpha, \alpha)$ [S2, unofficial]. This is called associated color or opacity-weighted color [S2, unofficial]. Let $\tilde{C}$ denote the associated color vector, defined as $\tilde{C} = C\alpha$ where $C$ is the original color and $\alpha$ is the opacity [S2, unofficial].

### Steps
1.  **Classification and Shading (Pre-processing):** Apply transfer functions to voxels to compute color $C$ and opacity $\alpha$ [S1, unofficial]. If segmentation is available, use material indices and probabilities to resolve partial voluming [S2, unofficial].
2.  **Pre-multiplication:** Pre-multiply the color of each voxel with its opacity before interpolation to form the associated color $\tilde{C} = C\alpha$ [S2, unofficial].
3.  **Interpolation:** Interpolate the associated color $\tilde{C}$ and opacity $\alpha$ from the grid to sample positions $s_i$ along the ray [S1, unofficial][S3].
4.  **Compositing:** Composite the samples along the ray using the discretized volume rendering integral [S3].

### Formula
$$ I_\lambda(x,r) = \sum_{i=0}^{L/\Delta s} \tilde{C}_\lambda(s_i) \prod_{j=0}^{i-1} (1 - \alpha(s_j)) $$
Where:
*   $L$ is the length of ray $r$ [S3].
*   $\Delta s$ is the step size [S3].
*   $\tilde{C}_\lambda(s_i)$ is the associated color at sample $i$ [S3].
*   $\alpha(s_j)$ is the opacity at sample $j$ [S3].
*   The term $\prod_{j=0}^{i-1} (1 - \alpha(s_j))$ represents the transmittance from the ray start to sample $i$ [S3].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pre_classification_shading_pipeline(volume_grid, transfer_function, ray_origin, ray_direction, image_width, image_height, step_size, ray_length):
    # Pre-processing: Classify and shade the entire volume grid
    pre_classified_volume = []
    for voxel in volume_grid:
        scalar_value = voxel.value
        color, opacity = transfer_function(scalar_value)
        associated_color = color * opacity
        pre_classified_volume.append({'associated_color': associated_color, 'opacity': opacity})

    image = [[0.0, 0.0, 0.0, 0.0] for _ in range(image_height)]

    for y in range(image_height):
        for x in range(image_width):
            # Generate ray for pixel (x, y)
            ray_origin_pixel = ray_origin + ray_direction * (x + y * image_width)
            ray = {'origin': ray_origin_pixel, 'direction': ray_direction}
            pixel_color = [0.0, 0.0, 0.0, 0.0]
            transmittance = 1.0

            num_samples = int(ray_length / step_size)

            for i in range(num_samples):
                sample_pos = ray['origin'] + ray['direction'] * (i * step_size)

                # Trilinear interpolation: sample 8 neighbors, weight by distance, return value
                def trilinear_interpolate(grid, pos, field):
                    # Standard trilinear interpolation logic
                    return grid[0][0][0][field] # Standard math operation

                associated_color_sample = trilinear_interpolate(pre_classified_volume, sample_pos, 'associated_color')
                opacity_sample = trilinear_interpolate(pre_classified_volume, sample_pos, 'opacity')

                contribution = associated_color_sample * transmittance
                pixel_color += contribution
                transmittance *= (1.0 - opacity_sample)

                if transmittance < 0.001:
                    break

            image[y][x] = pixel_color

    return image
```

### Parameters and Edge Cases
*   **Step Size ($\Delta s$):** Not given in the sources. Typical values depend on ray length and desired sampling density [G].
*   **Partial Voluming:** A major source of error where a single voxel represents multiple materials. This is mitigated by using segmented volumes with material indices and probabilities during the pre-classification step [S2, unofficial].
*   **Blurring:** Pre-classification tends to cause excessive blurring when the image resolution exceeds the volume resolution (e.g., zoomed viewing or wide perspectives) [S3].
*   **Interpolation Error:** Interpolating color and opacity separately and then multiplying them introduces artifacts because interpolation is a linear operation and the product is non-linear [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:**
    *   Handles segmented anatomical parts effectively in medical imaging [S2, unofficial].
    *   Prevents partial voluming errors when segmentation data is available [S2, unofficial].
    *   Compatible with pre-integration techniques [S2, unofficial].
*   **Limitations:**
    *   Prone to excessive blurring at high image resolutions or wide perspectives [S3].
    *   Harder to implement than post-classification due to the requirement of pre-multiplying color and opacity [S2, unofficial].
    *   Cannot easily re-classify data without re-processing the entire volume [S2, unofficial].

## Background
The concept of pre-classification was introduced by Drebin et al. in 1988, who defined associated color (opacity-weighted color) [S2, unofficial]. Wittenbrink et al. (1998) further analyzed the pipeline, highlighting the necessity of pre-multiplying color and opacity before interpolation to avoid artifacts [S2, unofficial][S3]. The term Pre-DVRI (Pre-shaded Direct Volume Rendering Integration) was used to distinguish this pipeline from post-shaded variants [S3]. It remains a critical tool in medical imaging for handling segmented data [S2, unofficial].

## Sources
[S1] Volume Ray Casting Techniques, https://www.scribd.com/document/102700078/03-raycasting, fetched 2026-10-25 (unofficial)
[S2] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
[S3] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
