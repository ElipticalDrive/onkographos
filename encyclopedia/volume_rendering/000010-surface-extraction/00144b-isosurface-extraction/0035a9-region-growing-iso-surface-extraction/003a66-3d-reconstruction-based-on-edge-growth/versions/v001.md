# 3D reconstruction based on edge growth

Also called: Marching cube algorithm based on edge growth

## Overview
3D reconstruction based on edge growth is a region-growing iso-surface extraction technique that reconstructs surfaces by propagating from seed triangles through a queue of "growth edges" [S1]. It solves the topological ambiguity and hole-generation problems inherent in standard Marching Cubes by explicitly tracking the connectivity of the iso-surface across cube boundaries [S1]. Unlike standard methods that process cubes independently, this method ensures global consistency by only extracting triangles connected to the current growth front, using specific topological configurations for triangle and quadrangle edges [S1].

## How it works
**Inputs:** A 3D scalar field (volume data), an iso-surface threshold $Y$, and one or more seed triangles (manually selected or from a designated 3D area) [S1].
**Outputs:** A set of iso-surface triangles forming the reconstructed mesh [S1].

### Data Structures
*   **Cube Grid:** The volume is discretized into cubes. Each cube maintains two integer markers:
    *   `Processing Marker`: Indicates the processing state (levels 0–4).
    *   `Growth Marker`: Indicates the status of growth information in the queue (levels 0–4) [S1].
*   **Growth Queue:** Stores pending growth edges. Each entry contains the edge type (triangle or quadrangle), the endpoints ($I, II$), and the associated cube index [S1].
*   **Growth Edge Types:**
    *   **Triangle Edge:** Endpoints lie on two adjacent edges of a cube face, forming a triangle with the face [S1].
    *   **Quadrangle Edge:** Endpoints lie on opposite edges of a cube face, dividing the face into two quadrangles [S1].

### Interpolation
The algorithm uses a three-segment interpolation method, which lies between linear interpolation and midpoint selection [S1]. For an edge with endpoints $p_1, p_2$ having gray values $hu_1, hu_2$, and iso-surface threshold $Y$, the intersection point is calculated such that the intersection lies strictly between the two endpoints on that side [S1].

### Vertex Naming Conventions
To resolve topological configurations, vertices are named relative to the growth edge and the surface containing it [S1].

**1. Triangle Edge Naming:**
*   Let the growth edge endpoints be $I$ and $II$.
*   Let $SS$ be the surface containing the growth edge.
*   Let $S\_S$ be the surface opposite to $SS$.
*   The vertex on $SS$ that forms a triangle with the growth edge is named $SS$.
*   The vertices on $SS$ adjacent to $SS$ are named $SS\_I$ and $SS\_II$.
*   The vertex on $SS$ diagonal to $SS$ is named $SS\_I\_II$.
*   Vertices on $S\_S$ share the same suffix as their adjacent counterparts on $SS$ (e.g., $S\_S$, $S\_S\_I$, etc.) [S1].

**2. Quadrangle Edge Naming:**
*   Let the growth edge endpoints be $I$ and $II$.
*   Let $SS$ be the surface containing the growth edge.
*   Let $S\_S$ be the surface opposite to $SS$.
*   Identify vertices on $SS$ adjacent to $I$:
    *   If the vertex meets the threshold requirement, name it $SS\_I$.
    *   If it does not meet the requirement, name it $SS\_UP\_I$.
*   Identify vertices on $SS$ adjacent to $II$:
    *   If the vertex meets the threshold requirement, name it $SS\_II$.
    *   If it does not meet the requirement, name it $SS\_UP\_II$.
*   Vertices on $S\_S$ share the same suffix as their adjacent counterparts on $SS$ [S1].

### Topological Configurations
The algorithm defines 17 specific topological configurations to handle the extraction of iso-surface triangles connected to the growth edge [S1].

**Triangle Edge Configurations (11 types):**
1.  Select a reference point $SS$.
2.  Compare $hu_{SS}$ with threshold $Y$.
    *   If $hu_{SS} > Y$: Vertices with $hu > Y$ "meet requirements" (large dots); others do not (small dots).
    *   If $hu_{SS} \le Y$: Vertices with $hu \le Y$ "meet requirements"; others do not [S1].
