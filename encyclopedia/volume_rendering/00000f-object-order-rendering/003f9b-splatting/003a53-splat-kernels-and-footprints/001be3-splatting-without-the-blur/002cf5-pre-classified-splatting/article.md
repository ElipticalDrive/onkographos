# Pre-Classified Splatting


## Overview
Pre-classified splatting is a volume rendering strategy where classification and shading are applied to the raw density volume prior to projection [S1, unofficial]. This approach results in a blurred edge image upon reconstruction due to the subsequent smoothing kernel [S1, unofficial]. It is distinguished from post-classified splatting, which performs classification after projection to preserve crisp edges [S1, unofficial].

## How it works
**Inputs:** Raw density volume, viewing parameters (camera position, orientation, field of view) [S1, unofficial].
**Outputs:** Final 2D image [S1, unofficial].

**Data Structures:**
1. **Color and Opacity Volume:** A 3D grid storing pre-classified color ($C$) and opacity ($\alpha$) values for each voxel, derived from the raw density volume [S1, unofficial].
2. **Sheet Buffer:** A 2D buffer aligned with the image plane (or principal axes) used to accumulate splatted contributions [S1, unofficial].

**Steps:**
1. **Classify and Shade:** Iterate through the raw density volume. For each voxel, apply a transfer function to determine its final color $C$ and opacity $\alpha$ [S1, unofficial].
2. **Splat:** Project the voxels from the Color and Opacity Volume onto the sheet buffer. Each voxel is treated as a splat with a specific footprint (e.g., Gaussian kernel) [S1, unofficial]. Footprint mapping is performed as usual, and multiple footprint rasterizations per point are required for axis-aligned or image-aligned cases [S1, unofficial].
3. **Composite:** Combine the accumulated values in the sheet buffer to produce the final pixel colors for the current view [S1, unofficial].
4. **Advance:** Update the sheet buffer position or structure for the next viewing angle or slice [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pre_classified_splatting(raw_density_volume, transfer_function, camera_params):
    # Step 1: Classify and Shade
    color_opacity_volume = []
    for voxel in raw_density_volume:
        c, a = transfer_function(voxel.density)
        color_opacity_volume.append({'color': c, 'opacity': a, 'pos': voxel.pos})
    
    # Step 2: Splat into Sheet Buffer
    sheet_buffer = [[{'color': [0.0, 0.0, 0.0], 'opacity': 0.0} for _ in range(camera_params.width)] for _ in range(camera_params.height)]
    for voxel in color_opacity_volume:
        if voxel['opacity'] > 0:
            # Footprint mapping as usual; requires multiple footprint rasterizations per point
            # for axis-aligned or image-aligned cases [S1, unofficial]
            footprint_pixels = project_and_rasterize(voxel['pos'], camera_params)
            for px in footprint_pixels:
                sheet_buffer[px.y][px.x]['color'] += voxel['color'] * voxel['opacity']
                sheet_buffer[px.y][px.x]['opacity'] += voxel['opacity']
    
    # Step 3: Composite
    final_image = []
    for y in range(camera_params.height):
        row = []
        for x in range(camera_params.width):
            c, a = sheet_buffer[y][x]['color'], sheet_buffer[y][x]['opacity']
            row.append(composite_pixel(c, a))
        final_image.append(row)
    return final_image

def project_and_rasterize(position, camera_params):
    # Standard screen projection and kernel footprint generation
    return generate_footprint_pixels(position, camera_params)

def generate_footprint_pixels(position, camera_params):
    # Project 3D position to 2D screen coordinates and generate kernel footprint pixels
    screen_x, screen_y = project_to_screen(position, camera_params)
    radius = get_kernel_radius(camera_params)
    return [(screen_x + dx, screen_y + dy) for dx in range(-radius, radius + 1) for dy in range(-radius, radius + 1)]

def project_to_screen(position, camera_params):
    # Standard perspective/orthographic projection
    return position.x * camera_params.scale_x, position.y * camera_params.scale_y

def get_kernel_radius(camera_params):
    # Standard kernel size based on resolution and sampling
    return 2

def composite_pixel(color, opacity):
    # Standard alpha blending of accumulated buffer values
    return color / max(opacity, 1e-6)
```

**Parameters:**
*   **Transfer Function:** Maps density to color and opacity [S1, unofficial].
*   **Splat Kernel:** Gaussian kernel is commonly used [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Blurred Edges:** The primary pitfall is that the reconstruction results in a blurred edge image because the classification is done before the smoothing effect of the splatting kernel is applied [S1, unofficial].
*   **Pre-processing Cost:** Requires significant pre-processing to classify and shade the entire volume [S1, unofficial].

## Subtopics

### Edge Splats
Edge splats are a solution to the blurred edges problem in pre-classified splatting by replacing normal splats with special edge splats [S1, unofficial]. This method aims to preserve edge sharpness but has shortcomings including required pre-processing, problems with discontinuities, and difficulty resolving 'micro-edges' [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Results in blurred edges upon reconstruction and requires pre-processing [S1, unofficial].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
Pre-classified splatting is a foundational technique in volume rendering, often contrasted with post-classified splatting (Mueller '99) [S1, unofficial]. It represents an early approach to handling volume data by pre-computing optical properties [S1, unofficial].

## Sources
[S1] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
