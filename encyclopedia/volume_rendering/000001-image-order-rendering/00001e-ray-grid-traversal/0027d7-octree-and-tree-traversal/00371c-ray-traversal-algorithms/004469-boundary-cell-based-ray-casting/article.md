# Boundary Cell-Based Ray Casting

Also called: BC-RC, Improved PARC

## Overview
Boundary Cell-Based Ray Casting (BC-RC) is a fast serial ray casting algorithm that accelerates volume rendering by skipping empty space using boundary cells [S1, unofficial]. It improves upon the PARC algorithm by using tiny grid cells for more accurate boundary estimation and employing fast incremental parallel projection to reduce projection overhead [S1, unofficial]. Unlike PARC, which uses pre-calculated sub-volumes whose projection is expensive when small, BC-RC projects only cells containing the object boundary [S1, unofficial]. The method builds on the hierarchical descent and ascent principles of [Ray Traversal Algorithms](7918-ray-traversal-algorithms.md) but replaces subtree traversal with a two-pass projection-and-traversal pipeline.

## How it works
**Inputs:**
- A regular grid of volume elements (voxels) with scalar values [S1, unofficial].
- Camera parameters defining parallel projection geometry and image dimensions [S1, unofficial].

**Outputs:**
- A rendered image with ray casting optimizations (adaptive sampling, early termination) [S1, unofficial].

**Data Structures:**
1.  **Volume Grid:** Regular grid of voxels [S1, unofficial].
2.  **Boundary Flags:** A binary flag $F$ for each cell $C(i, j, k)$ [S1, unofficial].
    -   $F=1$: The cell contains the object boundary [S1, unofficial].
    -   $F=0$: The cell is empty or fully inside/outside (not a boundary cell) [S1, unofficial].
3.  **Z-Buffers:**
    -   $Z_n$: Front z-buffer. Stores the closest intersection distance to the object boundary for each pixel [S1, unofficial].
    -   $Z_r$: Rear z-buffer. Stores the farthest intersection distance to the object boundary for each pixel [S1, unofficial].

**Algorithm Steps:**

1.  **Boundary Cell Detection (Preprocessing):**
    Scan the volume in an off-line stage. For each cell $C(i, j, k)$, determine if it contains the object boundary. If yes, set its flag $F=1$ [S1, unofficial].

2.  **Boundary Cell Projection:**
    Project only cells with $F=1$ onto the image plane using incremental parallel projection [S1, unofficial].
    -   Project the first cell $C(0,0,0)$ to establish the base position and depth [S1, unofficial].
    -   Calculate projected positions and depths of remaining boundary cells using incremental vectors (addition only) [S1, unofficial].
    -   Update $Z_n$ and $Z_r$ for each pixel based on the depth of the projected boundary cells. $Z_n$ stores the minimum depth, $Z_r$ stores the maximum depth [S1, unofficial].