3.  Based on the pattern of meeting/non-meeting vertices, one of 11 configurations is selected to extract the connected triangles [S1].

**Quadrangle Edge Configurations (6 types):**
1.  Compare all 8 cube vertices with threshold $Y$.
2.  Divide vertices into "meeting requirements" and "not meeting requirements".
3.  Select the smaller set (size 1 to 4) as the reference set [S1].
4.  Apply the naming convention defined above.
5.  Based on the pattern, one of 6 configurations is selected to extract the iso-surface [S1].

### Cube Markers and Growth Logic
Each cube has two markers with levels 0–4 [S1]:
*   **Level 0:** Unprocessed.
*   **Level 4:** Fully processed; no further action required.
*   **Levels 1–3:** Processed but may require further processing [S1].

**Marker Assignment Rules:**
1.  **Seed Cube:** After extracting the seed triangle, set both markers of the corresponding cube to 4 [S1].
2.  **Growth Marker Increment:** When growth edge information is added to the growth queue, increment the cube's `Growth Marker` by 1 [S1].
3.  **Processing Marker Assignment:** After processing a cube, set the `Processing Marker` based on the topological configuration used [S1]:
    *   Set to **4** if configuration is: 4, 5, 7, 8, 10, 11, 13, 15, or 17.
    *   Set to **3** if configuration is: 2, 3, 6, 9, 12, 14, or 16.
    *   Increment by **1** if configuration is: 1.
4.  **New Growth Edge Detection:** After interpolation, determine if the intersection of the iso-surface and the cube surface forms a new growth edge. It is a new growth edge if:
    *   The edge's processing label is not 4.
    *   The growth label of adjacent cubes sharing the side is not 4 [S1].

**Processing Constraints:**
*   If `Processing Marker` is 4, skip further processing of the cube [S1].
*   If `Growth Marker` is 4, do not add growth information to the growth queue [S1].
*   If `Processing Marker` is >1 and <4, only extract iso-surface for topological configuration 1 [S1].

### Algorithm Steps
1.  Initialize all cube markers to 0.
2.  Initialize an empty growth queue.
3.  Select a seed triangle (manually or from a designated area) [S1].
4.  Extract the iso-surface for the seed triangle using the appropriate configuration.
5.  Set the `Processing Marker` and `Growth Marker` of the seed cube to 4 [S1].
6.  While the growth queue is not empty:
    a.  Dequeue a growth edge (type, endpoints $I, II$, cube index).
    b.  Check if the cube's `Growth Marker` is 4. If so, skip [S1].
    c.  Check if the cube's `Processing Marker` is 4. If so, skip [S1].
    d.  Determine the edge type (Triangle or Quadrangle) [S1].
    e.  Apply the naming convention for the cube vertices [S1].
    f.  Determine the topological configuration based on vertex values relative to $Y$ [S1].
    g.  Extract the iso-surface triangles for this configuration [S1].
    h.  Update the cube's `Processing Marker` according to the configuration rules [S1].
    i.  Check adjacent cube faces for new growth edges.
    j.  If a new growth edge is found and adjacent cubes' growth labels are not 4:
        i.  Enqueue the new growth edge.
        ii. Increment the cube's `Growth Marker` [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Solves topological ambiguity and hole generation common in standard Marching Cubes by ensuring connected iso-surface extraction [S1].
*   **Limitations:** Requires careful seed selection; complexity of managing 17 configurations and dual markers [S1].
*   **When to use:** When high-quality, hole-free iso-surfaces are required and topological ambiguity is a concern [S1].
*   **When not to use:** Not specified.

## Background
The method was proposed in a paper titled "A marching cube algorithm based on edge growth" [S1]. It addresses the issue of ambiguous topology configurations that lead to holes in 3D models, referencing Masala's solution of expanding the topological configurations from 15 to 21 as a related approach to handling ambiguity [S1].

## Sources
[S1] A marching cube algorithm based on edge growth, https://doi.org/10.1016/j.vrih.2021.08.006, fetched 2026-10-26
