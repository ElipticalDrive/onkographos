# Reflected Light Component Lookup

Also called: Gradient-based shading lookup, Lookup Table Shading, Lookup Table for Shading, Material-Gradient Lookup Table, Shading via lookup tables

## Overview
Reflected Light Component Lookup is a shading method that pre-calculates surface responses to directional lighting for combinations of material types and quantized gradient directions, storing the results in a lookup table [S1, S2]. It solves the problem of efficiently rendering multiple-material volumes with specular highlights and directional lighting by decoupling expensive lighting calculations from the per-voxel rendering loop [S2]. Unlike sibling methods that rely solely on ambient lighting or simple gradient magnitude, this approach encodes the full reflected light component (diffuse and specular) based on the voxel's orientation relative to light sources, allowing for realistic surface appearance in volume rendering [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $m$: Number of material types specified by the client [S1, S2].
    *   $k$: Number of refinement levels for gradient quantization [S1, S2].
    *   Lighting parameters: Light colors, directions, and enabled states [S3].
    *   Material parameters: Shading properties (e.g., shininess) for each material [S3].
    *   Voxel data: Scalar field values and gradients [S1, S2].
*   **Outputs:**
    *   A pre-computed color lookup table (LUT) mapping indices to RGB(A) intensities [S1, S2].
    *   Two auxiliary fields stored per voxel: an encoded surface normal index and a weight index [S3].
    *   Rendered image with directional lighting effects [S2].

**Data Structures:**
1.  **Color Table:** A 3D array `float color_table[n][num_materials][color_channels]` [S3].
    *   $n$: Number of possible values for the color field (lookup index size) [S3].
    *   `num_materials`: $m$ [S3].
    *   `color_channels`: 1 for grayscale, 3 for RGB [S3].
    *   Values are floating-point intensities in $[0.0, 1.0]$ [S1, S3].
2.  **Weight Table:** A 2D array `float weight_table[m_w][num_materials]` [S3].
    *   $m_w$: Number of possible values for the weight field [S3].
    *   Values are fractional occupancies in $[0.0, 1.0]$ [S3].
    *   If `num_materials` is 1, this table is unused [S3].
3.  **Voxel Fields:**
    *   **Color Field:** Stores an index into the Color Table. This index encodes both the material type and the quantized gradient direction [S1, S2, S3].
    *   **Weight Field:** Stores an index into the Weight Table, representing the mixture of materials in the voxel [S3].

**Gradient Quantization:**
The gradient direction space is quantized into a finite set of unit vectors distributed evenly on a unit sphere [S1, S2]. The number of quantized gradients $g$ is determined by the refinement level $k$ [S1, S2]:
$$ g = 30 \cdot 4^k + 2 $$
[S1, S2]

**Lookup Index Calculation:**
For each voxel, a final lookup index $I$ is computed for the Color Table [S1, S2]:
*   If the voxel has reflection properties (classified as a surface):
    $$ I = M \cdot g + G $$
    Where:
    *   $M$: The material index of the voxel [S1, S2].
    *   $g$: The total number of quantized gradients [S1, S2].
    *   $G$: The quantized gradient index, found by mapping the voxel's actual gradient direction to the closest pre-computed quantized gradient vector [S1, S2].
*   If the voxel has no reflection properties (ambient/volume):
    $$ I = M $$
    [S1, S2]
    *   Note: Material 0 is the "null" material. Its entries in the lookup table store colors for ambient materials, which do not require a gradient index [S1, S2].

