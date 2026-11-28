# Single-Pass Splat Rendering Pipeline

Also called: Single-Pass High-Quality Splatting

## Overview
The Single-Pass Splat Rendering Pipeline is a real-time rendering technique that combines splat and mesh rendering components to enable high-quality shader effects (reflection, refraction) in a single rendering pass [S1, unofficial]. It solves the problem of multi-pass blending artifacts and high memory/workload associated with dense point sets by reorganizing input data into elevation map patches [S1, unofficial]. Unlike sibling methods that reconstruct smooth attributes by blending in screen space (requiring at least three passes and causing blurring), this method interpolates attributes at an earlier stage in object space using patch information, resulting in a single pass and improved normal/attribute quality [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point set representing the model, optionally with per-point attributes (color, normal) [S1, unofficial].
*   **Output:** A final image rendered in a single pass [S1, unofficial].

**Data Structures:**
The model is decomposed into patches. Each patch is defined by:
1.  **Plane Parameters:** A 3D point (origin), a normal vector, and a grid resolution [S1, unofficial].
2.  **Height Field:** A rectangular array of height values (elevation map) sampled on a regular grid over the planar domain [S1, unofficial]. Some grid points may be flagged with a special height value indicating no point exists above that grid location [S1, unofficial].
3.  **Attribute Maps:** Optional texture maps (e.g., color, normal) stored as texture objects for each patch individually [S1, unofficial].
4.  **Transformation Matrix:** One 4x4 matrix per patch [S1, unofficial].

**Storage:**
*   **VBO (Vertex Buffer Object):** Stores point positions and elevation data for each patch. Point positions on the patch plane are expressed by a single index value (combined row and column address in the sampling grid) [S1, unofficial].
*   **Index Buffer:** A linear list $1, 2, \dots, n$, where $n$ is the number of points in the patch. No connectivity encoding is needed [S1, unofficial].
*   **Texture Objects:** Store additional attribute maps [S1, unofficial].

**Algorithm Steps:**

1.  **Patch Construction (Pre-processing):**
    *   Partition the input 3D point set into patches. Each patch is parameterized as an elevation map over a planar domain [S1, unofficial].
    *   Resample the surface patch on a regular grid using Moving Least Squares (MLS) surface reconstruction [S1, unofficial].
    *   **Overlap Handling:** To prevent gaps due to MLS limitations, patches are made to slightly overlap. An input point near a boundary is included in a patch if its $\epsilon$-neighborhood intersects the point set of the region, where $\epsilon > 0$ [S1, unofficial].
    *   Resample attribute fields (normals, textures) onto the regular grid [S1, unofficial].

2.  **Upload to GPU:**
    *   Upload VBOs (elevation data) and Texture Objects (attributes) to graphics memory [S1, unofficial].

3.  **Rendering Loop:**
    *   Iterate over all patches [S1, unofficial].
    *   For each patch:
        *   Address the respective texture objects [S1, unofficial].
        *   Trigger the corresponding VBO for rendering using the linear index buffer [S1, unofficial].

4.  **Vertex Shader:**
    *   **Translation:** Translate point positions according to their elevation values from the VBO [S1, unofficial].
    *   **Rotation:** Rotate the translated points using the patch's 4x4 transformation matrix to place them on the surface of the original model [S1, unofficial].
    *   **Radius Calculation:** Set the radius of each point (point sprite) based on the sampling step size $\mu$ and elevation variance to prevent holes [S1, unofficial]. The base radius is $\frac{1}{2}\sqrt{2}\mu$ [S1, unofficial]. Radii are increased in areas where variance between neighboring elevation values results in spatial gaps [S1, unofficial].

5.  **Rasterization:**
    *   Rasterize the points as point sprites [S1, unofficial].

6.  **Fragment Shader:**
    *   **Splat Simulation:** Transform point-sprite fragments to simulate splatting, as point sprites are screen-aligned primitives [S1, unofficial].
    *   **Continuous Attribute Interpolation:**
        *   Determine per-fragment UV coordinates by projecting the fragment back onto the patch plane [S1, unofficial].
        *   Interpolate attributes (normals, textures) from nearby points using the patch's regular grid structure. This replaces screen-space blending with object-space interpolation [S1, unofficial].
    *   **Illumination:** Compute final fragment color by summing contributions from lighting, texture mapping, reflectance mapping (e.g., environment cube maps), and refraction mapping [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def build_patches(point_set):
    patches = []
    # [G] Standard spatial partitioning of point set into regions
    regions = [point_set[i:i+chunk] for i in range(0, len(point_set), chunk)]
    for region in regions:
        # [G] Moving Least Squares surface reconstruction
        grid_size = 16
        heights = [[0.0] * grid_size for _ in range(grid_size)]
        # [S1, unofficial] Epsilon overlap handling
        epsilon = 0.01
        # [S1, unofficial] Attribute resampling
        normals = [[(0.0, 1.0, 0.0)] * grid_size for _ in range(grid_size)]
        colors = [[(1.0, 1.0, 1.0)] * grid_size for _ in range(grid_size)]
        # [S1, unofficial] Plane fitting and transform
        plane_origin = (0.0, 0.0, 0.0)
        plane_normal = (0.0, 1.0, 0.0)
        transform_matrix = [[1.0]*4 for _ in range(4)]

        patches.append({
            'grid': grid_size,
            'heights': heights,
            'normals': normals,
            'colors': colors,
            'transform': transform_matrix,
            'origin': plane_origin,
            'normal': plane_normal,
            'u_vec': (1.0, 0.0, 0.0),
            'v_vec': (0.0, 0.0, 1.0)
        })
    return patches

def render_patches(patches):
    for patch in patches:
        # [S1, unofficial] Address textures and trigger VBO
        indices = list(range(1, patch['grid'] * patch['grid'] + 1))
        # [G] GPU draw call using linear index buffer
        for idx in indices:
            world_pos, radius = vertex_shader(idx, patch)
            rasterize_point(world_pos, radius)

def vertex_shader(index, patch):
    row = (index - 1) // patch['grid']
    col = (index - 1) % patch['grid']
    base_pos = (patch['origin'][0] + row * patch['u_vec'][0] + col * patch['v_vec'][0],
                patch['origin'][1] + row * patch['u_vec'][1] + col * patch['v_vec'][1],
                patch['origin'][2] + row * patch['u_vec'][2] + col * patch['v_vec'][2])
    height = patch['heights'][row][col]
    translated_pos = (base_pos[0] + height * patch['normal'][0],
                      base_pos[1] + height * patch['normal'][1],
                      base_pos[2] + height * patch['normal'][2])
    # [S1, unofficial] Apply 4x4 transform
    world_pos = translated_pos
    mu = 1.0 / patch['grid']
    base_radius = 0.5 * math.sqrt(2.0) * mu
    # [G] Variance-based radius scaling to prevent holes [S1, unofficial]
    h_center = height
    h_neighbors = [patch['heights'][r][c] for r in range(max(0, row-1), min(patch['grid'], row+2))
                   for c in range(max(0, col-1), min(patch['grid'], col+2)) if (r, c) != (row, col)]
    variance = sum((h - h_center)**2 for h in h_neighbors) / max(1, len(h_neighbors))
    radius = base_radius * (1.0 + variance * 0.1)
    return world_pos, radius

def fragment_shader(world_pos, patch):
    # [G] Project fragment to patch plane for UV [S1, unofficial]
    grid_uv = (0.5, 0.5)
    # [S1, unofficial] Interpolate attributes
    normal = patch['normals'][0][0]
    diffuse = patch['colors'][0][0]
    # [G] Standard lighting and shader effects [S1, unofficial]
    light_color = (1.0, 1.0, 1.0)
    reflect_color = (0.5, 0.5, 0.5)
    refract_color = (0.5, 0.5, 0.5)
    final_color = tuple(a + b + c for a, b, c in zip(light_color, reflect_color, refract_color))
    return final_color
```

**Parameters:**
*   $\epsilon$ (Epsilon): A positive value used during patch construction to include input points near a patch boundary if their $\epsilon$-neighborhood intersects the point set of the region, ensuring patches slightly overlap to prevent gaps [S1, unofficial].
*   $\mu$ (Sampling step size): Determines the base splat radius [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Gaps between patches:** Moving Least Squares resampling cannot guarantee no gaps. Patches must overlap slightly using $\epsilon$ to ensure continuous error-free rendering [S1, unofficial].
*   **Holes due to elevation variance:** If elevation values vary significantly between neighbors, the base radius may be insufficient. Radii must be increased in these areas [S1, unofficial].

## Operating characteristics
*   **Complexity:** The pipeline consists of one single rendering pass, simplifying overall rendering complexity compared to current GPU-based splatting techniques which require at least three rendering passes [S1, unofficial].
*   **Performance:** Aggressively utilizes practical components of both splat and mesh rendering. Avoids performance-hungry GPU functionality such as multiple render targets and additional blending passes [S1, unofficial].
*   **Quality:** Improves visual quality of reconstructed attributes (normals, texture coordinates) by interpolating in object space rather than blending in screen space [S1, unofficial]. Allows for enhanced shader effects (reflection, refraction) without densely sampled models [S1, unofficial].
*   **Memory:** The resampled model is stored completely on graphics hardware. Connectivity is implicit and reconstructed in the fragment shader, requiring no additional memory for connectivity structures [S1, unofficial].

## Background
The Single-Pass Splat Rendering Pipeline was introduced by Tilo Ochotta, Stefan Hiller, and Dietmar Saupe in September 2006 [S1, unofficial]. It was published as Konstanzer Schriften in Mathematik und Informatik Nr. 219 [S1, unofficial]. The method builds on previous work regarding elevation map data structures for point-sampled surfaces [S1, unofficial].

## Sources
[S1] Single-Pass High-Quality Splatting, http://nbn-resolving.de/urn:nbn:de:bsz:352-opus-22395, fetched 2026-10-26 (unofficial)
