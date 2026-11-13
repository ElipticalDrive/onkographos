# 3D reconstruction based on edge growth


## Overview
3D reconstruction based on edge growth is a region-growing iso-surface extraction technique that propagates from seed triangles through a queue of growth edges [S1]. It addresses topological ambiguity and hole generation in 3D models by explicitly tracking iso-surface connectivity across cube boundaries and extracting only triangles connected to the current growth front [S1].

## How it works
**Inputs:** A 3D scalar field (volume data), an iso-surface threshold $Y$, and one or more seed triangles (manually selected or from a designated 3D area) [S1].
**Outputs:** A set of iso-surface triangles forming the reconstructed mesh [S1].

### Data Structures
*   **Cube Grid:** The volume is discretized into cubes. Each cube maintains two integer markers: `Processing Marker` (records how the cube was processed) and `Growth Marker` (records how the cube's growth information is put into the growth queue) [S1].
*   **Growth Queue:** Stores pending growth edge information, including edge type, endpoints, and associated cube index [S1].
*   **Growth Edge Types:** Triangle edges (endpoints on adjacent edges, forming a triangle) and quadrangle edges (endpoints on opposite sides, dividing the cube into two quadrangles) [S1].

### Interpolation
The algorithm uses a three-segment interpolation method, which lies between linear interpolation and midpoint selection [S1]. For an edge with endpoints $p_1, p_2$ having gray values $hu_1, hu_2$, and iso-surface threshold $Y$, the intersection point coordinates are calculated such that the intersection lies strictly between the two endpoints on that side [S1].

### Vertex Naming Conventions
To resolve topological configurations, vertices are named relative to the growth edge endpoints $I$ and $II$, and the surface containing the growth edge ($SS$) [S1]. The surface opposite to $SS$ is named $S\_S$ [S1].
*   **Triangle Edge Naming:** The vertex on $SS$ that forms a triangle with the growth edge is named $SS$ [S1]. The vertices on $SS$ adjacent to $SS$ are named $SS\_I$ and $SS\_II$ [S1]. The vertex on $SS$ diagonal to $SS$ is named $SS\_I\_II$ [S1]. Vertices on $S\_S$ share the same suffix as their adjacent counterparts on $SS$ [S1].
*   **Quadrangle Edge Naming:** On $SS$, the vertex adjacent to $I$ that meets the threshold requirement is named $SS\_I$, while the adjacent vertex that does not meet the requirement is named $SS\_UP\_I$ [S1]. The vertex adjacent to $II$ that meets the requirement is named $SS\_II$, while the adjacent vertex that does not meet the requirement is named $SS\_UP\_II$ [S1]. Vertices on $S\_S$ share the same suffix as their adjacent counterparts on $SS$ [S1].

### Topological Configurations
The algorithm defines 17 specific topological configurations to handle the extraction of iso-surface triangles connected to the growth edge [S1].
*   **Triangle Edge Configurations (11 types):** Select a reference point $SS$. Compare its gray value $hu_{SS}$ with threshold $Y$ [S1]. If $hu_{SS} > Y$, vertices with $hu > Y$ "meet requirements" (large dots); others do not (small dots) [S1]. If $hu_{SS} \le Y$, vertices with $hu \le Y$ "meet requirements"; others do not [S1]. Based on the pattern of meeting/non-meeting vertices, one of 11 configurations is selected to extract the connected triangles [S1].
*   **Quadrangle Edge Configurations (6 types):** Compare all 8 cube vertices with threshold $Y$ [S1]. Divide vertices into "meeting requirements" and "not meeting requirements" [S1]. Select the smaller set (size 1 to 4) as the reference set [S1]. Apply the naming convention defined above [S1]. Based on the pattern, one of 6 configurations is selected to extract the iso-surface [S1].

### Cube Markers and Growth Logic
Each cube has two markers with levels 0–4 [S1]. Level 0 indicates the cube has not been processed [S1]. Level 4 indicates the cube has been processed and no further processing is required [S1]. Levels 1–3 indicate the cube has been processed but may still need processing [S1].
*   **Marker Assignment Rules:**
    1.  After extracting the seed triangle, set both markers of the corresponding cube to 4 [S1].
    2.  When growth edge information is added to the growth queue, increment the cube's `Growth Marker` by 1 [S1].
    3.  After processing a cube, set the `Processing Marker` based on the topological configuration used [S1]:
        *   Set to **4** if configuration is: 4, 5, 7, 8, 10, 11, 13, 15, or 17 [S1].
        *   Set to **3** if configuration is: 2, 3, 6, 9, 12, 14, or 16 [S1].
        *   Increment by **1** if configuration is: 1 [S1].
*   **New Growth Edge Detection:** After interpolation, determine if the intersection of the iso-surface and the cube surface forms a new growth edge [S1]. It is a new growth edge if the edge's processing label and the growth label of adjacent cubes sharing the side are both not 4 [S1].
*   **Processing Constraints:**
    *   If `Processing Marker` is 4, skip further processing of the cube [S1].
    *   If `Growth Marker` is 4, do not add growth information to the growth queue [S1].
    *   If `Processing Marker` is >1 and <4, only extract iso-surface for topological configuration 1 [S1].

### Algorithm Steps
1.  Initialize all cube markers to 0 [S1].
2.  Initialize an empty growth queue [S1].
3.  Select a seed triangle (manually or from a designated area) [S1].
4.  Extract the iso-surface for the seed triangle using the appropriate configuration [S1].
5.  Set the `Processing Marker` and `Growth Marker` of the seed cube to 4 [S1].
6.  While the growth queue is not empty:
    a.  Dequeue a growth edge (type, endpoints $I, II$, cube index) [S1].
    b.  Check if the cube's `Growth Marker` is 4. If so, skip [S1].
    c.  Check if the cube's `Processing Marker` is 4. If so, skip [S1].
    d.  Determine the edge type (Triangle or Quadrangle) [S1].
    e.  Apply the naming convention for the cube vertices [S1].
    f.  Determine the topological configuration based on vertex values relative to $Y$ [S1].
    g.  Extract the iso-surface triangles for this configuration [S1].
    h.  Update the cube's `Processing Marker` according to the configuration rules [S1].
    i.  Check adjacent cube faces for new growth edges [S1].
    j.  If a new growth edge is found and adjacent cubes' growth labels are not 4:
        i.  Enqueue the new growth edge [S1].
        ii. Increment the cube's `Growth Marker` [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Solves topological ambiguity and hole generation common in 3D reconstruction by ensuring connected iso-surface extraction [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The method originates from a 2021 paper on a marching cube algorithm based on edge growth [S1]. It addresses the issue of ambiguous topology configurations that lead to holes in 3D models [S1].

## Sources
[S1] A marching cube algorithm based on edge growth, https://doi.org/10.1016/j.vrih.2021.08.006, fetched 2026-10-26