**Shading Model:**
The color values in the lookup table are calculated using a "half-way vector" specularity model [TM67, CT82] combined with Lambertian diffuse shading [S1, S2]. The surface response to directional lights is computed for each combination of material and quantized gradient, added to the ambient component, and stored [S1, S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants matching evidence specifications
LIGHT_COLOR = 0
LIGHT_DIRECTION = 1
VP_LIGHT0 = 0
VP_LIGHT_BOTH_SIDES = 1

def setup_lighting(context, light_num, color_rgb, direction_xyz):
    # Direction vector is transformed by the current modeling matrix [S3]
    context['lights'][light_num]['color'] = color_rgb
    context['lights'][light_num]['direction'] = apply_modeling_matrix(direction_xyz)
    context['lights'][light_num]['enabled'] = True
    context['lights'][light_num]['two_sided'] = False

def compute_shading_table(context, color_table, num_materials, color_channels, quantized_gradients):
    # Initialize table with ambient component [S1, S2]
    for i in range(len(quantized_gradients)):
        for m in range(num_materials):
            for c in range(color_channels):
                color_table[i][m][c] = ambient_intensity[m][c] # [G] standard initialization
    # Calculate surface response to directional lights for each gradient/material combo [S1, S2]
    for i, grad in enumerate(quantized_gradients):
        for m in range(num_materials):
            for light in context['lights']:
                if light['enabled']:
                    # Half-way vector specularity and Lambertian diffuse [S1, S2]
                    half_way = normalize(grad + light['direction'])
                    diffuse = max(0.0, dot(grad, light['direction']))
                    specular = pow(max(0.0, dot(half_way, view_direction)), shininess[m])
                    color_table[i][m] += light['color'] * (diffuse + specular)
    # Clamp and convert to integers for texture map storage [S1, S2]
    for i in range(len(quantized_gradients)):
        for m in range(num_materials):
            for c in range(color_channels):
                color_table[i][m][c] = clamp(int(color_table[i][m][c] * 255), 0, 255)

def process_voxel(voxel_gradient, voxel_material_index, has_reflection, quantized_gradients):
    # Find closest quantized gradient index G [S1, S2]
    best_idx = 0
    max_dot = -1.0
    for idx, q_grad in enumerate(quantized_gradients):
        d = dot(normalize(voxel_gradient), q_grad)
        if d > max_dot:
            max_dot = d
            best_idx = idx
    G = best_idx
    M = voxel_material_index
    g = len(quantized_gradients)

    if has_reflection:
        lookup_index = M * g + G
    else:
        lookup_index = M
    return lookup_index

def render_voxel(voxel_color_index, voxel_weight_index, color_table, weight_table, num_materials, color_channels):
    base_colors = color_table[voxel_color_index]
    weights = weight_table[voxel_weight_index]
    final_color = [0.0] * color_channels
    for i in range(num_materials):
        for c in range(color_channels):
            final_color[c] += base_colors[i][c] * weights[i]
    return final_color
```

**Edge Cases and Pitfalls:**
*   **Recalculation Requirement:** The lookup table must be recalculated whenever any lighting factor changes, including volume rotation or light source movement [S1, S2]. This is a significant computational overhead compared to ambient-only methods [S2].
*   **Gradient Quantization Error:** The accuracy of the shading depends on the density of the quantized gradient set. Low $k$ values result in visible quantization artifacts in specular highlights [S1, S2].
*   **Material 0 Handling:** Material 0 (null material) uses a simplified index $M$ without gradient combination, as it represents ambient materials that do not reflect light directionally [S1, S2].

## Operating characteristics
*   **Time Complexity:** Setup cost is proportional to the number of table entries ($n \times m \times g$) and lighting parameters [S1, S2]. Rendering is $O(1)$ per voxel for lookup and weighted combination [S3].
*   **Memory Complexity:** Color table requires $O(n \cdot m \cdot c)$ space, weight table requires $O(m_w \cdot m)$ space [S3]. Two additional integer fields per voxel are stored [S3].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance degrades linearly with the number of materials and refinement levels $k$. The need to recalculate the table on rotation prevents efficient caching of the LUT during interactive rotation [S2].
*   **Strengths:** Supports multiple materials with distinct shading properties; enables realistic specular highlights and directional lighting in volume rendering; efficient per-voxel rendering cost [S1, S2, S3].
*   **Limitations:** Requires significant memory for the lookup tables; computationally expensive setup phase; requires table recalculation for any change in lighting or view orientation, unlike ambient-only methods [S1, S2].
*   **When to use:** When realistic surface shading with specular highlights is required for multi-material volumes and interactive rotation is not the primary concern, or when pre-computed frames are acceptable [S2, S3].
*   **When not to use:** When real-time interactive rotation with dynamic lighting is critical and the cost of table recalculation is prohibitive [S2].

## Background
*   **Origin:** The method is described in "Direct Volume Rendering with Shading via Three-Dimensional Textures" [S1, S2] and implemented in the VolPack library [S3].
*   **History:** It departs from earlier work that used gradient magnitude in isolation for classification [S1, S2]. It builds on the multiple-material voxel model introduced by Drebin, Carpenter, and Hanrahan (SIGGRAPH 88) [S3].
*   **Usage:** Implemented in the VolPack software library for volume rendering [S3].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19970021689: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970021689, fetched 2026-10-26
[S2] NASA Technical Reports Server (NTRS) 19970022121: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970022121, fetched 2026-10-26
[S3] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
