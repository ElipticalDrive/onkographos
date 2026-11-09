# Interactive Volume Illustration (Hatching)

Also called: Hatching, Object space hatching, Stroke-based volume rendering, Two-Pass Hatching Rendering

## Overview
Interactive Volume Illustration (Hatching) is an object-space technique for illustrating volumetric data by simulating free-hand line art (hatching) using hardware-assisted volume and line rendering [S2, unofficial][S1]. It highlights curved areas by tracing hatches as polylines over the implicit surface along either a fixed direction or a smoothed principal curvature direction [S1]. It differs from object- or geometry-based silhouette approaches by not extracting edges of a polygonal mesh, but instead computing silhouettes per-fragment for volumetric data using normal maps and view vectors [S2, unofficial].

## How it works
The method operates in two main phases: stroke generation (preprocessing/runtime) and two-pass rendering [S2, unofficial].

### Inputs and Outputs
*   **Inputs:** 3D scalar volume data, transfer function parameters, light direction $L$, view direction $V_{dir}$, user-defined fixed direction $\vec{s}_T$, smoothing weight $w_T$, flatness threshold $\epsilon$, and cross-hatch angle threshold [S1][S2, unofficial].
*   **Outputs:** A rendered image combining hatching strokes (line strips) and volumetric data (3D texture splatting) [S2, unofficial].

### Data Structures
1.  **Volume Data:** 3D texture map $T_V$ storing scalar values [S2, unofficial].
2.  **Gradient/Normal Map:** 3D texture map $T_N$ storing normalized gradients $\nabla f_n$ (surface normals) [S2, unofficial].
3.  **Curvature Data:** Precomputed or on-demand principal curvature directions $\vec{\kappa}_{1j}$ and magnitudes $\kappa_1, \kappa_2$ for seed points [S1].
4.  **Stroke Buffer:** A texture buffer or vertex buffer storing line-strip primitives. Each stroke consists of a sequence of vertices. Memory usage is at most one-eighth of the volume data set size [S2, unofficial].
5.  **Seed Points:** A set of 3D positions representing iso-surfaces, selected at runtime to ensure an optimal number of points [S2, unofficial].

### Preprocessing: Curvature Estimation
To guide hatches along principal curvature, partial derivatives are reconstructed from volume data using Sigg and Hadwiger's fast cubic B-spline filtering approach [S1].
1.  Compute partial derivatives $\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z}$ at each voxel [S1].
2.  Compute the shape index $s$ and principal curvatures $\kappa_1, \kappa_2$ (where $\kappa_1 \ge \kappa_2$) [S1].
    $$ s = \frac{2}{\pi} \arctan\left(\frac{\kappa_1 + \kappa_2}{\kappa_2 - \kappa_1}\right) $$
    Condition: $\kappa_1 \ge \kappa_2$ and $|\kappa_1| + |\kappa_2| > 0$ [S1].
3.  Compute surface reliability $\rho(\kappa_1, \kappa_2)$ [S1]:
    $$ \rho(\kappa_1, \kappa_2) = \begin{cases} 0 & \text{if } |\kappa_1| < \epsilon \text{ and } |\kappa_2| < \epsilon \\ 1 - |2(|s| - 0.5)| & \text{otherwise} \end{cases} $$
    Here $\epsilon$ defines which nearly flat surfaces are considered flat [S1].

### Stroke Generation (Object Space)
Hatches are represented by line-strip primitives. The geometry shader thread determines vertices connecting hatch segments by projecting the direction to the tangent plane [S1].

1.  **Seed Selection:** Select seed points at runtime to represent iso-surfaces [S2, unofficial].
2.  **Direction Calculation:** For each seed point $p_j$, determine the hatch direction $\vec{s}_i$. Directly tracing along principal curvature directions yields messy results when the main direction is not robustly defined [S1]. Therefore, smooth the curvature field by computing a weighted average of neighboring particles' principal curvature directions $\vec{\kappa}_{1j}$ [S1]:
    $$ \vec{s}_i = w_T \frac{\sum_j \rho_j \vec{\kappa}_{1j}}{\sum_j \rho_j} + (1 - w_T) \vec{s}_T $$
    Where:
    *   $\vec{\kappa}_{1j}$ is the main principal curvature direction of neighbor $j$.
    *   $\rho_j$ is the surface reliability of neighbor $j$.
    *   $\vec{s}_T$ is the user-defined fixed direction.
    *   $w_T$ is the trace reliability weight, determined by averaging the surface reliability of all neighboring particles:
        $$ w_T = \frac{\sum_j \rho_j}{\sum_j 1} $$
    Smoothing is executed on the GPU, estimating curvature information of neighboring particles on the fly [S1].
3.  **Tracing:** Trace the hatch stroke along the smoothed direction $\vec{s}_i$. Use a simple weighting scheme based on the number of segments from the seed point. Long strokes may intersect [S1].
4.  **Cross-Hatch Generation (Optional):** For every selected seed point, compute the dot product between its normal $\vec{N}$ and the light direction $\vec{L}$. If the angle is above a constant threshold, generate an additional stroke along the minimal curvature direction. These cross-hatches are shorter than initial strokes [S2, unofficial].

