# Tetrahedral Normal Calculation

Also called: Vertex-weighted tetrahedral normal

## Overview
This method computes smooth isosurface normals for tetrahedral meshes by calculating vertex normals weighted by the volume of connected tetrahedra [S1]. It solves the problem of severe visual discontinuities that arise when using the constant cell normal $N_t$ directly, which decreases rendering quality significantly [S1].

## How it works
**Inputs:**
*   A tetrahedral mesh where each tetrahedron $h_t$ has four vertices [G].
*   The connectivity list of tetrahedra [G].
*   Pre-computed plane normals $N_t$ and volumes $V(h_t)$ for each tetrahedron [S1].

**Outputs:**
*   A vertex normal vector $N_v$ for each mesh vertex, stored in the data file [S1].
*   During rendering, a smooth interpolated normal for any intersection point on the isosurface [S1].

**Data Structures:**
*   **Vertex Normal Array:** An array of 3D vectors, one per vertex [G].
*   **Tetrahedron Data:** For each tetrahedron, the pre-computed plane normal $N_t$ and volume $V(h_t)$ [S1].

**Steps:**

1.  **Compute Tetrahedron Plane Normal ($N_t$):**
    For each tetrahedron $h_t$, determine the normal $N_t = (a, b, c)^T$ of the plane describing the iso-surface within that cell [S1]. This normal is constant within the cell because the orientation of the plane is independent of the iso-value [S1].

2.  **Compute Vertex Normals ($N_v$):**
    For each vertex $v$ in the mesh, calculate its normal $N_v$ by summing the contributions of all tetrahedra $h_t$ connected to $v$, weighted by their volume $V(h_t)$ [S1]. The formula is:
    $$ N_v = \sum_{t \in \text{connected}(v)} N_t V(h_t) $$
    where $N_t$ is the plane normal of tetrahedron $h_t$ and $V(h_t)$ is its volume [S1]. After the summation, normalize the resulting vector:
    $$ N_v \leftarrow \frac{N_v}{\|N_v\|} $$
    Store $N_v$ for each vertex [S1].

3.  **Interpolate Normal at Intersection Point:**
    During ray traversal, when an intersection point $P$ is found within a tetrahedron $h_t$:
    a. Calculate the 3D barycentric coordinates $(\lambda_0, \lambda_1, \lambda_2, \lambda_3)$ of $P$ with respect to the four vertices of $h_t$, extending the 2D barycentric coordinate system used for triangles to the tetrahedral case [S1].
    b. Interpolate the pre-computed vertex normals using these coordinates:
       $$ N_{interp} = \sum_{i=0}^{3} \lambda_i N_{v,i} $$
       where $N_{v,i}$ is the pre-computed normal of the $i$-th vertex of the tetrahedron [S1].
    c. Normalize the interpolated vector:
       $$ N_{final} = \frac{N_{interp}}{\|N_{interp}\|} $$
    d. Use $N_{final}$ for shading calculations [S1].

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Discontinuities:** Using the constant cell normal $N_t$ directly results in severe discontinuities between tetrahedra surfaces [S1]. This method mitigates that by smoothing across vertices [S1].
*   **Zero Volume:** If a tetrahedron has zero volume (degenerate), it contributes nothing to the sum. Ensure degenerate tetrahedra are handled or excluded during pre-computation [G].
*   **Normalization:** Failure to normalize $N_v$ after summation and $N_{interp}$ after interpolation will result in incorrect lighting intensity and direction [G].

## Subtopics

### [Average Normal Calculation for Cut Triangles](11806-average-normal-calculation-for-cut-triangles.md)
This method computes surface normals for newly generated triangles created when cutting tetrahedra that contain only cutting planes (interior to the isosurface). It differs from gradient-based approximations by averaging the normals of the intersecting cutting planes, providing a geometrically consistent normal for interior cut faces. It builds on [Tetrahedral Normal Calculation](1941-tetrahedral-normal-calculation.md).

### Marching Tetrahedra Fragment Shader
In a fragment shader implementation, the interpolated normal vector is output as the color. The step involves setting the fragment color to the normal vector: `FragColor = vec4(oNormal, 1.0);` [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Described in a paper on fast ray traversal of tetrahedral and hexahedral meshes for direct volume rendering [S1].

## Sources
[S1] Fast Ray Traversal of Tetrahedral and Hexahedral Meshes for Direct Volume Rendering, https://doi.org/10.2312/vissym/eurovis06/235-242, fetched 2026-10-24
[S2] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
