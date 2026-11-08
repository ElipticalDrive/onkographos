# Tetrahedral Splatting

Also called: TetSplat

## Overview
Tetrahedral Splatting is a point-based technique for the real-time rendering of large, unstructured tetrahedral meshes [S1, unofficial]. It solves the problem of efficiently visualizing volumetric data defined by tetrahedral elements by approximating each tetrahedron with a single spherical or ellipsoidal splat [S1, unofficial]. It distinguishes itself from sibling methods like [Voxel Splat Rendering](3730-voxel-splat-rendering.md) by operating directly on unstructured geometric primitives rather than regular grids, and from [Triangle Mesh Splatting](8109-triangle-mesh-splatting.md) by representing volume interiors rather than just surfaces [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A static tetrahedral mesh defining the volume.
*   **Output:** A rendered 2D image of the volume.

**Data Structures:**
*   **Tetrahedron Representation:** Each tetrahedron $T_i$ is represented by a single splat primitive.
*   **Splat Geometry:** The splat is defined as a sphere or an ellipsoid. The position is typically the centroid or a representative point of the tetrahedron. The shape parameters (radius for sphere, axes for ellipsoid) are derived from the tetrahedron's geometry to approximate its volume [S1, unofficial].
*   **Hierarchical Structure:** A hierarchical data structure (e.g., a bounding volume hierarchy or octree) is built over the splats to enable efficient culling of back faces and occluded primitives [S1, unofficial].

**Preprocessing (Static Mesh Only):**
1.  **Degeneracy Removal:** The mesh is analyzed to identify degenerated tetrahedra (needles, spindles, slivers, and caps) [S1, unofficial].
2.  **Edge Collapse:** Degenerated tetrahedra are removed from the dataset using edge collapse operations [S1, unofficial]. This step is critical because degenerated shapes are difficult to represent accurately with standard splat kernels [S1, unofficial].
3.  **Splat Generation:** For each remaining tetrahedron, a corresponding spherical or ellipsoidal splat is generated [S1, unofficial].
4.  **Hierarchy Construction:** A hierarchical culling structure is built over the generated splats [S1, unofficial].

**Rendering Steps:**
1.  **Culling:** Using the hierarchical structure, splats that are back-facing or occluded by closer geometry are culled [S1, unofficial].
2.  **Transformation:** The remaining splats are transformed from object space to screen space using the Model-View-Projection (MVP) matrix [S1, unofficial].
3.  **Rasterization:** The transformed splats are rasterized onto the screen. Each splat contributes to the pixels within its footprint [S1, unofficial].
4.  **Compositing:** The contributions from all visible splats are accumulated to form the final image [S1, unofficial].

**Parameters:**
*   **Splat Shape:** Spherical or Ellipsoidal. Ellipsoids may be used to better fit the aspect ratio of the tetrahedron, but this increases complexity [S1, unofficial].
*   **Mesh Type:** Static. The method assumes the mesh does not change during rendering [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Tetrahedra:** The removal of degenerate tetrahedra is incompatible with applications requiring strict topological preservation, such as smoke volume rendering, where tetrahedra may naturally become near-degenerate during integration [S1, unofficial].
*   **Visible Gaps:** Representing a tetrahedron with a single splat can lead to visible gaps between splats, especially in small meshes, even when using ellipsoidal shapes [S1, unofficial].
*   **Dynamic Meshes:** The preprocessing step assumes a static mesh; dynamic objects are not supported in the current implementation [S1, unofficial].

## Subtopics

### [TetSplat Pre-processing: Mesh Partitioning](7483-tetsplat-pre-processing-mesh-partitioning.md)
Partitions the mesh into a Shell (boundary) and Solid (interior) to handle shading discontinuities.

## Operating characteristics
*   **Time Complexity:** Rendering time is dominated by the rasterization of splats and hierarchical culling. Culling efficiency depends on the hierarchy depth and scene complexity.
*   **Memory Complexity:** Linear in the number of tetrahedra, plus overhead for the hierarchical structure.
*   **Performance:** Enables real-time rendering of large tetrahedral meshes [S1, unofficial].
*   **Strengths:** Efficient culling via hierarchical structures; direct handling of unstructured meshes [S1, unofficial].
*   **Limitations:** Requires static meshes; preprocessing removes topological details (degenerate tetrahedra); single-splat-per-tetrahedron representation can cause visible artifacts (gaps) [S1, unofficial].
*   **When to Use:** Real-time visualization of static unstructured tetrahedral volumes where topological fidelity of degenerate elements is not critical.
*   **When Not to Use:** Dynamic meshes; smoke volume rendering requiring topological preservation; scenes where high-fidelity representation of individual tetrahedra is required without interpolation artifacts.

## Background
Tetrahedral Splatting (TetSplat) was introduced as a point-based technique for real-time rendering of large tetrahedral meshes [S1, unofficial]. It utilizes spherical or ellipsoidal splat representations and hierarchical data structures for efficient culling [S1, unofficial]. The method was presented in the context of smoke volume rendering, though its preprocessing requirements limit its applicability to dynamic or topologically sensitive volumes [S1, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
