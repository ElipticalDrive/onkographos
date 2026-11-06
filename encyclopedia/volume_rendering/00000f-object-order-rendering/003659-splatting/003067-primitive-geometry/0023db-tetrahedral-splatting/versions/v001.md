# Tetrahedral Splatting

Also called: TetSplat

## Overview
Tetrahedral Splatting (TetSplat) is a point-based, object-order rendering technique designed for the real-time visualization of large, static tetrahedral meshes [S1, unofficial]. It addresses the performance bottleneck of ray-casting unstructured grids by approximating each tetrahedron with a single spherical or ellipsoidal splat, leveraging hierarchical data structures for efficient culling [S1, unofficial]. Unlike [Voxel Splat Rendering](3730-voxel-splat-rendering.md), which relies on regular grid structures, TetSplat operates directly on unstructured tetrahedral elements, but it is distinct from [Circle Splat Rendering](2981-circle-splat-rendering.md) in its specific handling of tetrahedral topology and preprocessing requirements [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A static tetrahedral mesh $\mathcal{T}$ consisting of vertices $\mathbf{v}_i$ and tetrahedra $T_j$. Each tetrahedron has an associated scalar value (e.g., density or temperature) $\rho_j$.
*   **Output:** A 2D image frame buffer.

**Data Structures:**
*   **Splat Representation:** Each tetrahedron $T_j$ is represented by a single primitive (splat). The splat is defined by its center $\mathbf{c}_j$ (typically the centroid of the tetrahedron) and a shape kernel $h$.
    *   **Spherical Splat:** Defined by center $\mathbf{c}_j$ and radius $r_j$.
    *   **Ellipsoidal Splat:** Defined by center $\mathbf{c}_j$ and a covariance matrix $\Sigma_j$ derived from the tetrahedron's geometry to approximate its extent.
*   **Hierarchical Culling Structure:** A spatial hierarchy (e.g., bounding volume hierarchy or octree) is built over the splats to enable efficient back-face and occlusion culling [S1, unofficial].

**Preprocessing:**
1.  **Degeneracy Removal:** The mesh is processed to eliminate degenerated tetrahedra (needles, spindles, slivers, and caps) using edge collapse [S1, unofficial]. This step is critical because degenerated tetrahedra are difficult to represent accurately with standard splat kernels.
2.  **Partitioning:** The volume is partitioned into a 'Shell' (boundary) and 'Solid' (internal) parts [S1, 7483-tetsplat-pre-processing-mesh-partitioning.md, unofficial].
    *   The Shell is rendered as a standard opaque surface mesh using vertex normals.
    *   The Solid is rendered using normals derived from a CSG (Constructive Solid Geometry) probe, as interior tetrahedra are only visible at the intersection surface [S1, 7483-tetsplat-pre-processing-mesh-partitioning.md, unofficial].
3.  **Splat Generation:** For each remaining tetrahedron $T_j$, compute the centroid $\mathbf{c}_j$ and the kernel parameters (radius or covariance matrix) [S1, unofficial].

**Rendering Steps:**
1.  **Culling:** Use the hierarchical structure to cull splats that are outside the view frustum, back-facing, or occluded by closer geometry [S1, unofficial].
2.  **Transformation:** Transform the splat centers $\mathbf{c}_j$ from world space to screen space using the Model-View-Projection (MVP) matrix [S1, 6264-primitive-geometry.md, unofficial].
3.  **Rasterization:** For each visible splat, compute its 2D footprint on the screen. The contribution of a splat $j$ to a pixel $(x,y)$ is determined by the kernel function $h$:
    $$ h(\mathbf{u}) = \exp\left(-\frac{\mathbf{u}^T \Sigma^{-1} \mathbf{u}}{2}\right) $$
    where $\mathbf{u}$ is the vector from the splat center to the pixel, and $\Sigma$ is the covariance matrix (identity for spherical splats) [S1, 6264-primitive-geometry.md, unofficial].
4.  **Compositing:** Accumulate the contributions of all overlapping splats to the pixel. The final color $C$ and opacity $\alpha$ are computed using standard alpha blending, typically in back-to-front order to handle transparency correctly [S1, 6264-primitive-geometry.md, unofficial].

**Parameters:**
*   **Kernel Shape:** Spherical or Ellipsoidal. Ellipsoidal shapes provide better coverage for elongated tetrahedra but require more complex covariance calculations [S1, unofficial].
*   **Radius/Covariance:** Determined by the geometric extent of the tetrahedron.

**Edge Cases and Pitfalls:**
*   **Degeneracy:** The removal of degenerated tetrahedra is incompatible with smoke volume rendering, where topology must be maintained and many tetrahedra may naturally approach a degenerated state during integration [S1, unofficial].
*   **Gaps:** Representing a tetrahedron with exactly one splat can lead to visible gaps, especially in small meshes, even when using ellipsoidal shapes [S1, unofficial].

## Subtopics

### [TetSplat Pre-processing: Mesh Partitioning](7483-tetsplat-pre-processing-mesh-partitioning.md)
Partitions the mesh into Shell and Solid parts to handle boundary and interior rendering differently. Choose this when accurate boundary representation is required for unstructured tetrahedral meshes.

## Operating characteristics
*   **Time Complexity:** Rendering time is dominated by the rasterization of visible splats. Hierarchical culling reduces the number of splats processed, but the worst-case complexity remains proportional to the number of tetrahedra $N$ if no culling is effective. Preprocessing is $O(N)$ or $O(N \log N)$ depending on the hierarchy construction.
*   **Memory Complexity:** $O(N)$ to store splat parameters and the hierarchical structure.
*   **Performance:** Enables real-time rendering of large tetrahedral meshes on modern GPUs [S1, unofficial]. Performance is highly dependent on the efficiency of the culling structure and the number of degenerated tetrahedra removed.
*   **Strengths:** Real-time capability for unstructured grids; leverages hardware rasterization; efficient culling.
*   **Limitations:** Static mesh requirement (preprocessing assumes static geometry); quality issues due to single-splat-per-tetrahedron approximation (gaps); incompatibility with dynamic topology changes (e.g., smoke integration) due to degeneracy removal [S1, unofficial].
*   **When to Use:** Real-time visualization of large, static unstructured tetrahedral meshes where approximate volume representation is acceptable.
*   **When Not to Use:** Dynamic meshes (e.g., fluid simulation with topology changes), high-accuracy requirements where gaps are unacceptable, or scenes with many degenerated tetrahedra that cannot be safely removed [S1, unofficial].

## Background
Tetrahedral Splatting was introduced as a point-based technique for real-time rendering of large tetrahedral meshes [S1, unofficial]. It builds on the general splatting framework by adapting it to unstructured tetrahedral elements. The method was developed to address the limitations of ray-casting approaches for large datasets, offering a trade-off between performance and accuracy [S1, unofficial]. The approach is particularly relevant in fields requiring visualization of complex unstructured grids, such as computational fluid dynamics or finite element analysis, although its static mesh constraint limits its applicability in dynamic scenarios [S1, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
