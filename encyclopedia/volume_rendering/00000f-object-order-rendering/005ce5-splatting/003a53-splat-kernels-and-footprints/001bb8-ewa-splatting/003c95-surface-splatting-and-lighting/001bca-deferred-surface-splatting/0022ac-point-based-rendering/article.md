# Point-Based Rendering

Also called: Lines and points, Particle systems, Particle systems / Point-based rendering, Point Cloud Surfel Estimation, QSplat, Surfels

## Overview
Point-Based Rendering (PBR) is a rendering technique that approximates continuous surfaces or dense volumetric distributions using a set of discrete points (splats) rather than explicit polygonal meshes [S1][S4, unofficial]. It solves the problem of rendering large, unstructured point clouds or implicit surfaces efficiently by rasterizing these points as disc-like primitives [S4, unofficial]. Unlike its sibling [Deferred Surface Splatting](1072-deferred-surface-splatting.md), which focuses on deferred shading pipelines for surface normals and depth, standard PBR often relies on opaque primitives to avoid sorting costs, though this introduces specific aliasing trade-offs [S1]. It builds on the concept of particle systems introduced by Reeves [S1] and was formalized for large point-set data by Rusinkiewicz and Levoy [S4, unofficial].

## How it works
**Inputs:** A set of 3D points $P = \{p_i\}$, where each point $p_i$ includes position $\mathbf{x}_i$, normal $\mathbf{n}_i$, and color/weight attributes [S4, unofficial]. These points are usually generated via an offline sampling process of an implicit surface or mesh [S4, unofficial].

**Outputs:** A 2D image frame buffer.

**Data Structures:**
*   **Point Buffer:** An array of structures containing:
    *   `position`: 3D vector $(x, y, z)$.
    *   `normal`: 3D vector $(n_x, n_y, n_z)$.
    *   `color`: 3D vector $(r, g, b)$ or scalar weight.
    *   `radius`: Scalar defining the splat size [G].

