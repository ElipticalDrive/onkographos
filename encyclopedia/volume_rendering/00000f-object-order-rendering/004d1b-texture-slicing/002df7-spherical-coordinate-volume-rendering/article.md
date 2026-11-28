# Spherical Coordinate Volume Rendering

Also called: Geodetic Volume Rendering, Spherical Coordinate Volume Rendering for Atmospheric Data, Spherical coordinate volume ray casting, Weather radar volume rendering implementation

## Overview
Spherical Coordinate Volume Rendering is a volume ray casting technique designed to render radar or atmospheric data directly in spherical coordinates, eliminating the need for preprocessing or resampling to a uniform Cartesian grid [S1, book][S2, unofficial]. It solves the accuracy loss and memory trade-offs associated with Cartesian resampling or the geometric gaps left by textured conic surface approximations [S1, book]. Unlike standard texture slicing which relies on parallel planar cuts, this method operates on the intrinsic curvilinear structure of the data, making it distinct from siblings like [View-Aligned Slicing with 3D Textures](870-view-aligned-slicing-with-3d-textures.md) which assume regular grid structures.

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset stored as stacked 2D images representing radar sweeps, where each image corresponds to a specific elevation angle. The dataset contains non-uniform elevation spacing.
*   **Output:** A rendered 2D image (e.g., 800×800 canvas) composited via hardware alpha blending.

**Data Structures:**
The volume is stored in memory as a single large image formed by stacking 2D images vertically [S1, book].
*   **Image Layout:** Each 2D image represents one elevation sweep.
    *   **Columns:** 360 columns, spaced 1 degree apart in azimuth ($\phi$).
    *   **Rows:** 400 rows, spaced 250 meters apart in range ($r$).
*   **Coordinate System:** The data is indexed by spherical coordinates $(r, \phi, \theta)$, where $\theta$ is the elevation angle.

**Algorithm Steps:**
The method extends general volume ray casting [S1, book] by modifying the sample lookup process to handle spherical coordinates and non-uniform elevation spacing.

1.  **Ray Traversal:** For each pixel, traverse the ray through the volume in tens to hundreds of steps (e.g., 200 steps for an 800×800 canvas) [S1, book].
2.  **Coordinate Conversion:** At each step, let the current sample point in Cartesian coordinates be $(x, y, z)$. Convert these to spherical coordinates $(r, \phi, \theta)$ using the following equations [S1, book]:
    $$ r = \sqrt{(x - 0.5)^2 + (y - 0.5)^2 + (z - 0.5)^2} $$
    $$ \phi = \arctan(y, x) + \pi $$
    $$ \theta = \arccos(z/\phi) $$
    *Note: The source uses $\phi$ in the denominator of the $\theta$ equation, which is mathematically inconsistent with standard spherical definitions where $\phi$ is the azimuth. However, this is the exact formula provided in the source [S1, book].*
3.  **Azimuth and Range Lookup:**
    *   Map $\phi$ to the column index (0–359).
    *   Map $r$ to the row index within the elevation slice (0–399).
4.  **Elevation Search (Handling Non-Uniform Spacing):**
    *   Because elevation angles are not uniformly incremented, there is no direct linear mapping from $\theta$ to the slice index in the stacked image [S1, book].
    *   Perform a search to find the two elevation slices (images) immediately above and below the current sample point's $\theta$ value.
5.  **Interpolation:**
    *   Retrieve the values from the two bounding elevation slices at the calculated $(r, \phi)$ position.
    *   Interpolate between these two values to determine the final volume value at the sample point [S1, book].
