# Gouraud Shading

Also called: intensity interpolation shading, smooth shading

## Overview
Gouraud shading is a surface shading technique that computes illumination colors at mesh vertices and linearly interpolates these colors across the resulting primitives to approximate smooth lighting [S1]. It distinguishes itself from sibling methods by interpolating the final color value rather than the geometric normal vector, as done in [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A mesh defined by vertices, vertex normals, and a directional light source [G].
*   **Output:** A rasterized image with per-pixel colors.

**Data Structures:**
*   **Vertex Attributes:** Each vertex stores a position and a normal vector $\mathbf{n}$ [G].
*   **Interpolation:** Linear interpolation of scalar color values across triangle edges and interior [S1].

**Steps:**
1.  **Compute Vertex Normals:** For curved surfaces, calculate the normal for each vertex by averaging the normals of all adjacent faces sharing that vertex [S2, unofficial]. For planar surfaces, the same normal is applied to all defining vertices [S3, unofficial].
2.  **Calculate Vertex Colors:** For each vertex, compute the color using the lighting model [S2, unofficial]. The intensity depends on the angle between the light source direction and the vertex normal vector [S3, unofficial].
3.  **Interpolate Colors:** During rasterization, linearly interpolate the computed vertex colors across the surface of the primitive to determine the color of each pixel [S1].

**Formulas:**
The sources do not specify an exact mathematical lighting equation. The vertex color is determined by the angle between the light source direction and the vertex normal vector [S3, unofficial]. The interpolated color at any point on the primitive is a linear combination of the vertex colors [S1].

**Parameters:**
*   **Normal Vector:** A vertex attribute pointing perpendicular to the surface, used to estimate light reflection [S3, unofficial].
*   **Shading Model Flag:** In OpenGL, `glShadeModel(GL_SMOOTH)` enables this method [S3, unofficial].

**Pitfalls:**
*   **Faceted Appearance:** If normals are not averaged for curved surfaces, the discontinuity in slope between faces results in a faceted or gem-like appearance [S3, unofficial].
*   **Intensity Discontinuities:** Without shading, curved surfaces approximated by polygons exhibit Mach band effect intensity discontinuities along edges [S2, unofficial].

## Operating characteristics
*   **Complexity:** Computation is performed per-vertex, making it faster than per-pixel shading methods like Phong shading [G].
*   **Strengths:** Provides a smoother appearance than flat shading by diminishing intensity discontinuities [S2, unofficial].
*   **Limitations:** Not given in the sources.

## Background
Gouraud shading, also known as intensity interpolation shading, was introduced as an approach to render approximate surfaces with smoother color variations than flat shading [S1]. It is implemented in systems such as Smokeview for visualizing fire dynamics simulation data, where it is referred to as smooth shading [S3, unofficial].

## Sources
[S1] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S2] DTIC ADA259586: Scientific Visualization of Volumetric Radar Cross Section Data, https://archive.org/details/DTIC_ADA259586, fetched 2026-10-26 (unofficial)
[S3] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
