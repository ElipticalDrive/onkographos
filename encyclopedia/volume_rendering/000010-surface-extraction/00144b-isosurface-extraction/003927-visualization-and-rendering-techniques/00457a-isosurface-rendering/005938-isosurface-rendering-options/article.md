# Isosurface Rendering Options

Also called: Rendering Options

## Overview
Isosurface Rendering Options provide configurable parameters for the visual presentation and performance optimization of polygonal meshes extracted from volumetric data [S1, unofficial]. This method addresses the need for user-specified trade-offs between rendering speed and geometric accuracy, allowing for efficient navigation of large datasets or high-fidelity visualization of scientific results [S1, unofficial]. It is distinct from [Isosurface Transparency via Alpha Blending](15614-isosurface-transparency-via-alpha-blending.md), which focuses on volumetric transparency rather than surface mesh rendering properties [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A polygonal triangle list representing the isosurface mesh.
- Output: Rendered image with applied shading, lighting, and optimized geometry buffers.

**Data Structures:**
- **Hash Table:** Maps vertex indices to accumulated normal vectors for smooth shading [S1, unofficial].
- **Triangle Strip Buffer:** Contiguous array of vertex indices representing the mesh as triangle strips for compact storage and faster GPU upload [S1, unofficial].

**Numbered Steps:**
1. Parse the input triangle list.
2. Compute face normals for each triangle.
3. Accumulate face normals into the hash table for shared vertices.
4. Normalize accumulated normals and assign them to vertices.
5. Optionally convert the triangle list to triangle strips using vertex lookup.
6. Optionally compile geometry into display lists.
7. Apply the selected shading mode during rasterization.

**Formulas:**
- Face normal: $n_i = (v_1 - v_0) \times (v_2 - v_0)$, where $v_0, v_1, v_2$ are triangle vertices.
- Accumulated normal: $n_{acc}(v) = \sum_{i \in \text{triangles sharing } v} n_i$.
- Vertex normal: $\hat{n}(v) = \frac{n_{acc}(v)}{\|n_{acc}(v)\|}$.

**Parameters:**
1. **Normal Direction:** Boolean toggle to reverse the direction of normals generated for isosurface triangles [S1, unofficial].
2. **Shading Mode:** Enum selecting wire frame, flat shading, or smooth (Gouraud) shading [S1, unofficial].
3. **Display Lists:** Boolean flag to use display lists to increase rendering speed [S1, unofficial].
4. **Triangle Strips:** Boolean flag to use triangle strips instead of individual triangles for more compact representation and faster rendering, at the cost of extra computation time during creation [S1, unofficial].

**Pseudocode for Smooth Shading:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_smooth_normals(triangles):
    hash_table = {}  # Maps vertex index to accumulated normal vector
    for tri in triangles:
        v0, v1, v2 = tri.vertices
        face_normal = cross(v1 - v0, v2 - v0)
        for v in (v0, v1, v2):
            if v not in hash_table:
                hash_table[v] = face_normal
            else:
                hash_table[v] = hash_table[v] + face_normal
    for v in hash_table:
        hash_table[v] = normalize(hash_table[v])
    return hash_table
```

**Pseudocode for Triangle Strip Generation:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_triangle_strips(triangles):
    strip_indices = []
    last_two = None
    for tri in triangles:
        v0, v1, v2 = tri.indices
        if last_two is None:
            strip_indices.extend([v0, v1, v2])
            last_two = (v1, v2)
        else:
            if last_two[0] == v0:
                strip_indices.extend([v2])
                last_two = (v1, v2)
            else:
                strip_indices.extend([v1])
                last_two = (v0, v1)
    return strip_indices
```

**Edge Cases and Pitfalls:**
- Incorrect Normal Direction toggle may cause incorrect lighting under standard models [S1, unofficial].
- Hash table computation and strip generation add extra CPU time during mesh creation [S1, unofficial].
- Periodic boundaries may wrap isosurfaces, creating stretched triangles; clipping must be enabled to prevent artifacts [S2, unofficial].

## Subtopics

### Cell-Boundary Clipping for Periodic Grids
A rendering setting that clips isosurfaces at the boundaries of the simulation cell rather than wrapping them across periodic boundaries [S2, unofficial]. This matches the behavior of VESTA software and prevents visual artifacts caused by stretched triangles across cell edges [S2, unofficial]. Controlled by the `periodic` boolean flag; set to `false` to enable clipping [S2, unofficial].

## Operating characteristics
- **Trade-offs:** Users can prioritize speed for interactive navigation or accuracy for final visualization [S1, unofficial].
- **Speed:** Wire frame rendering is chosen for quick navigation through datasets [S1, unofficial]. Smooth shading with display lists provides precise surface rendering [S1, unofficial].
- **Generation Cost:** Triangle strip generation and smooth normal computation require additional CPU time during mesh creation [S1, unofficial].
- **Periodic Handling:** Clipping at cell boundaries avoids wrapping artifacts in periodic grids [S2, unofficial].

## Background
The options were introduced in the DYNAISOLOD system, a component of a scientific data analysis toolkit designed for dynamic isosurface extraction and level-of-detail in voxel space [S1, unofficial]. The cell-boundary clipping option was added to MatterViz to align with VESTA's behavior for charge density isosurfaces [S2, unofficial].

## Sources
[S1] Dynamic Isosurface Extraction and Level-of-Detail in Voxel Space, https://digital.library.unt.edu/ark:/67531/metadc694729/?q=marching cubes, fetched 2026-10-28 (unofficial)
[S2] Isosurface performance: grid downsampling, debounced rendering, UI fixes, https://github.com/janosh/matterviz/pull/322, fetched 2026-10-27 (unofficial)