### Rendering (Two-Pass)

#### Pass 1: Stroke Rendering
Render hatching strokes as line strips starting at the currently selected seed points [S2, unofficial].
1.  Enable depth test. Update depth buffer.
2.  Render line strips from the stroke buffer.
3.  **Illumination:** Illuminate using stored gradients. Apply anisotropic shading model for lines [S2, unofficial].
4.  **Front/Back Facing:** Determine if a line belongs to a front or back-facing part of the surface using surface normal $\vec{N}$ and view vector $\vec{V}$.
    *   Calculate two 2D lookup textures (varying diffuse/specular color).
    *   Map textures to lines.
    *   Interpolate color contributions based on dot product $\vec{N} \cdot \vec{V}$ in the per-fragment shader [S2, unofficial].
    *   Optionally add self-shadowing term using surface normal [S2, unofficial].
    *   *Visual Cue:* Front-facing lines can be colored differently (e.g., blue) and back-facing lines (e.g., yellow) to convey spatial position [S2, unofficial].

#### Pass 2: Volume Rendering
Render the volumetric data set using 3D texture maps [S2, unofficial].
1.  Render transparent volume structures back-to-front without changing OpenGL state.
2.  Use depth-test and alpha-blending to merge transparent/opaque regions with hatches.
3.  **Hatch Visibility for Opaque Surfaces:** To ensure only hatches in front of the iso-surface remain visible for opaque rendering:
    *   Combine alpha-test and depth-test during re-sampling to render only texture samples closest to the viewpoint and above/below a selected threshold [S2, unofficial].
    *   Write white color to the frame buffer after rendering hatches (or use a specific masking technique) so that only hatches drawn in front of the iso-surface remain visible when the volume is rendered [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_gradient(volume_data, pos):
    # Reconstruct partial derivatives using cubic B-spline filtering [S1]
    # Standard central difference approximation for demonstration
    dx = volume_data[pos.x + 1, pos.y, pos.z] - volume_data[pos.x - 1, pos.y, pos.z]
    dy = volume_data[pos.x, pos.y + 1, pos.z] - volume_data[pos.x, pos.y - 1, pos.z]
    dz = volume_data[pos.x, pos.y, pos.z + 1] - volume_data[pos.x, pos.y, pos.z - 1]
    return (dx, dy, dz)

def compute_principal_curvatures(grad):
    # Compute principal curvatures k1, k2 (k1 >= k2) from gradient/Hessian [S1]
    # In practice, this solves the shape operator eigenvalue problem.
    k1, k2 = 1.0, 0.5  # Placeholder for eigenvalue computation
    return k1, k2

def compute_shape_index(k1, k2):
    # Condition: k1 >= k2 and |k1| + |k2| > 0 [S1]
    if abs(k1) + abs(k2) == 0:
        return 0.0
    return (2.0 / math.pi) * math.atan((k1 + k2) / (k2 - k1))

def compute_surface_reliability(k1, k2, s, epsilon):
    # Surface reliability rho [S1]
    if abs(k1) < epsilon and abs(k2) < epsilon:
        return 0.0
    return 1.0 - abs(2.0 * (abs(s) - 0.5))

def smooth_direction(neighbors, s_T, epsilon):
    # Neighbors are particles p_j in the neighborhood [S1]
    sum_rho_k1 = [0.0, 0.0, 0.0]
    sum_rho = 0.0
    count = 0
    
    for neighbor in neighbors:
        grad = compute_gradient(neighbor.volume_data, neighbor.pos)
        k1_j, k2_j = compute_principal_curvatures(grad)
        s_j = compute_shape_index(k1_j, k2_j)
        rho_j = compute_surface_reliability(k1_j, k2_j, s_j, epsilon)
        
        # Get main principal curvature direction k1_j_vec [S1]
        k1_j_vec = neighbor.k1_direction
        
        sum_rho_k1[0] += rho_j * k1_j_vec[0]
        sum_rho_k1[1] += rho_j * k1_j_vec[1]
        sum_rho_k1[2] += rho_j * k1_j_vec[2]
        sum_rho += rho_j
        count += 1
        
    # Trace reliability weight w_T [S1]
    w_T = sum_rho / count if count > 0 else 0.0
    
    if sum_rho > 0:
        avg_k1 = [sum_rho_k1[i] / sum_rho for i in range(3)]
    else:
        avg_k1 = [0.0, 0.0, 0.0]
        
    s_i = [w_T * avg_k1[i] + (1.0 - w_T) * s_T[i] for i in range(3)]
    norm = math.sqrt(sum(x*x for x in s_i))
    return [x/norm for x in s_i] if norm > 0 else [0.0, 0.0, 0.0]

def render_strokes(stroke_buffer, normal_map, view_vector, light_direction):
    # Enable depth test implicitly
    for stroke in stroke_buffer:
        for segment in stroke.segments:
            # Get normal N from normal_map at segment start [S2, unofficial]
            N = normal_map[segment.start_pos]
            
            # Determine front/back facing [S2, unofficial]
            dot_NV = sum(n*v for n, v in zip(N, view_vector))
            is_front = dot_NV > 0
            
            # Anisotropic shading for lines [S2, unofficial]
            # Calculate 2D lookup textures for diffuse/specular
            # Map textures to lines and interpolate color based on dot product N.V
            color = [abs(dot_NV), abs(dot_NV), abs(dot_NV)]
            
            # Draw line segment
            draw_line_segment(segment.start_pos, segment.end_pos, color)

def draw_line_segment(start, end, color):
    # Standard rasterization of a line with given color
    pass

def render_volume(volume_texture, normal_map, stroke_buffer, threshold):
    # Render transparent volume structures back-to-front [S2, unofficial]
    # Enable alpha blending and depth test
    for sample in volume_texture.samples_back_to_front:
        if sample.value > threshold:
            N = normal_map[sample.pos]
            # Apply volume shading
            color = [0.5, 0.5, 0.5]
            # Blend color with sample.alpha
            pass
```

### Parameters and Typical Values
*   **$\epsilon$:** Parameter defining which nearly flat surfaces are considered flat. Value not explicitly given in sources, but used in reliability calculation [S1].
*   **$w_T$:** Trace reliability weight. Determined by averaging surface reliability of neighbors [S1].
*   **$\vec{s}_T$:** User-defined fixed direction.
*   **Cross-hatch angle threshold:** Constant angle between normal and light direction to trigger cross-hatch. Value not given [S2, unofficial].
*   **Viewport Resolution:** 600x600 pixels [S2, unofficial].
*   **Strokes per Example:** 20K to 50K strokes generated [S2, unofficial].
*   **Average Line Segments:** 10 line segments per stroke [S2, unofficial].

### Edge Cases and Pitfalls
*   **Messy Results:** Directly tracing along principal curvature directions yields messy results when the main direction is not robustly defined (e.g., flat areas) [S1]. Smoothing is required.
*   **Stroke Intersections:** Tracing hatches based on smoothed curvature directions and number of segments from the seed point might lead to intersections for long hatch strokes [S1].
*   **Hatch Visibility:** To ensure only hatches in front of the iso-surface remain visible for opaque rendering, write white color into the frame buffer after rendering hatches, then render volume with alpha/depth test [S2, unofficial].
*   **Flat Surfaces:** Surfaces with $|\kappa_1| < \epsilon$ and $|\kappa_2| < \epsilon$ are considered flat and have reliability 0, causing hatches to follow the user-defined fixed direction $\vec{s}_T$ [S1].

## Operating characteristics
*   **Time Complexity:** Preprocessing time depends on curvature estimation and octree construction. For a $256^3$ volume, preprocessing took 1.3 minutes [S2, unofficial]. Seed placement takes 0.5 to 4 seconds [S2, unofficial].
*   **Memory Complexity:** At most one-eighth of the memory consumed by the volume data set is required to store the selected strokes [S2, unofficial].
*   **Performance:** Rendering performance is 4 to 8 fps for various datasets (engine block, hydrogen molecule, iron protein, aneurysm, human foot, human skull) on a 600x600 viewport [S2, unofficial]. Toon shader performance is about 7 fps [S2, unofficial].
*   **Strengths:** Interactive, hardware-assisted, minimizes memory and generation time via data-driven seed point selection, conveys shape and spatial correlations via hatching [S2, unofficial][S1].
*   **Limitations:** May produce intersecting strokes for long hatches; requires robust curvature estimation; performance depends on number of strokes and volume resolution [S1][S2, unofficial].
*   **When to use:** When interactive illustration of volumetric data with free-hand line art style is desired, especially for technical illustrations and sketches [S2, unofficial].
*   **When not to use:** When photorealistic rendering is required, or when high precision in curvature estimation is not feasible on the target hardware.

## Background
The method was presented by R.F.P. van Pelt, A. Vilanova, and H.M.M. van de Wetering in "GPU-based Particle Systems for Illustrative Volume Rendering" (2008) [S1] and "Interactive volume illustration" (2008) [S2, unofficial]. It builds on the concept of using particle systems for illustrative volume rendering [S1]. The shape index $s$ was introduced by Koenderink and Van Doorn [KvD92]. The fast cubic B-spline filtering approach for partial derivatives was presented by Sigg and Hadwiger [SH05]. The anisotropic shading model for lines was proposed in [1] and extended for real-time shading in [35, 40, 14] [S2, unofficial].

## Sources
[S1] GPU-based Particle Systems for Illustrative Volume Rendering, https://doi.org/10.2312/vg/vg-pbg08/089-096, fetched 2026-10-26
[S2] Interactive volume illustration, https://openalex.org/W2147354080, fetched 2026-10-27 (unofficial)
