# Tetrahedral Normal Calculation

Also called: Vertex-weighted tetrahedral normal

## Overview
This method computes smooth isosurface normals for tetrahedral meshes by calculating vertex normals weighted by the volume of connected tetrahedra [S1]. It solves the problem of severe visual discontinuities that arise when using the constant cell normal $N_t$ directly, which decreases rendering quality significantly [S1]. It differs from its siblings, such as [Regularised Marching Tetrahedra](1301-regularised-marching-tetrahedra.md) which modifies the topology, by retaining the standard Marching Tetrahedrons topology but enhancing the shading data through pre-computed vertex attributes.

## How it works
**Inputs:**
*   A tetrahedral mesh where each tetrahedron $h_t$ has four vertices.
*   Scalar field values at the vertices (used implicitly to determine the plane orientation, though the normal is constant per cell).
*   The connectivity list of tetrahedra.

**Outputs:**
*   A vertex normal vector $N_v$ for each mesh vertex, stored in the data file.
*   During rendering, a smooth interpolated normal for any intersection point on the isosurface.

**Data Structures:**
*   **Vertex Normal Array:** An array of 3D vectors, one per vertex.
*   **Tetrahedron Data:** For each tetrahedron, the pre-computed plane normal $N_t$ and volume $V(h_t)$.

**Steps:**

1.  **Compute Tetrahedron Plane Normal ($N_t$):**
    For each tetrahedron $h_t$, determine the normal $N_t = (a, b, c)^T$ of the plane describing the iso-surface within that cell. This normal is constant within the cell because the orientation of the plane is independent of the iso-value [S1].
    *Note: The specific geometric calculation of $N_t$ from vertex positions and scalar values is assumed to be handled by the parent Marching Tetrahedrons Algorithm. This method assumes $N_t$ is available.*

2.  **Compute Vertex Normals ($N_v$):**
    For each vertex $v$ in the mesh, calculate its normal $N_v$ by summing the contributions of all tetrahedra $h_t$ connected to $v$. The contribution is weighted by the volume of the tetrahedron $V(h_t)$ [S1].
    
    The formula is:
    $$ N_v = \sum_{t=0}^{n} N_t V(h_t) $$
    where:
    *   $n$ is the number of tetrahedra connected to vertex $v$.
    *   $N_t$ is the plane normal of tetrahedron $h_t$.
    *   $V(h_t)$ is the volume of tetrahedron $h_t$.
    
    After the summation, normalize the resulting vector:
    $$ N_v \leftarrow \frac{N_v}{\|N_v\|} $$
    
    Store $N_v$ for each vertex [S1].

3.  **Interpolate Normal at Intersection Point:**
    During ray traversal, when an intersection point $P$ is found within a tetrahedron $h_t$:
    a. Calculate the 3D barycentric coordinates $(\lambda_0, \lambda_1, \lambda_2, \lambda_3)$ of $P$ with respect to the four vertices of $h_t$. This extends the 2D barycentric coordinate system used for triangles to the 3D tetrahedral case [S1].
    b. Interpolate the pre-computed vertex normals using these barycentric coordinates:
       $$ N_{interp} = \sum_{i=0}^{3} \lambda_i N_{v,i} $$
       where $N_{v,i}$ is the pre-computed normal of the $i$-th vertex of the tetrahedron.
    c. Normalize the interpolated vector:
       $$ N_{final} = \frac{N_{interp}}{\|N_{interp}\|} $$
    d. Use $N_{final}$ for shading calculations [S1].

**Parameters:**
*   None specific to the calculation itself, other than the mesh geometry and scalar field.

**Edge Cases and Pitfalls:**
*   **Discontinuities:** Using the constant cell normal $N_t$ directly results in severe discontinuities between tetrahedra surfaces [S1]. This method mitigates that by smoothing across vertices.
*   **Zero Volume:** If a tetrahedron has zero volume (degenerate), it contributes nothing to the sum. Ensure degenerate tetrahedra are handled or excluded during pre-computation.
*   **Normalization:** Failure to normalize $N_v$ after summation and $N_{interp}$ after interpolation will result in incorrect lighting intensity and direction.

## Subtopics

### [Average Normal Calculation for Cut Triangles](11806-average-normal-calculation-for-cut-triangles.md)
This method computes smooth isosurface normals for newly generated triangles created by cutting tetrahedra during the Marching Tetrahedrons Algorithm. It specifically addresses triangles that lie strictly inside the isosurface (i.e., the parent tetrahedron contains only cutting planes, not the isosurface itself). It differs from gradient-based approximations by averaging the normals of the intersecting cutting planes, providing a geometrically consistent normal for interior cut faces [11806-average-normal-calculation-for-cut-triangles.md].

### Marching Tetrahedra Fragment Shader
In a fragment shader implementation, the interpolated normal vector is output as the color. The step involves setting the fragment color to the normal vector: `FragColor = vec4(oNormal, 1.0);` [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Pre-computation is $O(V \cdot T_{avg})$ where $V$ is the number of vertices and $T_{avg}$ is the average number of tetrahedra per vertex. Rendering interpolation is $O(1)$ per intersection point.
*   **Memory Complexity:** Requires storing one 3D vector per vertex.
*   **Strengths:** Produces smooth shading across the isosurface, significantly improving visual quality over flat shading with cell normals [S1].
*   **Limitations:** Requires pre-computation and storage of vertex normals. Does not address topological artifacts, only shading artifacts.
*   **When to use:** When smooth shading is required for tetrahedral volume rendering and the parent algorithm provides tetrahedron plane normals.

## Background
This method was described in the context of fast ray traversal of tetrahedral and hexahedral meshes for direct volume rendering [S1]. It addresses the visual quality issues inherent in the basic Marching Tetrahedrons algorithm when using flat shading.

## Sources
[S1] Fast Ray Traversal of Tetrahedral and Hexahedral Meshes for Direct Volume Rendering, https://doi.org/10.2312/vissym/eurovis06/235-242, fetched 2026-10-24
[S2] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
