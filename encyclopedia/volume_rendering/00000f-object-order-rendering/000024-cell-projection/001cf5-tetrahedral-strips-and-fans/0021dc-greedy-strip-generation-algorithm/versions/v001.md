# Greedy Strip Generation Algorithm

Also called: Baseline Greedy Strip Generation, Simple Greedy Algorithm

## Overview
The Greedy Strip Generation Algorithm is a method for converting a tetrahedral mesh into a sequence of tetrahedral strips and fans to optimize data transmission to graphics hardware. It addresses the bandwidth bottleneck between main memory and the GPU by reducing the number of vertices required to render each tetrahedron, similar to how triangle strips optimize triangle meshes. Unlike triangle strips, tetrahedral strips require computing a correctly-ordered list of vertex replacements from the tetrahedra in the strip, often involving edge chain analysis. The baseline algorithm is distinguished by its simplicity: it selects the first available unvisited neighbor without complex heuristics, serving as the foundational reference for more advanced variants.

## How it works
**Inputs:** A tetrahedral mesh defined by its connectivity.
**Outputs:** A list of tetrahedral strips and fans, where each strip is a sequence of tetrahedra sharing faces in a specific order to minimize vertex transmission.

**Data Structures:**
1.  **Adjacency Array:** An array where the $i$-th row contains four integers identifying the four face neighbors of tetrahedron $i$. The convention used is that the tetrahedron referred to by the first element in the $i$-th row is the tetrahedron opposite the first vertex of tetrahedron $i$ [S1, unofficial].
2.  **Visit Flags:** Each tetrahedron is marked with a boolean flag indicating whether it has been visited.
3.  **Unvisited Neighbor Count:** Each tetrahedron stores an integer indicating the number of its neighbors that have not yet been visited.

**Algorithm Steps:**
1.  **Initialization:** Mark all tetrahedra as unvisited. Compute the initial count of unvisited neighbors for each tetrahedron based on the adjacency array.
2.  **Start Selection:** Select the first unvisited tetrahedron from the adjacency array to serve as the start of the current strip [S1, unofficial].
3.  **Greedy Extension:**
    *   Let $T_{current}$ be the last tetrahedron added to the strip.
    *   Identify the unvisited neighbors of $T_{current}$.
    *   If unvisited neighbors exist, select the **first** unvisited neighbor (based on the order in the adjacency array) as the next tetrahedron $T_{next}$ in the strip.
    *   Mark $T_{next}$ as visited.
    *   Update the unvisited neighbor counts for all neighbors of $T_{next}$ (decrement by 1).
    *   Append $T_{next}$ to the strip and set $T_{current} = T_{next}$.
    *   Repeat this step until $T_{current}$ has no unvisited neighbors.
4.  **Reverse and Extend:**
    *   Once no unvisited neighbors remain, reverse the order of the tetrahedra in the current strip.
    *   Seek to extend the strip by starting again at the very first tetrahedron of the reversed strip (which is now the end of the original sequence).
    *   Attempt to add new unvisited neighbors to this end. If successful, continue extending. If not, finalize the strip and return to Step 2 to find a new start for the next strip from the remaining unvisited tetrahedra.

**Edge Chain Analysis for Vertex Ordering:**
To render the strip, the vertex replacement order must be determined using the edge chain, defined as the union of common edges shared by sequences of three or more adjacent tetrahedra in the strip [S1, unofficial].
*   **Fan:** The edge chain has a single shared edge. Vertex ordering is computed by ordering the remaining vertices clockwise or counterclockwise around this shared edge [S1, unofficial].
*   **Sequential Strip:** The edge chain has no branches and no edges shared by more than three consecutive tetrahedra. The edge chain visits all but the first and last vertex of the strip in the correct transmission order [S1, unofficial].
*   **Generalized Strip:** The edge chain has a branch or shared edge for each replacement that does not follow the Least Recently Used (LRU) order [S1, unofficial].

**Parameters and Performance:**
*   On sample meshes, this baseline algorithm produces mean strip sizes of 9–14 tetrahedra [S1, unofficial].
*   This corresponds to a transmission rate of 1.20–1.33 vertices per tetrahedron [S1, unofficial].

## Subtopics

### Edge Chain Analysis
A method for determining the correct vertex transmission order for a generated strip by analyzing the geometry of shared edges. It distinguishes between fans (single shared edge, vertices ordered around it), sequential strips (no branches, edge chain visits intermediate vertices in order), and generalized strips (branches or shared edges indicate deviations from LRU order) [S1, unofficial].

### Heuristic Strip Generation
An improved variant of the greedy algorithm that changes the selection criterion for the next tetrahedron. Instead of choosing the first unvisited neighbor, it selects the neighbor with the fewest unvisited neighbors. This heuristic boosts the average strip length up to 49 tetrahedra and is typically more effective than the baseline because it helps avoid leaving isolated singleton strips in the mesh [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of tetrahedra, as each tetrahedron is visited once and adjacency lookups are constant time.
*   **Memory Complexity:** Linear, requiring storage for the adjacency array and visit flags/counts.
*   **Performance:** The baseline algorithm yields modest compression (1.20–1.33 vertices/tetrahedron). Heuristic variants can significantly improve strip length (up to 49 tetrahedra) but require more complex neighbor selection logic [S1, unofficial].
*   **Strengths:** Simple to implement; requires no sorting pass; moves order-independent transparency computations to hardware.
*   **Limitations:** Baseline performance is limited by the "first available" selection strategy, which can lead to fragmented strips and isolated singletons.

## Background
The algorithm was introduced in the context of interactive tetrahedral volume rendering to address the bandwidth bottleneck between main memory and the graphics processor. It is described as the tetrahedral equivalent of the greedy algorithm for triangle strip generation used by Akeley et al. and Evans et al. [S1, unofficial]. The work is part of an architecture designed for interactive rendering, leveraging tetrahedral strips and fans to eliminate the need for software sorting.

## Sources
[S1] An Architecture For Interactive Tetrahedral Volume Rendering, https://www.researchgate.net/publication/221257851_An_Architecture_For_Interactive_Tetrahedral_Volume_Rendering, fetched 2026-11-01 (unofficial)
