# Tetrahedral Strips and Fans

Also called: Bandwidth reduction for tetrahedral meshes, Rendering Pattern and Stripification, Tetrahedral Fans, Tetrahedral Fans and Tetrahedral Strips, Tetrahedral Strips, Tetrahedral primitives

## Overview
Tetrahedral Strips and Fans are graphics primitives designed to address the bandwidth bottleneck between main memory and the graphics processor when rendering unstructured tetrahedral meshes [S2, unofficial]. They reduce the number of vertices transmitted per tetrahedron, achieving rates approaching 1.2 vertices per tetrahedron, similar to how triangle strips optimize triangle meshes [S2, unofficial]. Unlike static triangle strips, these primitives are necessary because tetrahedral decomposition is view-dependent, preventing the use of static display lists when changing viewpoints [S2, unofficial]. They enable performance improvements of 72 to 85 times over triangle-based hardware approaches, making interactive volumetric rendering feasible [S1, unofficial].

## How it works
**Inputs:** A tetrahedral mesh (set of tetrahedra defined by vertex indices) [S2, unofficial].
**Outputs:** A sequence of drawing commands and vertex data optimized for transmission [S2, unofficial].

**Data Structures:**
The core data structure is a list of tetrahedra grouped into strips or fans [S2, unofficial]. Each tetrahedron is defined by 4 vertices [S2, unofficial].
*   **Tetrahedral Fan:** A sequence of tetrahedra sharing a common edge [S2, unofficial].
*   **Tetrahedral Strip:** A sequence of tetrahedra connected by shared faces, not necessarily sharing a single common edge [S2, unofficial].
*   **Vertex Slots:** For generalized strips, a buffer of 4 vertex slots is maintained [S2, unofficial].

**Algorithm Steps:**

1.  **Grouping:** Group tetrahedra into fans or strips based on adjacency [S2, unofficial].
    *   *Fan:* Identify a common edge shared by multiple tetrahedra [S2, unofficial].
    *   *Strip:* Identify a chain of tetrahedra where each shares a face with the next [S2, unofficial]. This often involves analyzing "edge chains" (unions of common edges shared by sequences of three or more adjacent tetrahedra) [S2, unofficial].
2.  **Ordering:** Determine the order of vertices for transmission to ensure correct geometry reconstruction [S2, unofficial].
    *   *Fan:* The first two vertices form the shared edge [S2, unofficial]. Subsequent vertices define new tetrahedra using the shared edge and the previous new vertex [S2, unofficial].
    *   *Strip:* Determine which vertex of the previous tetrahedron is replaced by the new vertex [S2, unofficial].
3.  **Transmission:** Send vertex data and primitive commands to the GPU [S2, unofficial].

**Formulas and Definitions:**

