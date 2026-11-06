# Sorted Rasterization of Projected Tetrahedra

Also called: GPU DVR for unstructured data via sorted PT, Radiance Mesh Rendering via Hardware Rasterization, Tet-by-Tet Ray Casting

## Overview
Sorted Rasterization of Projected Tetrahedra is a GPU-based direct volume rendering (DVR) method for unstructured tetrahedral data that achieves interactive rates by treating each tetrahedron as a distinct primitive and rasterizing them using hardware operations [S1, unofficial]. It differs from traditional ray casting by leveraging hardware rasterization and blending to compute pixel color, avoiding explicit per-pixel ray-tetrahedron intersection tests [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh where each tetrahedron $k$ has constant density $\sigma_k$ and color fields $\mathbf{c}_k(\mathbf{r})$ defined at any point $\mathbf{r}$ within it [S1, unofficial]. The camera position (origin) is required for sorting [S1, unofficial].
*   **Output:** A 2D image where each pixel contains the accumulated color $C$ and opacity $\alpha$ based on front-to-back blending [S1, unofficial].

**Data Structures:**
*   **Tetrahedron Attributes:** For each tetrahedron $k$, store:
    *   Vertices: 4 vertices defining the geometry.
    *   Density: Scalar $\sigma_k$.
    *   Color Fields: Functions or values defining $\mathbf{c}_k^{\text{in}}$ and $\mathbf{c}_k^{\text{out}}$ at entry and exit points.
    *   Circumsphere Power: A scalar value used for sorting relative to the camera origin.
*   **Sorted List:** Tetrahedra sorted by their circumsphere power relative to the camera, loaded from back to front [S1, unofficial].

**Numbered Steps:**

1.  **Sorting:** Sort all tetrahedra based on the power of their circumspheres relative to the camera origin to approximate a back-to-front order [S1, unofficial].
2.  **Rendering Setup:**
    *   **Without Mesh Shaders:** Use instanced shaders. Draw an instance of a triangle strip for each tetrahedron. The vertex shader uses the instance index to load tetrahedron attributes and plane equations. Each tetrahedron is loaded 4 times, once per vertex [S1, unofficial].
    *   **With Mesh Shaders:** Replace vertex shaders with mesh shaders. An entire thread block collaboratively generates mesh data passed directly to the hardware rasterizer. Use warp shuffle operations to share data loaded within each tetrahedron, loading each tetrahedron only once [S1, unofficial].
3.  **Vertex Processing:** For each tetrahedron, emit 4 vertices with tetrahedron data attached to each to the interpolation registers [S1, unofficial].
4.  **Fragment Shader Execution:**
    *   For each pixel hit by a tetrahedron, read back the whole tetrahedron data attached to the triangle hit to reconstruct the back intersection [S1, unofficial].
    *   Compute the entry point $t_k^{\text{in}}$ and exit point $t_k^{\text{out}}$ along the ray $\mathbf{r}$ for the tetrahedron.
    *   Calculate the optical depth $d_k$ and opacity $\alpha_k$:
        $$d_k = (t_k^{\text{out}} - t_k^{\text{in}}) \sigma_k$$
        $$\alpha_k = 1 - e^{-d_k}$$
    *   Evaluate the color at the entry and exit points:
        $$\mathbf{c}_k^{\text{in}} = \mathbf{c}_k(\mathbf{r}(t_k^{\text{in}}))$$
        $$\mathbf{c}_k^{\text{out}} = \mathbf{c}_k(\mathbf{r}(t_k^{\text{out}}))$$
    *   Compute the color change $\Delta \mathbf{C}_k$ using the closed-form solution:
        $$\Delta \mathbf{C}_k = \left(1 - \frac{\alpha_k}{d_k}\right) \mathbf{c}_k^{\text{in}} + \left(\frac{\alpha_k}{d_k} - e^{-d_k}\right) \mathbf{c}_k^{\text{out}}$$
5.  **Compositing:** Blend the pre-sorted tetrahedra along the ray. The final pixel color is:
    $$C = \sum_k w_k \Delta C_k$$
    where the weight $w_k = \prod_{l=1}^{k-1} (1 - \alpha_\ell)$ is efficiently computed using hardware blending operations [S1, unofficial]. The compositor composites the fragment according to the load order [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def intersect_ray_tet(ray_origin, ray_dir, tet_vertices):
    """Standard ray-tetrahedron intersection [G]. Returns (t_in, t_out) or (None, None)."""
    t_min, t_max = float('inf'), float('-inf')
    for i in range(4):
        v0, v1, v2 = tet_vertices[i], tet_vertices[(i+1)%4], tet_vertices[(i+2)%4]
        edge1 = v1 - v0
        edge2 = v2 - v0
        normal = np.cross(edge1, edge2)
        denom = np.dot(ray_dir, normal)
        if abs(denom) > 1e-8:
            t = np.dot(v0 - ray_origin, normal) / denom
            if t > 1e-6:
                p = ray_origin + t * ray_dir
                v3 = tet_vertices[(i+3)%4]
                edge3 = v3 - v0
                if np.dot(np.cross(edge1, p - v0), normal) >= 0 and \
                   np.dot(np.cross(edge2, p - v1), normal) >= 0 and \
                   np.dot(np.cross(edge3, p - v2), normal) >= 0:
                    t_min = min(t_min, t)
                    t_max = max(t_max, t)
    if t_min < t_max:
        return t_min, t_max
    return None, None

def render_pixel(ray_origin, ray_dir, sorted_tets):
    accumulated_color = np.array([0.0, 0.0, 0.0])
    accumulated_opacity = 0.0
    
    for tet in sorted_tets:
        t_in, t_out = intersect_ray_tet(ray_origin, ray_dir, tet.vertices)
        
        if t_in is None:
            continue
            
        sigma = tet.density
        d_k = (t_out - t_in) * sigma
        alpha_k = 1.0 - math.exp(-d_k)
        
        if alpha_k < 1e-6:
            continue
            
        c_in = tet.color_field(ray_origin + ray_dir * t_in)
        c_out = tet.color_field(ray_origin + ray_dir * t_out)
        
        # Limit for d_k -> 0 using L'Hôpital's rule [G]
        if d_k < 1e-6:
            delta_C = np.array([0.0, 0.0, 0.0])
        else:
            alpha_over_d = alpha_k / d_k
            delta_C = (1.0 - alpha_over_d) * c_in + (alpha_over_d - math.exp(-d_k)) * c_out
            
        weight = 1.0 - accumulated_opacity
        accumulated_color += weight * delta_C
        accumulated_opacity += weight * alpha_k
        
        if accumulated_opacity >= 1.0:
            break
            
    return accumulated_color, accumulated_opacity
```

**Parameters:**
*   **Density ($\sigma_k$):** Constant for each tetrahedron [S1, unofficial].
*   **Color Fields ($\mathbf{c}_k$):** Defined by the dataset [S1, unofficial].
*   **Sorting Criterion:** Power of circumsphere relative to camera origin [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** When $d_k$ approaches zero, the ratio $\frac{\alpha_k}{d_k}$ approaches 1. The limit yields $\Delta \mathbf{C}_k \to \mathbf{0}$, which is handled by a threshold check [G].
*   **Precision:** Not given in the sources.
*   **Degenerate Tetrahedra:** Not given in the sources.

## Subtopics

### Radiance Meshes
A specific implementation of sorted rasterization of projected tetrahedra that treats each tetrahedron as a distinct primitive and outputs individually wound tetrahedra [S1, unofficial].

### Instanced Rendering
A rendering technique used when mesh shaders are not available, drawing an instance of a triangle strip for each tetrahedron [S1, unofficial].

### Mesh Shaders
A modern GPU feature that allows collaborative generation of mesh data, reducing redundant data loads [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Achieves interactive rates on commodity GPUs [S1, unofficial]. Mesh shaders provide the fastest rendering approach by reducing redundant data loads [S1, unofficial].
*   **Strengths:** Leverages hardware rasterization and blending, avoiding explicit ray casting per pixel. Efficient for unstructured data.
*   **Limitations:** Requires sorting of tetrahedra.
*   **When to Use:** Interactive volume rendering of unstructured tetrahedral data on modern GPUs.
*   **When Not to Use:** Not given in the sources.

## Background
The method is part of the broader field of GPU-based direct volume rendering for unstructured data [S2, unofficial]. It builds on the concept of projected tetrahedra and optimizes it for hardware rasterization [S1, unofficial]. The specific implementation using mesh shaders and warp shuffle operations is described in [S1, unofficial].

## Sources
[S1] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
