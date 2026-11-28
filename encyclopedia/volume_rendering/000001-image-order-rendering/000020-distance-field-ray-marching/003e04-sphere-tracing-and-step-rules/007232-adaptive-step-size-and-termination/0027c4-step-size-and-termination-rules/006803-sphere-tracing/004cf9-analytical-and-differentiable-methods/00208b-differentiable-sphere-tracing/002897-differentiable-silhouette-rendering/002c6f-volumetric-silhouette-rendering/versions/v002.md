# Volumetric Silhouette Rendering

Also called: Silhouette tracing

## Overview
Volumetric Silhouette Rendering is a technique to render silhouettes of volumetric data by detecting edges where the view vector is perpendicular to the surface normal, using a 3D normal map and a pre-computed cube map for view vector normalization [S1, unofficial]. It sets itself apart from geometry-based silhouette extraction by operating directly on volumetric representations via rasterization of normal fields, allowing interactive rendering of complex volumetric datasets [S1, unofficial].

## How it works
**Inputs:**
- A 3D volumetric dataset [S1, unofficial].
- A pre-computed 3D normal map texture derived from the volume [S1, unofficial].
- A pre-computed cube map storing normalized view vectors for all possible interpolated coordinates [S1, unofficial].
- Camera position (eye point $C$) [S1, unofficial].

**Outputs:**
- A binary silhouette mask (black or white) merged with the base color (e.g., from a toon-shader) [S1, unofficial].

**Data Structures:**
1. **3D Normal Map:** A 3D texture where each texel stores the surface normal vector $(N_x, N_y, N_z)$ at that volumetric position [S1, unofficial].
2. **Cube Map:** A cube map texture where each face stores pre-normalized view vectors [S1, unofficial]. The coordinates used to fetch from this map are the interpolated, un-normalized view vector directions [S1, unofficial].

**Steps:**
1. **Render Normal Map:** Render the 3D normal map. At each vertex of the rendered polygons, issue the normalized view vector as an additional texture coordinate [S1, unofficial].
2. **Interpolate:** During rasterization, interpolate the texture coordinates (including the view vector components) on a per-fragment basis [S1, unofficial].
3. **Re-normalize View Vector:** Because normalization is not preserved by linear interpolation, the interpolated view vector components are used as coordinates to fetch texture values from the pre-computed cube map, returning the correctly normalized view vector $V_{norm}$ for the fragment [S1, unofficial].
4. **Compute Dot Product:** Use register combiners (or equivalent hardware shader logic) to compute the dot product between the normalized view vector $V_{norm}$ and the normal vector $N$ fetched from the 3D normal map: $D = N \cdot V_{norm}$ [S1, unofficial].
5. **Square Result:** Square the dot product result to obtain a positive value: $S = D^2$ [S1, unofficial].
6. **Trigger Multiplexer:** Use the squared value $S$ to trigger a multiplexer. Depending on the dot product between view vector and normal, the multiplexer outputs black or white [S1, unofficial].
7. **Merge:** Merge the silhouette result (black/white) with the colors produced by the toon-shader [S1, unofficial].

**Formula:**
The geometric definition of a silhouette edge for a mesh is given by:
$$ ((N_1 \cdot (C - V_1))(N_2 \cdot (C - V_2))) \leq 0 $$
where $N_1, N_2$ are face normals, $V_1, V_2$ are points on the faces, and $C$ is the eye point [S1, unofficial].

For the volumetric implementation, the condition is approximated by the dot product between the surface normal $N$ and the view vector $V$:
$$ D = N \cdot V_{norm} $$
The silhouette is highlighted when $D \approx 0$. The squared value $D^2$ is used to trigger the output [S1, unofficial].

**Parameters:**
- Threshold for silhouette detection: Not given in the sources.
- Cube map resolution: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Interpolation Error:** Normalization is not preserved by interpolation, necessitating the cube map lookup [S1, unofficial].
- **Hardware Dependency:** The method relies on "register combiners" and specific hardware features like 3D texture support and cube maps, which may not be available on modern hardware without shader equivalents [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Requires storage for a 3D normal map and a cube map [S1, unofficial].
- **Measured Performance:** Experiments were run on a Pentium4 processor at 1.5 GHz and a GeForce4 graphics unit with 128 MB local memory and 3D texture support [S1, unofficial].
- **Scaling:** Not given in the sources.
- **Strengths:** Allows interactive rendering of silhouettes for volumetric data sets [S1, unofficial].
- **Limitations:** Hardware-dependent (register combiners, 3D textures); relies on pre-computed cube maps which may consume significant memory [S1, unofficial].

## Background
The method was proposed in the context of interactive volume illustration, aiming to provide silhouette cues for volumetric data similar to those available in geometry-based rendering [S1, unofficial]. It builds on the geometric definition of silhouette edges for polygonal meshes [S1, unofficial].

## Sources
[S1] Interactive volume illustration, https://openalex.org/W2147354080, fetched 2026-10-27 (unofficial)