*   **Fan Transmission Cost:** For a fan comprising $N$ tetrahedra, only $N+2$ vertices are transmitted [S2, unofficial].
*   **Sequential Strip:** A strip where the new vertex always replaces the least-recently-used (LRU) vertex of the previous tetrahedron [S2, unofficial].
*   **Generalized Strip:** A strip with no constraints on the direction of the next tetrahedron; vertices may be replaced in any order [S2, unofficial].
*   **Generalized Strip Overhead:** Requires a 2-bit flag per vertex to indicate which of the four slots (1, 2, 3, or 4) is replaced [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Simulated hardware API for tetrahedral primitives
def start_primitive(primitive_type):
    """Initialize primitive state."""
    pass

def send_vertex(vertex):
    """Transmit a vertex to the graphics processor."""
    pass

def end_primitive():
    """Finalize primitive drawing."""
    pass

def send_replacement_flag(slot_index):
    """Transmit a 2-bit flag indicating which of the 4 vertex slots to replace."""
    pass

def render_tetrahedral_fan(vertices):
    """
    Renders a sequence of tetrahedra sharing a common edge.
    vertices: List of 3D points.
    Assumption: vertices[0] and vertices[1] form the shared edge.
    """
    start_primitive("TET_FAN")
    send_vertex(vertices[0])
    send_vertex(vertices[1])
    send_vertex(vertices[2])
    send_vertex(vertices[3])
    for k in range(4, len(vertices)):
        send_vertex(vertices[k])
    end_primitive()

def render_sequential_tet_strip(vertices):
    """
    Renders a sequential tetrahedral strip.
    vertices: List of 3D points.
    Assumption: Each new vertex replaces the least-recently-used (LRU) vertex of the previous tetrahedron.
    """
    start_primitive("SEQUENTIAL_TET_STRIP")
    send_vertex(vertices[0])
    send_vertex(vertices[1])
    send_vertex(vertices[2])
    send_vertex(vertices[3])
    for i in range(4, len(vertices)):
        send_vertex(vertices[i])
    end_primitive()

def render_generalized_tet_strip(vertices, replacement_flags):
    """
    Renders a generalized tetrahedral strip.
    vertices: List of 3D points.
    replacement_flags: List of integers (1, 2, 3, or 4) indicating which slot is replaced.
    """
    start_primitive("GENERAL_TET_STRIP")
    slots = [vertices[0], vertices[1], vertices[2], vertices[3]]
    send_vertex(vertices[0])
    send_vertex(vertices[1])
    send_vertex(vertices[2])
    send_vertex(vertices[3])
    for i, flag in enumerate(replacement_flags):
        send_replacement_flag(flag)
        new_vertex = vertices[4 + i]
        slots[flag - 1] = new_vertex
        send_vertex(new_vertex)
    end_primitive()
```

**Parameters:**
*   `tmean`: Mean number of tetrahedra in each strip [S2, unofficial].
*   `tmed`: Median number of tetrahedra in each strip [S2, unofficial].
*   `tstd`: Standard deviation of the number of tetrahedra in each strip [S2, unofficial].
*   `tmax`: Maximum number of tetrahedra in a strip [S2, unofficial].
*   `vpert`: Number of vertices per tetrahedron (transmission rate) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **View Dependency:** Static strips cannot be used because the projection of tetrahedra changes with the viewpoint [S2, unofficial].
*   **Bandwidth:** Rendering tetrahedra requires 50-60% more bandwidth than rendering individual triangles, even with fans [S2, unofficial].
*   **Generalized Strip Overhead:** Using generalized strips incurs an added expense of transmitting 2 bits for the flag per vertex [S2, unofficial].

## Subtopics

### [Greedy Strip Generation Algorithm](2626-greedy-strip-generation-algorithm.md)
A method for converting a tetrahedral mesh into a sequence of tetrahedral strips and fans to optimize data transmission. It addresses the bandwidth bottleneck by reducing the number of vertices required to render each tetrahedron. Unlike triangle strips, tetrahedral strips require computing a correctly-ordered list of vertex replacements from the tetrahedra in the strip, often involving edge chain analysis. The baseline algorithm is distinguished by its simplicity: it selects the first available unvisited neighbor without complex heuristics, serving as the foundational reference for more advanced variants.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources. The generation of strips involves graph traversal and edge chain analysis, which is computationally intensive but performed on the CPU [S2, unofficial].

**Measured Performance:**
*   Performance improvements of 72 to 85 times over triangle graphics hardware approaches [S1, unofficial].
*   Transmission rates approaching 1.2 vertices per tetrahedron [S2, unofficial].
*   Strip lengths vary by method and dataset:
    *   Method 2 on Phoenix dataset: 441 strips, mean length 29.4, median 10, max 347, vpert 1.10 [S2, unofficial].
    *   Method 2 on Langley dataset: 1432 strips, mean length 49.0, median 13, max 865, vpert 1.06 [S2, unofficial].
    *   Average strip lengths of 17 on representative datasets [S1, unofficial].
    *   Method 2 achieved the best results with mean strip lengths of 29 tetrahedra per strip and median lengths over 10 tetrahedra per strip [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Significantly reduces bandwidth; enables interactive rendering; makes software sorting pass unnecessary by moving computations to hardware [S1, unofficial]. Gains are greater than expected for a direct analog to triangle stripiﬁcation [S2, unofficial].
*   **Limitations:** Requires hardware support for the new primitives, though the orderings can also be used for software compression [S2, unofficial].

**When to Use:**
*   Use for interactive tetrahedral volume rendering where bandwidth is a bottleneck [S2, unofficial].
*   Use when rendering unstructured volume data on hardware that supports vertex caching and the proposed primitives [S2, unofficial].

**When Not to Use:**
*   Not suitable for static meshes where triangle strips can be precomputed, as tetrahedral decomposition is view-dependent [S2, unofficial].

## Background
Tetrahedral Strips and Fans were introduced to address the bandwidth bottleneck in interactive tetrahedral volume rendering [S2, unofficial]. They build on the Projected Tetrahedra algorithm, which generates triangles for hardware acceleration but suffers from high bandwidth requirements [S2, unofficial]. The primitives were proposed in the context of a new hardware architecture (R-buffer) designed for volume rendering [S1, S2, unofficial]. The work was published by Craig M. Wittenbrink and colleagues [S2, unofficial]. The approach allows performance improvements similar to triangle strips for triangle meshes but adapted for the complexities of tetrahedral meshes [S1, unofficial].

## Sources
[S1] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S2] An Architecture For Interactive Tetrahedral Volume Rendering, https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)
[S3] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
