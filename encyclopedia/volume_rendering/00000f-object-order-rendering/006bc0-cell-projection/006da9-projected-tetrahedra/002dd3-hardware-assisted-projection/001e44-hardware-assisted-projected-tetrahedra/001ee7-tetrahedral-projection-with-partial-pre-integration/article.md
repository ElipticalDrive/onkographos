# Tetrahedral Projection with Partial Pre-integration

Also called: PT algorithm, PTINT (Projected Tetrahedra with Partial Pre-Integration), Projected Tetrahedra with Partial Pre-Integration

## Overview
Tetrahedral Projection with Partial Pre-integration (PTINT) is a hybrid volume and isosurface rendering method that projects tetrahedra onto the screen and computes fragment colors using pre-integrated optical depth tables [S1, unofficial]. It solves the performance bottleneck of explicit isosurface extraction by computing isosurfaces per fragment within the rendering pipeline, avoiding the case-handling complexity of marching tetrahedra [S1, unofficial]. Unlike full pre-integration, PTINT uses a transfer-function-independent $\psi$ table, allowing interactive transfer function editing without expensive table recomputation [S2, unofficial]. It builds on [Hardware-Assisted Projected Tetrahedra](1729-hardware-assisted-projected-tetrahedra.md) by leveraging GPU rasterization for cell projection [S2, unofficial].

## How it works
**Inputs:** Regular 3D scalar volume data (stored as hexahedral volunits), viewing vector $\vec{v}$, isovalue (optional), transfer function (TF), Phong illumination coefficients (optional) [S2, unofficial].
**Outputs:** Final color buffer [S1, unofficial].

**Data Structures:**
1.  **Volume Data:** Regular grid of hexahedral volunits, each decomposed into 5 tetrahedra [S2, unofficial].
2.  **Exponential Texture:** 1D texture (256 entries) storing $e^{-u}$ for $u \in [0,1]$ [S2, unofficial].
3.  **$\psi$ Table:** 2D texture storing partial pre-integration integral values, independent of the transfer function [S2, unofficial].
4.  **Transfer Function Texture:** 1D texture mapping scalar values to RGBA colors [S2, unofficial].
5.  **Vertex/Color/Normal Arrays:** Fixed-size arrays passed to `glMultiDrawElements`, containing data for 5 tetrahedra per volunit (4 vertices + 1 thick vertex per tetrahedron) [S2, unofficial].
6.  **Index Arrays:** 5 arrays of indices defining the triangle fan order for the 5 tetrahedra, created once for all volunits [S2, unofficial].

**Algorithm Steps:**

1.  **Sorting:** Determine traversal order based on viewing vector $\vec{v} = \{v_x, v_y, v_z\}$ [S2, unofficial].
    *   If $v_x > 0$, traverse x-indices ascending; else descending [S2, unofficial].
    *   If $v_y > 0$, traverse y-indices ascending; else descending [S2, unofficial].
    *   If $v_z > 0$, traverse z-indices ascending; else descending [S2, unofficial].
    *   The relative order of axes traversal does not matter for regular data [S2, unofficial].
    *   Only 8 traversal permutations are required to cover all viewing angles [S2, unofficial].

2.  **Data Preparation (CPU):**
    *   For each volunit, compute vertex positions and colors [S2, unofficial].
    *   Compute normal vectors (valid for all volunits, computed once) [S2, unofficial].
    *   Upload vertex, color, and normal arrays to GPU memory [S2, unofficial].
    *   If isosurfaces/lighting are disabled, skip gradient computation and normal interpolation [S1, unofficial].

3.  **Rendering (GPU Vertex Shader):**
    *   Read basis hexahedron information from global uniform variables [S2, unofficial].
    *   Displace vertices correctly using the basis information [S2, unofficial].
    *   Pass vertex attributes (position, color, normal) to the fragment shader [S2, unofficial].

4.  **Rasterization:**
    *   Render each volunit as 5 triangle fans (one per tetrahedron) [S2, unofficial].
    *   Interpolate color attributes across the triangles [S2, unofficial].

