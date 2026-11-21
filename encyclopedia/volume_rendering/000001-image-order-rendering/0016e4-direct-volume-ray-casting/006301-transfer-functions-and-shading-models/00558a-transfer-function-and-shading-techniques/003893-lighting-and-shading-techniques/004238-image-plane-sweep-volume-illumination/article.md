# Image Plane Sweep Volume Illumination

Also called: Image Plane Sweep, Sundén et al

## Overview
Image Plane Sweep Volume Illumination is a GPU-based volume ray-casting technique that reduces illumination computation complexity by exploiting the plane sweep paradigm [S1]. It enables interactive frame rates while supporting advanced effects such as scattering and shadowing [S1]. Unlike standard ray casting which casts viewing rays directly from the viewpoint, this method computes illumination for a screen-space line first, then casts viewing rays line by line [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scalar volume data $V$, light direction $\mathbf{l}$, view direction $\mathbf{v}$, camera parameters, and transfer functions for color $C_{base}$ and opacity $\alpha$ [G].
*   **Output:** Final composited image with volumetric illumination (shadows and scattering) [S1][S2, unofficial].

**Data Structures:**
*   **Illumination Buffer:** A 2D buffer storing precomputed illumination values for each screen-space line segment [G].
*   **Volume Texture:** 3D texture storing the scalar field $V$ [G].

**Steps:**
1.  **Project Light Direction:** Project the light direction vector $\mathbf{l}$ onto the image plane to determine the sweep direction [S2, unofficial].
2.  **Line-Sweep Illumination Computation:** Perform a line-sweep computation in image space along the projected light direction [S2, unofficial]. This step calculates illumination information (such as shadowing and scattering contributions) for each screen-space line [S2, unofficial]. The optical model considers either single scattering with a single light direction, or multiple scattering resulting from a rather high albedo [S1]. To capture high-frequency shadows and diffuse scattering simultaneously, these two formulations are combined [S1].
3.  **Viewing Ray Casting:** Once illumination information for a line is computed, cast viewing rays from the image plane line by line [S2, unofficial].
4.  **Shading Modification:** Modify the shading of the samples using the ray casting algorithm based on the precomputed line lighting [S2, unofficial]. The base color $C_{base}$ and opacity $\alpha$ are computed as per standard direct volume ray casting, and the final color is modulated by the precomputed illumination term [S1][S2, unofficial]. The opacity $\alpha$ is derived solely from the transfer function and is generally not modified by the shading model, preserving the physical interpretation of density [G]. The shading model affects only the color channel [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def image_plane_sweep_volume_illumination(volume, light_dir, view_dir, camera):
    # Step 1: Project light direction onto the image plane
    sweep_dir = normalize(light_dir - dot(light_dir, view_dir) * view_dir)

    # Step 2: Line-sweep illumination computation in image space
    width, height = camera.resolution
    illumination_buffer = [[0.0] * width for _ in range(height)]

    # Group pixels into lines aligned with sweep_dir
    lines = group_pixels_by_line(sweep_dir, width, height)

    for line in lines:
        # Compute illumination for the line (shadows, scattering)
        # Sources do not specify the exact integration formula for line illumination [Comment]
        line_illum = compute_line_illumination(line, volume, light_dir, camera)

        for i, pixel in enumerate(line):
            illumination_buffer[pixel.y][pixel.x] = line_illum[i]

    # Step 3 & 4: Cast viewing rays line by line and apply shading
    image = [[0.0, 0.0, 0.0] for _ in range(height)]

    for line in lines:
        for pixel in line:
            ray_origin = camera.origin
            ray_dir = normalize(pixel.position - ray_origin)

            acc_color = [0.0, 0.0, 0.0]
            acc_alpha = 0.0

            for sample in march_volume(ray_origin, ray_dir, volume):
                val = volume.sample(sample.position)
                base_color, alpha = transfer_function(val)

                light_term = illumination_buffer[pixel.y][pixel.x]
                shaded_color = [c * light_term for c in base_color]

                weight = alpha * (1.0 - acc_alpha)
                acc_color = [acc_color[i] + shaded_color[i] * weight for i in range(3)]
                acc_alpha += weight

            image[pixel.y][pixel.x] = acc_color

    return image

def group_pixels_by_line(sweep_dir, width, height):
    # Standard rasterization: assign each pixel to a line index based on sweep_dir [G]
    lines = {}
    for y in range(height):
        for x in range(width):
            line_idx = x * sweep_dir.y - y * sweep_dir.x
            if line_idx not in lines:
                lines[line_idx] = []
            lines[line_idx].append((x, y))
    return list(lines.values())

def compute_line_illumination(line, volume, light_dir, camera):
    # Sources do not specify the exact integration formula for line illumination [Comment]
    return [1.0] * len(line)

def transfer_function(val):
    # Standard transfer function mapping scalar value to color and opacity [G]
    return [0.5, 0.5, 0.5], 0.1

def march_volume(origin, direction, volume):
    # Standard volume marching along ray [G]
    return [(origin[i] + j * direction[i]) for j in range(10) for i in range(3)]
```

**Parameters:**
*   **Sweep Resolution:** Not given in the sources.
*   **Scattering Model:** Combines single scattering and multiple scattering approximations [S1].
*   **Lighting Model:** Typically uses Phong illumination or simplified intensity models as per the parent article [G].

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.
*   **Degenerate Input:** If the light direction is parallel to the image plane, the sweep direction may become degenerate. Not given in the sources.

## Subtopics

### Single Scattering Approximation
Computes illumination assuming a single light direction and single scattering events [S1]. Sets it apart by capturing high-frequency shadowing effects [S1]. Choose when direct lighting and sharp shadows are the primary visual requirement. See [Direct volume ray casting](19-direct-volume-ray-casting.md).

### Multiple Scattering Approximation
Computes illumination assuming multiple scattering events, typically triggered by high albedo volumes [S1]. Sets it apart by capturing diffuse scattering effects [S1]. Choose when volumetric haze or fog-like appearance is desired. See [Direct volume ray casting](19-direct-volume-ray-casting.md).

## Operating characteristics
*   **Time Complexity:** Reduces illumination computation complexity compared to standard ray casting by leveraging the plane sweep paradigm [S1].
*   **Memory Complexity:** Requires additional storage for the illumination buffer [G].
*   **Measured Performance:** Achieves interactive frame rates [S1].
*   **Strengths:** Supports scattering and shadowing interactively; combines high-frequency shadows with diffuse scattering [S1].
*   **Limitations:** Limited to specific lighting configurations (single light direction for single scattering) [S1].
*   **When to Use:** When interactive performance is required for volumes with complex lighting effects like shadows and scattering [S1].
*   **When Not to Use:** When global illumination with multiple lights is required without approximation [S1].

## Background
Image Plane Sweep Volume Illumination was proposed by Sundén et al. [S1][S2, unofficial]. It is based on a reformulation of the optical model to integrate advanced illumination effects into GPU-based volume ray-casters [S1]. The method was developed to address the computational cost of volumetric lighting in real-time applications [S1].

## Sources
[S1] Interactive Volume Rendering with Volumetric Illumination, https://doi.org/10.2312/conf/eg2012/stars/053-074, fetched 2026-10-26
[S2] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
