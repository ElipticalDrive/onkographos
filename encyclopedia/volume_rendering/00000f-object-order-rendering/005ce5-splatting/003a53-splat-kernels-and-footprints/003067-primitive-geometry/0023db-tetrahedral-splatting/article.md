# Tetrahedral Splatting

Also called: TetSplat

## Overview
Tetrahedral Splatting (TetSplat) is a point-based technique for the real-time rendering of large, static tetrahedral meshes by approximating each tetrahedron with a spherical or ellipsoid splat [S1, unofficial]. It replaces complex volumetric primitives with screen-space footprint functions and uses hierarchical data structures for efficient back-face and occlusion culling [S1, unofficial]. Unlike [Voxel Splat Rendering](3730-voxel-splat-rendering.md), which relies on regular grid discretization, or [Triangle Mesh Splatting](8109-triangle-mesh-splatting.md), which projects surface boundaries, TetSplat operates directly on volumetric tetrahedral elements [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A static, unstructured tetrahedral mesh representing a volumetric field [S1, unofficial].
*   **Output:** A 2D image buffer with accumulated color and opacity [G].

**Data Structures:**
1.  **Partitioned Mesh:** The input mesh is preprocessed and partitioned into distinct topological sets [S1, unofficial].
2.  **Splat Representation:** Each tetrahedron $T_j$ is mapped to a single splat primitive [S1, unofficial].
    *   **Spherical Splat:** Defined by a center $\mathbf{c}_j$ and radius $r_j$ [G].
    *   **Ellipsoid Splat:** Defined by a center $\mathbf{c}_j$ and a $3 \times 3$ covariance matrix $\Sigma_j$ [G].
3.  **Hierarchical Culling Structure:** A spatial hierarchy (e.g., BVH or octree) built over the splats to enable efficient back-face and occlusion culling [S1, unofficial].

**Preprocessing (Static Assumption):**
1.  **Degeneracy Removal:** Identify and eliminate degenerated tetrahedra (needles, spindles, slivers, and caps) using edge collapse [S1, unofficial].
2.  **Partitioning:** Separate the remaining tetrahedra into topological sets for rendering [S1, unofficial].
3.  **Splat Generation:** Compute the center $\mathbf{c}_j$ (e.g., centroid) and shape parameters (radius or covariance) for each tetrahedron [G].
4.  **Hierarchy Construction:** Build the hierarchical data structure for culling [G].

**Rendering Steps:**
1.  **Culling:** Use the hierarchical structure to cull splats that are back-facing or occluded by previously rendered closer splats [S1, unofficial].
2.  **Projection:** For each remaining splat, transform its center $\mathbf{c}_j$ and shape parameters into screen space using the Model-View-Projection (MVP) matrix [G].
3.  **Footprint Evaluation:** For each pixel $(u,v)$ within the screen-space bounding box of the splat, evaluate the footprint function $F(u,v)$ [G].
    *   For a spherical splat with screen-space radius $R$:
        $$ F(u,v) = \max\left(0, 1 - \frac{(u - u_c)^2 + (v - v_c)^2}{R^2}\right) $$
    *   For an ellipsoid splat, the distance metric is modified by the projected covariance matrix [G].
4.  **Compositing:** Accumulate the splat's contribution to the image buffer using alpha blending [G]. Assuming the splat represents a density or color value $I_j$ and opacity $\alpha_j$:
    $$ C_{out}(u,v) = C_{in}(u,v) + I_j \cdot F(u,v) \cdot \alpha_j \cdot (1 - A_{in}(u,v)) $$
    $$ A_{out}(u,v) = A_{in}(u,v) + \alpha_j \cdot F(u,v) \cdot (1 - A_{in}(u,v)) $$
    where $C$ is color, $A$ is accumulated opacity, and $(u_c, v_c)$ is the screen-space center [G].

**Parameters:**
*   **Splat Shape:** Spherical or Ellipsoid [S1, unofficial].
*   **Hierarchy Depth:** Not given in the sources. [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Degenerate Tetrahedra:** The method explicitly removes degenerate tetrahedra [S1, unofficial]. This is a significant limitation for dynamic volumes like smoke, where topology must be maintained and tetrahedra naturally approach degenerate states during integration [S1, unofficial].
*   **Visible Gaps:** Since each tetrahedron is represented by exactly one splat, small meshes or poorly adapted splat shapes result in visible gaps between splats, degrading visual quality [S1, unofficial].
*   **Static Mesh Requirement:** The preprocessing step assumes a static mesh [S1, unofficial]. The method is unsuitable for dynamic objects in its current implementation [S1, unofficial].

## Subtopics

### [TetSplat Pre-processing: Mesh Partitioning](7483-tetsplat-pre-processing-mesh-partitioning.md)
This method partitions a large unstructured tetrahedral mesh into two distinct topological parts: the Shell (non-convex boundary) and the Solid (internal fragments). This separation allows the Shell to be rendered as a standard opaque surface mesh with vertex normals, while the Solid is rendered using normals derived from a Constructive Solid Geometry (CSG) probe, as interior tetrahedra are only visible at the intersection surface.

## Operating characteristics
*   **Time Complexity:** Rendering time is dominated by the number of visible splats and the cost of footprint evaluation per pixel [G]. Hierarchical culling reduces the number of splats processed [S1, unofficial]. Footprint evaluation is $O(1)$ per pixel per splat [G].
*   **Memory Complexity:** Linear in the number of tetrahedra $N$ for storing splat parameters and the hierarchy [G].
*   **Measured Performance:** Enables real-time rendering of large tetrahedral meshes [S1, unofficial].
*   **Strengths:** High performance for static volumes; simple footprint functions allow efficient rasterization; hierarchical culling reduces overdraw [S1, unofficial].
*   **Limitations:** Requires static meshes; preprocessing removes topological details (degenerate tetrahedra); single-splat-per-tetrahedron representation leads to visible gaps and loss of geometric fidelity; not suitable for dynamic smoke volumes where topology changes [S1, unofficial].
*   **When to use:** Real-time visualization of large, static tetrahedral meshes where performance is critical and some geometric approximation is acceptable [S1, unofficial].
*   **When not to use:** Dynamic volumes (e.g., smoke simulation), scenes requiring high geometric fidelity, or meshes with many degenerate elements that cannot be removed without altering topology [S1, unofficial].

## Background
Tetrahedral Splatting (TetSplat) was introduced as a point-based technique for real-time rendering of large tetrahedral meshes [S1, unofficial]. It addresses the limitations of exact volume integration methods by approximating volumetric primitives with splats [S1, unofficial]. The method relies on a preprocessing step that partitions the volume and removes degenerate elements, making it suitable for static scenes [S1, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