5.  **Fragment Color Computation (Fragment Shader):**
    *   **Isosurface Rendering (Optional):**
        *   Check if isovalue is within the scalar range of the current fragment [S1, unofficial].
        *   Retrieve front gradient $\vec{g}_f$ and back gradient $\vec{g}_b$ [S1, unofficial].
        *   Compute normal vector $\vec{n}$ by combining $\vec{g}_f$ and $\vec{g}_b$ with weights relative to the distance between the isosurface and the front/back intersection points [S1, unofficial].
        *   Compute Phong illumination (diffuse, specular, ambient) using $\vec{n}$ [S1, unofficial].
    *   **Volume Rendering:**
        *   Retrieve front color $C_f$ and back color $C_b$ from the transfer function texture using scalar values at the front and back intersections [S2, unofficial].
        *   Retrieve thickness $l$ (optical depth or geometric thickness) [S2, unofficial].
        *   Compute indices for the $\psi$ table using $C_f$, $C_b$, and $l$ [S2, unofficial].
        *   Lookup value from the $\psi$ table [S2, unofficial].
        *   Compute final fragment color using the $\psi$ table value [S2, unofficial].
        *   *Alternative (Simpler/Faster):* Compute chromaticity as average of $C_f$ and $C_b$ [S2, unofficial]. Compute opacity $\alpha = 1 - e^{-\tau \cdot l}$, where $\tau$ is mean alpha [S2, unofficial]. Use exponential texture for $e^{-u}$ lookup [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_fragment_color(front_scalar, back_scalar, thickness_l, isosurface_enabled, isovalue, view_vector):
    # 1. Transfer function lookup
    C_f = tf_texture[front_scalar]
    C_b = tf_texture[back_scalar]
    
    # 2. Exponential lookup for opacity
    u = thickness_l
    exp_val = exp_texture[u]
    alpha = 1.0 - exp_val
    
    # 3. Partial pre-integration lookup
    # Compute indices for psi table using C_f, C_b, and l
    # (Exact indexing formula not given in sources [G])
    psi_idx = compute_psi_index(C_f, C_b, thickness_l)
    psi_value = psi_table[psi_idx]
    
    # 4. Final color composition
    # (Exact composition formula not given in sources [G])
    final_color = psi_value
    
    # 5. Isosurface contribution (optional)
    if isosurface_enabled:
        if min(front_scalar, back_scalar) <= isovalue <= max(front_scalar, back_scalar):
            # Retrieve gradients
            g_f = gradient_array[front_scalar]
            g_b = gradient_array[back_scalar]
            # Compute distances to intersection points
            d_f = distance_to_front_intersection(front_scalar)
            d_b = distance_to_back_intersection(back_scalar)
            d_iso = distance_to_isosurface_intersection(isovalue)
            # Weights relative to distance between isosurface and intersection points
            w_f = abs(d_iso - d_f)
            w_b = abs(d_iso - d_b)
            # Combine gradients for normal
            n = normalize(w_f * g_f + w_b * g_b)
            # Phong illumination
            iso_color = phong_lighting(n, view_vector)
            final_color = iso_color + final_color
    return final_color

def normalize(v):
    mag = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return (v[0]/mag, v[1]/mag, v[2]/mag)

def phong_lighting(normal, view_dir):
    # Standard Phong model using pre-loaded coefficients [S1, unofficial]
    diffuse = max(0.0, sum(n * l for n, l in zip(normal, light_dir)))
    specular = max(0.0, sum(n * r for n, r in zip(normal, reflect_dir))) ** 4.0
    return (diffuse * ambient + specular * specular_coeff, 0.0, 0.0, 1.0)

def compute_psi_index(cf, cb, l):
    # Placeholder for exact indexing logic [G]
    return (int(cf * 255), int(cb * 255))

def distance_to_front_intersection(scalar):
    # Placeholder for ray-tetrahedron intersection math [G]
    return 0.0

def distance_to_back_intersection(scalar):
    return 0.0

def distance_to_isosurface_intersection(iso):
    return 0.0
```

**Parameters:**
*   **Exponential Texture Size:** 256 entries [S2, unofficial].
*   **Storage per Tetrahedron:** 24 bytes (assuming 4 tets per vertex, 16 bytes per texture element) [S1, unofficial].
*   **Traversal Orders:** 8 possible permutations [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Clipping:** Clipping must be parallel to the volume's bounding box sides to maintain regular properties required for implicit sorting [S2, unofficial].
*   **Memory:** The entire dataset must fit in main memory to avoid swapping, as no extra data structures are created [S2, unofficial].
*   **Performance:** Partial pre-integration is slower than simple scalar averaging but necessary for interactive TF editing [S2, unofficial]. Full pre-integration requires expensive table recomputation on TF change [S2, unofficial].

## Operating characteristics
**Time Complexity:** Sorting is $O(1)$ due to regular data spacing and implicit 3D array structure [S2, unofficial]. Rendering is proportional to the number of tetrahedra [S2, unofficial].
**Memory Complexity:** 24 bytes per tetrahedron on GPU memory [S1, unofficial]. No extra data structures beyond volume data [S2, unofficial].
**Performance:**
*   On Intel Pentium IV 3.6 GHz, GeForce 6800, 512x512 viewport:
    *   Blunt Fin: 11.30 fps (PTINT) vs 4.07 fps (GATOR), 5.20 fps (VICP), 4.47 fps (HARC), 6.09 fps (HAVS) [S1, unofficial].
    *   Oxygen Post: 4.49 fps (PTINT) vs 8.63 fps (HARC) [S1, unofficial].
    *   Fuel (1.2M tet): 70.78 fps, 5.99 M tet/s [S2, unofficial].
    *   Tooth (5M tet): 1.22 fps, 6.30 M tet/s [S2, unofficial].
    *   Aneurism (83M tet): 2.42 fps, 5.35 M tet/s [S2, unofficial].
*   Rendering time accounts for >60% of total time for most datasets [S2, unofficial].
**Strengths:** High frame rates, interactive transfer function editing, no explicit isosurface case handling, low memory footprint compared to ray casting [S1, S2, unofficial].
**Limitations:** Limited to regular data, clipping constraints, slower than simple averaging, requires data to fit in memory [S2, unofficial].

## Background
PTINT is based on the Projected Tetrahedra (PT) method by Roettger et al. (2000) and partial pre-integration by Moreland and Angel (2004) [S2, unofficial]. It was developed to leverage GPU acceleration for cell projection while maintaining interactive capabilities for volume visualization [S1, unofficial].

## Sources
[S1] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
[S2] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2007/20825/, fetched 2026-10-29 (unofficial)
