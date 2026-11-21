# Real-Time Volumetric Shadows using 1D Min-Max Mipmaps

Also called: 1D Min-Max Mipmap Volumetric Shadows

## Overview
This method computes real-time volumetric shadows in single-scattering participating media on the GPU by evaluating the scattering integral over the intersections of camera rays with a shadow map treated as a 2D height field [S1]. It distinguishes itself from sibling methods by using epipolar rectification to reduce the problem to 1D height fields and employing 1D min-max mipmaps to find lit segments in parallel, allowing all camera rays to be processed simultaneously using only standard pixel shader features without requiring GPGPU APIs or camera rectification [S1].

## How it works
The algorithm evaluates the single-scattering integral for each pixel by identifying the visible segments of the volume along the camera ray that are illuminated by the light source [S1].

### Inputs and Outputs
*   **Inputs:** Camera depth map, light shadow map, light position, and camera position [S1].
*   **Outputs:** Final color buffer with volumetric shadows [S1].

### Data Structures
1.  **Shadow Map:** A 2D depth texture representing distances from the light to the volume boundary [S1].
2.  **Rectified Shadow Map:** A 2D texture where rows correspond to epipolar lines [S1].
3.  **1D Min-Max Mipmap:** A hierarchy per row storing minimum and maximum depth values for coarse-to-fine traversal to find lit segments [S1].

### Steps

1.  Render a depth map from the camera and a shadow map from the light [S1].
2.  Perform epipolar rectification on the shadow map so that each camera ray travels through a single row of the rectified shadow map (an epipolar slice) [S1].
3.  Compute a low-rank approximation to all but the visibility terms in the scattering integral [S1].
4.  For each row of the rectified shadow map, compute a 1D min-max mipmap [S1].
5.  For each camera ray, traverse the min-max mipmap to find lit segments and accumulate the scattering integral [S1].
6.  If the epipole is on or near the screen, compute the scattering near the epipole using brute force ray marching [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_volumetric_shadows(camera_depth, shadow_map, light_pos, camera_pos):
    # 1. Render depth maps (provided as inputs)
    # 2. Epipolar rectification
    rectified_map = epipolar_rectify(shadow_map, light_pos, camera_pos)
    # 3. Low-rank approximation for non-visibility terms
    scattering_params = compute_scattering_params() # [G] Precomputed density/scattering coefficients
    # 4. Build 1D min-max mipmaps for each row
    mipmaps = []
    for row in rectified_map.rows:
        mipmaps.append(build_min_max_mipmap(row.depths))
    # 5. Process each pixel
    output = [[0.0 for _ in range(camera_depth.width)] for _ in range(camera_depth.height)]
    for y in range(camera_depth.height):
        for x in range(camera_depth.width):
            if is_near_epipole(x, y, light_pos, camera_pos): # [G] Geometric proximity check
                output[y][x] = brute_force_ray_march(camera_depth, shadow_map, light_pos, camera_pos, x, y)
            else:
                row_idx = get_row_index(x, y, rectified_map)
                mipmap = mipmaps[row_idx]
                ray_depth = camera_depth[y][x]
                segments = find_lit_segments(mipmap, ray_depth)
                output[y][x] = integrate_segments(segments, scattering_params)
    return output

def build_min_max_mipmap(depths):
    levels = [depths]
    current = depths
    while len(current) > 1:
        next_level = []
        for i in range(0, len(current) - 1, 2):
            next_level.append((min(current[i], current[i+1]), max(current[i], current[i+1])))
        levels.append(next_level)
        current = next_level
    return levels

def find_lit_segments(mipmap, ray_depth):
    segments = []
    for level in reversed(mipmap):
        i = 0
        while i < len(level):
            min_d, max_d = level[i]
            if min_d < ray_depth < max_d:
                if len(level) == 1:
                    segments.append((i, i + 1))
            i += 1
    return segments

def epipolar_rectify(shadow_map, light_pos, camera_pos):
    # [G] Standard transformation aligning shadow map rows with epipolar lines
    return shadow_map.transform_to_epipolar_grid()

def compute_scattering_params():
    # [G] Standard precomputation of volume density and scattering coefficients
    return {"sigma_s": 0.1, "L_in": 1.0}

def is_near_epipole(x, y, light_pos, camera_pos):
    # [G] Standard check for epipole projection proximity
    return True

def brute_force_ray_march(camera_depth, shadow_map, light_pos, camera_pos, x, y):
    # [G] Standard ray marching loop for high-accuracy integration
    return 0.0

def get_row_index(x, y, rectified_map):
    # [G] Standard mapping from screen coordinates to rectified row index
    return y

def integrate_segments(segments, params):
    # [G] Standard accumulation of scattering integral over lit intervals
    return 0.0
```

### Parameters and Edge Cases
*   **Epipole Handling:** The threshold for "near the epipole" determines when to switch to brute-force ray marching [S1].
*   **Not given in the sources:** Exact mipmap level counts, precision thresholds, low-rank approximation dimensions, and the specific low-rank computation method.

## Operating characteristics
*   **Time Complexity:** The outer loop is over pixels, allowing all camera rays to be processed in parallel [S1].
*   **Memory Complexity:** Requires storage for the shadow map, rectified shadow map, and 1D min-max mipmaps for each row [S1].
*   **Performance:** Achieves real-time performance using only DirectX 9 pixel shader features, avoiding GPGPU APIs and large vertex transfers, making it suitable for high-resolution shadow maps and complex scenes like forests [S1].
*   **Strengths:** Parallel processing, no GPGPU API dependency, handles high-resolution shadow maps, supports textured lights [S1].
*   **Limitations:** Sacrifices worst-case guarantees compared to incremental integration methods [S1].

## Background
The method was introduced in the paper "Real-Time Volumetric Shadows using 1D Min-Max Mipmaps" [S1]. It builds on the observation that epipolar rectification can simplify the 2D shadow map intersection problem to 1D [S1]. Min-max mipmaps, previously used for soft shadows and global illumination, are adapted here for volumetric shadow computation [S1]. The method contrasts with Baran et al.'s incremental integration, which requires GPGPU APIs and camera rectification, and Billeter et al.'s method, which processes shadow volume elements rather than pixels [S1].

## Sources
[S1] Real-Time Volumetric Shadows using 1D Min-Max Mipmaps, https://groups.csail.mit.edu/graphics/mmvs/mmvs.pdf, fetched 2026-10-28
