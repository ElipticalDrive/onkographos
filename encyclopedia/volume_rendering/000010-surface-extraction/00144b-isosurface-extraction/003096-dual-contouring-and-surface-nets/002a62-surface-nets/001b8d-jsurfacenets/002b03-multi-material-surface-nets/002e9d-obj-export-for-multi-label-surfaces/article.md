# OBJ Export for Multi-Label Surfaces

Also called: Export to OBJ, OBJ Export

## Overview
This method converts the multi-label volumetric data produced by [Multi-Material Surface Nets](4930-multi-material-surface-nets.md) into a standard Wavefront OBJ file format. It solves the problem of visualizing and processing segmented materials by exporting each material as a distinct, closed surface mesh. It differs from single-material exporters by organizing geometry per material label and ensuring consistent vertex winding to support rendering logic that distinguishes between inside and outside faces [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The multi-label SurfaceNets data structure, where each material label has an associated set of vertices and quadrilateral faces.
*   **Output:** A `.obj` file containing geometry for all materials.

**Data Structures:**
The source data for each material is organized as follows [S1, unofficial]:
1.  **Vertex List:** A `std::vector` of floating-point coordinates (e.g., `glm::vec3` or `float[3]`).
2.  **Index List:** A `std::vector` of integer indices. Each index references a vertex in the vertex list.

**Steps:**
1.  **Iterate Materials:** Loop through each unique material label present in the segmentation.
2.  **Initialize OBJ Header:** For each material, write the OBJ header line `o Material_<Label>` to identify the object group.
3.  **Export Vertices:** Iterate through the vertex list for the current material. For each vertex position $(x, y, z)$, write a line:
    ```
    v x y z
    ```
4.  **Export Triangles:** Each quadrilateral face in the SurfaceNets output must be partitioned into two triangles.
    *   Let the four vertices of a quadrilateral be $V_0, V_1, V_2, V_3$ in counterclockwise order.
    *   The choice of diagonal partitioning affects the local surface shape, particularly for non-planar quads [S1, unofficial]. A common stable partition is $(V_0, V_1, V_2)$ and $(V_0, V_2, V_3)$.
    *   For each triangle, write a line using 1-based indexing (standard for OBJ):
        ```
        f i1 i2 i3
        ```
        Where $i1, i2, i3$ are the 1-based indices of the triangle vertices in the vertex list.
5.  **Consistent Winding:** Ensure all vertices are ordered consistently in a counterclockwise (CCW) direction relative to the outward normal. This allows rendering engines to distinguish between inside and outside faces, which is critical for correct shading and culling [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def export_multi_label_obj(materials, output_path):
    """
    materials: dict mapping label_id -> { 'vertices': list of (x,y,z), 'quads': list of (v0_idx, v1_idx, v2_idx, v3_idx) }
    """
    with open(output_path, 'w') as f:
        for label_id, data in materials.items():
            # Write object header
            f.write(f"o Material_{label_id}\n")
            
            # Write vertices
            for v in data['vertices']:
                f.write(f"v {v[0]} {v[1]} {v[2]}\n")
            
            # Write triangles
            # Note: OBJ indices are 1-based
            for quad in data['quads']:
                v0, v1, v2, v3 = quad
                # Partition quad into two triangles
                # Triangle 1: v0, v1, v2
                f.write(f"f {v0+1} {v1+1} {v2+1}\n")
                # Triangle 2: v0, v2, v3
                f.write(f"f {v0+1} {v2+1} {v3+1}\n")
```

**Parameters:**
*   **Partitioning Strategy:** Not explicitly defined in the source, but noted as affecting shape for non-planar quads [S1, unofficial].
*   **Index Base:** 1-based (standard OBJ convention).

**Edge Cases:**
*   **Non-planar Quadrilaterals:** The choice of diagonal for splitting quads can distort the surface if the quad is highly non-planar (e.g., near edges) [S1, unofficial].
*   **Empty Materials:** If a material label has no vertices, no object header or geometry should be written.

## Subtopics

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of vertices and faces across all materials.
*   **Memory Complexity:** Linear with respect to the output file size.
*   **Strengths:** Standard compatibility with rendering and geometry processing tools; preserves material separation.
*   **Limitations:** OBJ format does not natively support multi-material objects in a single mesh block without material libraries (`.mtl`); this method exports separate object groups.
*   **When to use:** When exporting segmented volumes for visualization in standard 3D viewers or further geometry processing.
*   **When not to use:** When real-time rendering of all materials in a single draw call is required (consider GPU buffers or other formats).

## Background
This export method is part of the SurfaceNets for Multi-Label Segmentations library, which focuses on preserving sharp boundaries in volumetric segmentations [S1, unofficial]. The OBJ format is a standard representation used for rendering and geometry processing [S1, unofficial].

## Sources
[S1] SurfaceNets for Multi-Label Segmentations with Preservation of Sharp Boundaries., https://europepmc.org/article/MED/36325473, fetched 2026-10-26 (unofficial)