**Steps:**
1.  **Sampling (Offline):** Sample the target geometry (implicit surface or mesh) into a dense point cloud [S4, unofficial]. Adaptive sampling may be used, guided by curvature or energy redistribution to ensure point density correlates with surface detail [S4, unofficial].
2.  **Projection:** Transform each point from world space to clip space using the view and projection matrices [G].
3.  **Rasterization:** Project each point onto the screen as a disc (splat) [S4, unofficial]. The radius of the disc is adapted to the distance from the camera to maintain roughly pixel-sized coverage [S1]. The screen-space radius is computed as $r = \max(1, \lfloor \alpha \cdot z_{clip} \rfloor)$, where $\alpha$ is a scaling constant and $z_{clip}$ is the homogeneous depth [G].
4.  **Shading/Blending:** Shade the splats. To avoid the cost of sorting transparent splats, primitives are often rendered as opaque [S1]. If blending is required, ray-casting techniques may be employed to blend between splats or compute shading based on the underlying implicit function [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sample_surface(implicit_fn, bounds, step_size):
    """Offline sampling of an implicit surface into a point cloud.
    Sources do not specify the exact sampling algorithm; grid sampling is shown.
    Adaptive or curvature-guided energy redistribution can replace the uniform grid [S4, unofficial]."""
    points = []
    x_min, x_max, y_min, y_max, z_min, z_max = bounds
    x = x_min
    while x <= x_max:
        y = y_min
        while y <= y_max:
            z = z_min
            while z <= z_max:
                pos = (x, y, z)
                val = implicit_fn(pos)
                if abs(val) < 0.01:
                    eps = 0.001
                    gx = (implicit_fn((x+eps, y, z)) - implicit_fn((x-eps, y, z))) / (2*eps)
                    gy = (implicit_fn((x, y+eps, z)) - implicit_fn((x, y-eps, z))) / (2*eps)
                    gz = (implicit_fn((x, y, z+eps)) - implicit_fn((x, y, z-eps))) / (2*eps)
                    norm_len = math.sqrt(gx*gx + gy*gy + gz*gz)
                    if norm_len > 0:
                        norm = (gx/norm_len, gy/norm_len, gz/norm_len)
                    else:
                        norm = (0.0, 0.0, 1.0)
                    points.append({'pos': pos, 'normal': norm, 'color': (0.5, 0.5, 0.5)})
                z += step_size
            y += step_size
        x += step_size
    return points

def render_points(points, view_proj_matrix, width, height):
    """Project and rasterize points as disc splats."""
    frame_buffer = [[(0.0, 0.0, 0.0, 1.0) for _ in range(width)] for _ in range(height)]
    depth_buffer = [[float('inf') for _ in range(width)] for _ in range(height)]

    for pt in points:
        clip_pos = transform_clip(pt['pos'], view_proj_matrix)
        if clip_pos[3] <= 0.0:
            continue

        ndc_x = clip_pos[0] / clip_pos[3]
        ndc_y = clip_pos[1] / clip_pos[3]
        ndc_z = clip_pos[2] / clip_pos[3]

        sx = int((ndc_x + 1.0) * width / 2.0)
        sy = int((1.0 - ndc_y) * height / 2.0)

        # Adaptive radius: roughly pixel-sized based on depth [S1]
        radius = max(1, int(2.0 * clip_pos[3]))

        x_min = max(0, sx - radius)
        x_max = min(width, sx + radius)
        y_min = max(0, sy - radius)
        y_max = min(height, sy + radius)

        for y in range(y_min, y_max):
            for x in range(x_min, x_max):
                dx = x - sx
                dy = y - sy
                if dx*dx + dy*dy <= radius*radius:
                    if ndc_z < depth_buffer[y][x]:
                        depth_buffer[y][x] = ndc_z
                        frame_buffer[y][x] = pt['color'] + (1.0,)
    return frame_buffer

def transform_clip(pos, matrix):
    """Standard 4x4 matrix-vector multiplication for clip space transform."""
    x, y, z = pos
    w = matrix[3]*x + matrix[7]*y + matrix[11]*z + matrix[15]
    x_clip = matrix[0]*x + matrix[4]*y + matrix[8]*z + matrix[12]
    y_clip = matrix[1]*x + matrix[5]*y + matrix[9]*z + matrix[13]
    z_clip = matrix[2]*x + matrix[6]*y + matrix[10]*z + matrix[14]
    return (x_clip, y_clip, z_clip, w)
```

**Parameters:**
*   **Sampling Density:** Determines the number of points, directly impacting vertex transform costs [S1].
*   **Splat Radius:** Controls the visual size of points, typically adapted to distance to maintain roughly pixel-sized coverage [S1].

**Edge Cases and Pitfalls:**
*   **Popping:** Distance adaptation is often implemented by suppressing (culling) points, which can cause visual "popping" artifacts as points appear or disappear [S1].
*   **Performance Bottleneck:** Vertex transform costs are significant, limiting performance for huge sparse geometry [S1].
*   **Antialiasing:** Primitives are generally made opaque to avoid sorting and blending costs, which prevents proper antialiasing [S1].

## Subtopics

### [Iso-splatting](12054-iso-splatting.md)
Iso-splatting is a point-based alternative to isosurface visualization that generates point samples per cell instead of triangles, projecting them onto the isosurface using function values and gradients. It splits extraction into two independent steps: generating point samples throughout the volumetric domain, and projecting these points onto the isosurface of interest, followed by rendering using a surface splatting algorithm. Unlike standard polygon-based methods where geometry is discarded and recomputed for each new isovalue, iso-splatting uses point geometry that can represent a range of isosurfaces, requiring only small adjustments for slightly different isovalue, thereby decreasing computation time and storage. It builds on [Point-Based Rendering](2827-point-based-rendering.md) by utilizing surface splatting kernels to render the projected points.

### GPU Point-Based Stream Surface
A point-based algorithm that does not compute an explicit mesh representation but uses a very dense set of particles advected at interactive speeds, combined with point-based rendering [S5]. It advects a very dense set of particles at interactive speeds and renders the particles using point-based rendering techniques [S5]. Presented by Schafhitzel et al. using a GPU-based approach [S5].

## Operating characteristics
*   **Time Complexity:** Dominated by vertex transformation and rasterization [S1]. Performance is limited by the vertex transform rate, which is significantly slower than fill rate on many GPUs [S1].
*   **Memory Complexity:** Linear in the number of points $N$ [G].
*   **Measured Performance:** At 60Hz and 1280x1024 resolution, textured polygons allow redrawing each pixel ~20 times, whereas points allow only once due to the vertex transform bottleneck [S1].
*   **Strengths:** Efficient for rendering large, unstructured point clouds; avoids the overhead of mesh generation and sorting for transparent objects; suitable for implicit surfaces [S4, unofficial].
*   **Limitations:** Poor antialiasing if opaque; popping artifacts from distance-based culling; expensive offline sampling process [S1][S4, unofficial].
*   **When to Use:** Rendering dense point clouds, vegetation (forests), or implicit surfaces where mesh generation is prohibitive [S1][S4, unofficial].
*   **When Not to Use:** When high-quality antialiasing is required without sorting, or when the geometry is sparse and occlusion culling is critical [S1].

## Background
Point-based rendering was conceived by Levoy and Whitted [S4, unofficial]. It was proven for large point-set data by Rusinkiewicz and Levoy with their QSplat software [S4, unofficial]. The concept of procedurally drawing simple primitives (particle systems) was introduced by Reeves in 1983 [S1]. The method was further developed for real-time rendering of vegetation and dense scenes by researchers such as Pfister et al. [PZvBG00] and others [S1].

## Sources
[S1] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S3] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
[S4] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S5] IRIS: Illustrative Rendering for Integral Surfaces, https://doi.org/10.1109/tvcg.2010.173, fetched 2026-10-27