6.  **Compositing:**
    *   Accumulate the color and opacity of the sample using the standard hardware alpha blending equation, processing slices in depth order [S1, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_spherical_volume(ray_origin, ray_direction, volume_data, elevation_angles):
    """
    ray_origin: (x, y, z)
    ray_direction: (dx, dy, dz)
    volume_data: List of 2D arrays (images), one per elevation angle.
                 Each image is 400 rows (range) x 360 cols (azimuth).
    elevation_angles: List of theta values corresponding to each image in volume_data.
    """
    num_steps = 200
    step_size = length(ray_direction) # Assumed normalized or derived from scene bounds
    accumulated_color = (0, 0, 0)
    accumulated_opacity = 0
    
    # Precompute elevation angle bounds for faster search if needed, 
    # but source implies a search process is required [S1, book].
    
    for i in range(num_steps):
        # Current point in Cartesian coordinates
        t = i * step_size
        x = ray_origin.x + ray_direction.x * t
        y = ray_origin.y + ray_direction.y * t
        z = ray_origin.z + ray_direction.z * t
        
        # Convert to spherical coordinates [S1, book]
        r = sqrt((x - 0.5)**2 + (y - 0.5)**2 + (z - 0.5)**2)
        phi = atan2(y, x) + pi
        # Note: Source formula uses phi in denominator, potentially erroneous 
        # but implemented as specified [S1, book].
        theta = acos(z / phi)
        
        # 1. Find bounding elevation slices [S1, book]
        # Search for indices k such that elevation_angles[k] <= theta < elevation_angles[k+1]
        # This requires a search process due to non-uniform spacing.
        lower_idx, upper_idx = find_bounding_elevation(theta, elevation_angles)
        
        # 2. Map r and phi to image coordinates
        # Range resolution: 250m per row. Azimuth: 1 deg per col.
        # Assuming r and phi are normalized or scaled appropriately to indices.
        row_idx = r / 250.0 
        col_idx = phi / (2 * pi) * 360.0
        
        # Clamp indices to valid range [0, 399] and [0, 359]
        row_idx = clamp(row_idx, 0, 399)
        col_idx = clamp(col_idx, 0, 359)
        
        # 3. Interpolate between elevation slices [S1, book]
        # Interpolate factor based on theta distance between lower and upper angles
        theta_lower = elevation_angles[lower_idx]
        theta_upper = elevation_angles[upper_idx]
        t_theta = (theta - theta_lower) / (theta_upper - theta_lower)
        
        # Get values from the two slices at the same (row, col)
        val_lower = volume_data[lower_idx][row_idx][col_idx]
        val_upper = volume_data[upper_idx][row_idx][col_idx]
        
        volume_value = lerp(val_lower, val_upper, t_theta)
        
        # 4. Apply transfer function to get color and opacity
        color, opacity = transfer_function(volume_value)
        
        # 5. Composite
        # Standard alpha blending: C_dest = C_src * A_src + C_dest * (1 - A_src)
        # [S1, book] references general volume ray casting compositing
        alpha_contrib = opacity * (1 - accumulated_opacity)
        accumulated_color += color * alpha_contrib
        accumulated_opacity += alpha_contrib
        
        if accumulated_opacity >= 1.0:
            break
            
    return accumulated_color
```

**Parameters:**
*   **Ray Traversal Steps:** Tens to hundreds per ray. Example: 200 steps for 800×800 rendering [S1, book].
*   **Azimuth Resolution:** 360 columns, 1 degree spacing [S1, book].
*   **Range Resolution:** 400 rows, 250 meters spacing [S1, book].

**Edge Cases and Pitfalls:**
*   **Non-Uniform Elevation Spacing:** The irregular spacing of elevation angles prevents direct index calculation, necessitating a search process to find bounding slices, which increases computational complexity [S1, book].
*   **Precision:** The conversion formulas must be implemented carefully. The source formula for $\theta$ uses $\phi$ in the denominator, which may lead to division by zero or incorrect values if $\phi$ is near 0 or if the standard definition of spherical coordinates is assumed [S1, book].
*   **Performance:** The method requires a high number of non-trivial computations (trigonometric functions, searches, interpolations) per pixel, resulting in complex shaders [S1, book].

## Operating characteristics
*   **Time Complexity:** High per-pixel cost due to trigonometric conversions and elevation searches for every sample point along every ray [S1, book].
*   **Memory Complexity:** Efficient in terms of preprocessing; no resampling to a larger Cartesian grid is required, avoiding the memory bloat associated with uniform grid resampling [S1, book].
*   **Performance:** Requires powerful hardware and specific WebGL implementations (e.g., Firefox with native OpenGL, Opera) to compile and run the complex shaders efficiently [S1, book]. Not feasible for real-time use on older or less capable hardware.
*   **Strengths:** Preserves original data detail without resampling artifacts; avoids gaps associated with conic surface approximations [S1, book].
*   **Limitations:** Computationally intensive; dependent on specific browser/hardware capabilities for WebGL shader compilation [S1, book].

## Background
The method was developed for real-time interactive weather radar data visualization using WebGL [S1, book]. It addresses issues identified in previous approaches where resampling caused memory/detail trade-offs and textured conic surfaces left geometric gaps [S1, book]. The specific implementation details and formulas are documented in a book chapter by John Edgar Congote et al. [S1, book]. A related concept for curvilinear grids is discussed in research on volume rendering for unstructured grids, which highlights the benefits of using spherical coordinates to avoid Cartesian-to-geodetic conversion errors [S2, unofficial].

## Sources
[S1] Volume Ray Casting in WebGL, https://www.vicomtech.org/upload/download/publicaciones/20120330_JohnEdgarCongote_ComputerG_inbook_14.pdf, fetched 2026-10-29 (book)
[S2] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
