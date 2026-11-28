# Wavelet-Based Adaptive Grid Volume Rendering

Also called: Direct summation of wavelet coefficients during volumetric ray casting, Volume rendering in compression domain, Volumetric ray casting of multi-scale datasets in wavelet space

## Overview
This method visualizes volume data by superimposing scaling functions on a coarse regular grid with wavelets on an adaptive mesh, capitalizing on sparse data representation and wavelet compression [S1][S2, unofficial]. It differs from sibling techniques such as [Wavelet-based Adaptive Resolution Volume Rendering](1171-wavelet-based-adaptive-resolution-volume-rendering.md) and [Wavelet-based Multiresolution Rendering](1588-wavelet-based-multiresolution-rendering.md) by performing visualization directly in wavelet space on the same adaptive grid rather than reconstructing a uniform grid [S2, unofficial], making it suitable for time-dependent data on space-time varying meshes [S1][S2, unofficial].

## How it works
**Inputs:**
*   Wavelet coefficients representing the volume data on an adaptive mesh [S1][S2, unofficial].
*   A wavelet threshold for coefficient retention [S2, unofficial].
*   User-prescribed resolution for the coarse scaling function grid [S1][S2, unofficial].
*   Camera parameters for ray generation [G].

**Outputs:**
*   Rendered image of the volume [S1][S2, unofficial].

**Data Structures:**
*   **Adaptive Mesh:** A list of active wavelet coefficients. Only coefficients with magnitude greater than the wavelet threshold are retained [S2, unofficial].
*   **Coarse Grid:** A uniform grid storing scaling function values at a user-prescribed resolution [S1][S2, unofficial].

**Steps:**
1.  **Preprocessing (Partial Inverse Transform):** Perform a partial inverse wavelet transform to obtain scaling functions on a uniform grid at the user-prescribed resolution [S1][S2, unofficial]. This creates the base representation in physical space [S2, unofficial].
2.  **Representation:** Represent the solution in physical space as a superposition of scaling functions on the coarse regular grid and wavelets on the adaptive mesh [S1][S2, unofficial].
3.  **Ray Casting Initialization:** Execute an efficient and accurate ray casting algorithm based on the coarse scaling functions [S2, unofficial]. This provides the base rendering [S2, unofficial].
4.  **Detail Addition:** Add additional details during ray tracing by including wavelets based on the following criteria [S2, unofficial]:
    *   Support overlap with the interpolation point.
    *   Wavelet coefficient magnitude.
    *   Opacity accumulation (front to back ordering).
    *   Deviation from frontal viewing direction.

**Parameters:**
*   **Wavelet Threshold:** Coefficients greater than this threshold are retained for the adaptive mesh [S2, unofficial]. This controls the compression factor, which is typically 10–10^2 relative to a uniform grid of the same effective resolution [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_wavelet_volume(wavelet_coeffs, coarse_scaling_grid, camera_origin, camera_direction, image_width, image_height, threshold):
    # Initialize output image
    image = [[vec3(0.0)] * image_width for _ in range(image_height)]

    for y in range(image_height):
        for x in range(image_width):
            # Generate ray for pixel (x, y)
            ray_origin = camera_origin
            ray_dir = normalize(camera_direction + vec3(x - image_width / 2.0, y - image_height / 2.0, 0.0))

            color = vec3(0.0)
            opacity = 0.0

            # Step 3: Base ray casting on coarse scaling functions
            # Evaluate coarse grid along ray using standard trilinear interpolation [G]
            coarse_val = interpolate_3d(coarse_scaling_grid, ray_origin, ray_dir)
            base_color, base_opacity = apply_transfer_function(coarse_val)

            # Step 4: Add details from wavelets
            # Sort wavelets by distance along ray for front-to-back ordering [G]
            sorted_wavelets = sorted(wavelet_coeffs, key=lambda w: dot(w.position - ray_origin, ray_dir))

            for w in sorted_wavelets:
                # Check support overlap with interpolation point [S2, unofficial]
                if not check_support_overlap(w, ray_origin, ray_dir):
                    continue
                # Check wavelet coefficient magnitude [S2, unofficial]
                if abs(w.coefficient) < threshold:
                    continue
                # Check deviation from frontal viewing direction [S2, unofficial]
                if abs(dot(normalize(ray_dir), w.normal)) < 0.5:  # [G] standard directional threshold
                    continue

                # Evaluate wavelet contribution
                wavelet_val = w.coefficient * w.basis_function(ray_origin, ray_dir)
                w_color, w_opacity = apply_transfer_function(wavelet_val)

                # Front-to-back compositing [S2, unofficial]
                alpha = w_opacity * (1.0 - opacity)
                color += w_color * alpha
                opacity += alpha
                if opacity >= 1.0:
                    break

            # Composite base color with accumulated wavelet details [G]
            final_color = color + base_color * (1.0 - opacity)
            image[y][x] = final_color

    return image

def interpolate_3d(grid, origin, direction):
    # Standard trilinear interpolation along ray [G]
    t = 0.0
    step = 1.0 / grid.resolution
    val = 0.0
    for _ in range(grid.resolution):
        pos = origin + direction * t
        val += grid.sample(pos)
        t += step
    return val / grid.resolution

def check_support_overlap(wavelet, origin, direction):
    # Check if ray passes through wavelet support volume [G]
    return dot(wavelet.center - origin, direction) > 0 and distance(wavelet.center, origin) < wavelet.radius

def apply_transfer_function(value):
    # Standard transfer function mapping [G]
    return vec3(value, value, value), clamp(value, 0.0, 1.0)
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1][S2, unofficial].
*   **Memory Complexity:** Not given in the sources [S1][S2, unofficial].
*   **Measured Performance:** Not given in the sources [S1][S2, unofficial].
*   **Scaling:** Not given in the sources [S1][S2, unofficial].
*   **Strengths:** Capitalizes on sparse data representation and wavelet compression [S1][S2, unofficial]. Allows visualization of time-dependent data on space-time varying adaptive meshes [S1][S2, unofficial]. Provides a compression factor of 10–10^2 relative to a uniform grid [S2, unofficial].
*   **Limitations:** Not given in the sources [S1][S2, unofficial].
*   **When to Use:** When visualizing large volume datasets that overwhelm video memory, especially if the data is already represented on a wavelet-based adaptive computational mesh [S1][S2, unofficial].

## Background
This technique was developed to address the limitations of existing volumetric visualization techniques that do not take full advantage of multi-resolution wavelet analysis [S1][S2, unofficial]. It is tailored for visualization of compressed solutions on wavelet-based adaptive computational meshes [S1][S2, unofficial]. The method was illustrated and compared to existing volume visualization software for Rayleigh-Benard thermal convection and electron density data sets [S2, unofficial].

## Sources
[S1] Volumetric Rendering on Wavelet-Based Adaptive Grid, https://doi.org/10.3390/fluids7070245, fetched 2026-10-26
[S2] Volumetric Rendering on Wavelet-Based Adaptive Grid - MDPI, https://www.mdpi.com/2311-5521/7/7/245, fetched 2026-10-28 (unofficial)