3.  **Ray Traversal:**
    Cast rays only from viable pixels where $Z_n$ and $Z_r$ indicate an intersection (i.e., $Z_n < Z_r$ and within bounds) [S1, unofficial].
    -   Traverse each ray from the closest intersection ($Z_n$) to the farthest intersection ($Z_r$) [S1, unofficial].
    -   Sample and composite the volume data starting from the intersection [S1, unofficial].
    -   Incorporate adaptive image sampling and early ray termination [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def boundary_cell_based_ray_casting(volume, camera_params):
    # Inputs: volume (3D array), camera_params (image_size, projection_step, project_3d_to_2d, get_ray_origin, get_ray_direction)
    # Step 1: Boundary Cell Detection
    num_cells_x, num_cells_y, num_cells_z = volume.shape
    boundary_flags = np.zeros((num_cells_x, num_cells_y, num_cells_z), dtype=bool)

    # [G] Standard boundary detection: flag cell if adjacent cells have different values
    for i in range(num_cells_x):
        for j in range(num_cells_y):
            for k in range(num_cells_z):
                is_boundary = False
                for di, dj, dk in [(-1,0,0), (1,0,0), (0,-1,0), (0,1,0), (0,0,-1), (0,0,1)]:
                    ni, nj, nk = i+di, j+dj, k+dk
                    if 0 <= ni < num_cells_x and 0 <= nj < num_cells_y and 0 <= nk < num_cells_z:
                        if volume[i,j,k] != volume[ni,nj,nk]:
                            is_boundary = True
                            break
                if is_boundary:
                    boundary_flags[i, j, k] = True

    # Step 2: Boundary Cell Projection
    image_width, image_height = camera_params.image_size
    Z_n = np.full((image_height, image_width), np.inf)
    Z_r = np.full((image_height, image_width), -np.inf)

    # Incremental projection parameters from camera setup
    du, dv, dd = camera_params.incremental_projection_step

    prev_u, prev_v, prev_d = None, None, None

    for i in range(num_cells_x):
        for j in range(num_cells_y):
            for k in range(num_cells_z):
                if boundary_flags[i, j, k]:
                    if prev_u is None:
                        # Project first boundary cell C(0,0,0) or first found cell
                        center_x, center_y, center_z = i + 0.5, j + 0.5, k + 0.5
                        prev_u, prev_v, prev_d = camera_params.project_3d_to_2d(center_x, center_y, center_z)

                    u, v, d = prev_u, prev_v, prev_d

                    # Update Z-buffers if within image bounds
                    ix, iy = int(round(u)), int(round(v))
                    if 0 <= ix < image_width and 0 <= iy < image_height:
                        if d < Z_n[iy, ix]:
                            Z_n[iy, ix] = d
                        if d > Z_r[iy, ix]:
                            Z_r[iy, ix] = d

                    # Incremental update for next boundary cell
                    prev_u += du
                    prev_v += dv
                    prev_d += dd

    # Step 3: Ray Traversal
    image = np.zeros((image_height, image_width, 3))

    for y in range(image_height):
        for x in range(image_width):
            if Z_n[y, x] < Z_r[y, x]: # Viable pixel
                t_start = Z_n[y, x]
                t_end = Z_r[y, x]
                ray_origin = camera_params.get_ray_origin(x, y)
                ray_direction = camera_params.get_ray_direction(x, y)
                color = [0.0, 0.0, 0.0]
                alpha = 0.0
                step = 0.1 # [G] Fixed step size for traversal
                t = t_start
                while t < t_end:
                    sample_pos = ray_origin + t * ray_direction
                    val = volume.sample(sample_pos) # [G] Trilinear interpolation
                    color, alpha = composite(val, color, alpha) # [G] Front-to-back compositing
                    if alpha > 0.95: break # Early ray termination
                    t += step
                image[y, x] = color

    return image

def composite(sample_val, color, alpha):
    # [G] Standard front-to-back volume compositing
    c, a = sample_val.color, sample_val.alpha
    new_alpha = alpha + (1.0 - alpha) * a
    new_color = color + c * (1.0 - alpha)
    return new_color, new_alpha
```

## Subtopics

## Operating characteristics
-   **Time Complexity:** Not given in the sources.
-   **Memory Complexity:** Not given in the sources.
-   **Strengths:** Fast serial rendering; accurate boundary estimation; reduced projection overhead compared to PARC; compatible with existing ray casting optimizations (adaptive sampling, early termination) [S1, unofficial].
-   **Limitations:** Requires off-line preprocessing; serial algorithm (though a parallel version was proposed) [S1, unofficial].

## Background
Boundary Cell-Based Ray Casting was proposed as an improvement on the PARC algorithm [S1, unofficial]. It was developed to address the computational overhead of projecting pre-calculated sub-volumes in PARC, which becomes expensive when the sub-volumes are small [S1, unofficial]. The algorithm was part of research into time-critical volume rendering, including the development of hierarchical level-of-detail representations and user interfaces for interactive exploration [S1, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19990040407: Time-Critical Volume Rendering, https://archive.org/details/NASA_NTRS_Archive_19990040407, fetched 2026-10-26 (unofficial)
