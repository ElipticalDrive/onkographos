# BCC Grid Rendering

Also called: BCC vs CC Grid Rendering, Body-Centered Cartesian Grid

## Overview
BCC Grid Rendering is a volume rendering technique that utilizes a Body-Centered Cartesian (BCC) lattice for spatial sampling instead of the standard Cubic Cartesian (CC) grid [S2, unofficial]. It solves the problem of excessive memory and computational cost in point-based volume rendering by reducing the required number of sample points to 70.3% of a CC grid while maintaining lossless compression for spherically bandlimited signals [S2, unofficial]. It is distinct from CC rendering by its higher packing efficiency for isotropic signals [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset defined on a BCC lattice, camera parameters (view and projection matrices), and a rendering kernel [S2, unofficial].
*   **Output:** A 2D raster image.

**Data Structures:**
The data is stored in a BCC lattice structure. A BCC lattice can be viewed as two interleaved simple cubic lattices, or as a cubic lattice with an additional point at the center of each cube [G].
*   **Lattice Points:** Defined by integer coordinates $(i, j, k)$ and a parity bit or offset indicating whether the point belongs to the primary cubic sublattice or the body-centered sublattice [G].
*   **Attributes:** Each lattice point stores density/extinction coefficient $\rho$ and potentially other scalar/vector fields [G].

**Steps:**
1.  **Lattice Generation:** Construct the BCC grid from the input data. If the input is on a CC grid, resample or interpolate onto the BCC lattice. Not given in the sources.
2.  **Transformation:** Transform the BCC lattice points from world space to clip space using the standard Model-View-Projection (MVP) matrix, identical to CC rendering [G].
3.  **Slicing/Sorting:** Sort the transformed points by depth (Z-buffer) to ensure correct front-to-back or back-to-front compositing [G].
4.  **Rasterization (Splatting):** For each point, project its 3D footprint onto the 2D image plane.
    *   **Shift:** When rendering BCC grids, slices must be shifted by $1/2$ relative to the CC grid alignment to account for the body-centered offset [S2, unofficial].
    *   **Footprint:** Compute the 2D footprint $F(x,y)$ for each point. The evidence states that BCC grids assume spherically bandlimited signals, under which assumption compression is lossless [S2, unofficial].
    *   **Weighting:** Calculate the weight based on the distance from the pixel center to the point's projected center, modulated by the point's density $\rho$ [G].
5.  **Compositing:** Accumulate the splatted contributions into the frame buffer using alpha blending [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_bcc_grid(points, camera, image_width, image_height):
    """
    points: List of BCC lattice points. Each point has:
        - pos: (x, y, z) in world space
        - density: scalar value
        - type: 'primary' or 'center' (indicating BCC sublattice)
    camera: Object with view_proj_matrix and shift_offset (0.5 for BCC)
    """
    image = [[0.0, 0.0, 0.0, 0.0] for _ in range(image_height)]
    
    transformed_points = []
    for p in points:
        clip_pos = camera.view_proj_matrix * p.pos
        if clip_pos.w > 0:
            transformed_points.append({'pos': clip_pos, 'density': p.density, 'type': p.type})
    
    transformed_points.sort(key=lambda p: p.pos.z / p.pos.w)
    
    for p in transformed_points:
        sx = (p.pos.x / p.pos.w + 1) * 0.5 * image_width
        sy = (1 - (p.pos.y / p.pos.w + 1) * 0.5) * image_height
        
        sx += camera.shift_offset
        sy += camera.shift_offset
        
        radius = max(1, int(3.0 * p.density))
        
        for dy in range(-radius, radius + 1):
            for dx in range(-radius, radius + 1):
                px = int(sx + dx)
                py = int(sy + dy)
                
                if 0 <= px < image_width and 0 <= py < image_height:
                    dist_sq = dx*dx + dy*dy
                    weight = max(0.0, 1.0 - dist_sq / (radius * radius))
                    
                    alpha = weight * p.density
                    
                    dst_alpha = image[py][px][3]
                    new_alpha = alpha + dst_alpha * (1 - alpha)
                    if new_alpha > 0:
                        src_color = (0.5, 0.5, 0.5)
                        image[py][px][0] = (src_color[0] * alpha + image[py][px][0] * dst_alpha * (1 - alpha)) / new_alpha
                        image[py][px][1] = (src_color[1] * alpha + image[py][px][1] * dst_alpha * (1 - alpha)) / new_alpha
                        image[py][px][2] = (src_color[2] * alpha + image[py][px][2] * dst_alpha * (1 - alpha)) / new_alpha
                        image[py][px][3] = new_alpha

    return image
```

**Parameters:**
*   **Shift Offset:** $0.5$ in normalized screen coordinates or lattice units [S2, unofficial].
*   **Kernel:** Spherical/Gaussian, optimized for isotropic signals [G].

**Edge Cases:**
*   **Non-Spherical Signals:** If the signal is not spherically bandlimited, the BCC grid may not provide lossless compression, and artifacts may appear [S2, unofficial].
*   **Boundary Conditions:** Not given in the sources.

## Subtopics

### [BCC Grid Sampling](13572-bcc-grid-sampling.md)
BCC Grid Sampling uses a Body-Centered Cartesian (BCC) lattice to store volume data, requiring fewer sample points than a standard Cubic Cartesian (CC) grid while maintaining lossless compression for spherically bandlimited signals. It is distinct from CC sampling by its higher packing efficiency for isotropic signals.

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to the number of sample points. Since BCC reduces the number of required points to 70.3% of a CC grid, the theoretical rendering time is reduced by a similar factor, assuming constant cost per point [S2, unofficial].
*   **Memory Complexity:** Memory usage is reduced by approximately 29.7% compared to CC grids for the same spatial resolution and signal fidelity [S2, unofficial].
*   **Measured Performance:** In the Turbulent Jet 4D example, BCC rendering took 1.01s (71% of CC time) with 138 time steps, compared to 1.23s for CC with 99 time steps [S2, unofficial]. The BCC grid had 87M total points vs 168M for CC, but only 7.4M relevant voxels vs 9.4M for CC [S2, unofficial]. For the Foot dataset, BCC rendering took 70% of the time of CC rendering, although renderings were almost identical [S1, book].
*   **Strengths:** Significant reduction in memory and computation for isotropic, spherically bandlimited signals. Lossless compression under the spherical bandlimit assumption [S2, unofficial].
*   **Limitations:** Only optimal for spherically bandlimited signals. Requires careful handling of the 1/2 shift during rasterization. Not given in the sources.
*   **When to Use:** Point-based volume rendering of isotropic data where memory and performance are critical. Not given in the sources.
*   **When Not to Use:** Non-isotropic signals, or when compatibility with standard CC-based tools is required. Not given in the sources.

## Background
The BCC grid was introduced as an alternative to the Cubic Cartesian (CC) grid for volumetric data representation. The concept is based on the work of Theussl ('01) and Neophytou ('02) [S2, unofficial]. The BCC lattice is known in mathematics and signal processing for its optimal packing density for spheres, which translates to fewer sample points needed for reconstruction of bandlimited signals [S2, unofficial]. It has been applied to time-varying datasets such as turbulent jets and flows [S2, unofficial].

## Sources
[S1] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
[S2] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
