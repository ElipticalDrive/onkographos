# Brute-Force Geometric Neighbor Triangulation

Also called: Brute-Force Edge Point Triangulation

## Overview
Brute-Force Geometric Neighbor Triangulation is a procedural algorithm for generating the Marching Cubes triangle lookup table by dynamically computing edge intersections and forming triangles based on geometric proximity for all 256 vertex configurations [S1, unofficial]. It differs from standard lookup tables by deriving topology through a "nearest neighbor" heuristic on edge points rather than relying on pre-defined topological cases, aiming to avoid intersection and duplication through explicit geometric rules [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A vertex configuration of a cube, represented as an 8-bit integer (0–255) where each bit indicates if a corner is "on" (1) or "off" (0) [S1, unofficial].
*   **Output:** A list of triangles, where each triangle is defined by three edge point indices [S1, unofficial].

**Data Structures:**
*   **Edge Point Array:** A 12-element integer array, initialized to 0, representing the 12 edges of the cube [S1, unofficial].
*   **Valid Vertex Indices:** A list storing indices of edge points that are enabled for the current configuration [S1, unofficial].
*   **Used Vertices:** A temporary list tracking vertices currently being assembled into a triangle [S1, unofficial].
*   **Old Vertices:** A list tracking vertices used in previously formed triangles to prevent reuse [S1, unofficial].
*   **Used Tris:** A list storing completed triangles (arrays of 3 indices) to check for duplicates [S1, unofficial].

**Parameters:**
*   **Vertex Position Grid:** The spatial coordinates of edge points are assumed to be perfect multiples of 0.5 (e.g., 0, 0.5, 1.0) to simplify distance calculations [S1, unofficial].

**Steps:**
1.  **Numbering:** Programmatically number the 8 cube corners and 12 edges in a consistent, reproducible manner [S1, unofficial].
2.  **Indexing:** Map the 256 possible vertex combinations to a binary index (on=1, off=0) [S1, unofficial].
3.  **Enable Edges:** For every corner point that is "on", mark its three adjacent edges as "enabled" in the 12-element array [S1, unofficial].
4.  **Resolve Overlaps:** Iterate through the edge array. If an edge was enabled more than once (i.e., both its adjacent corners are "on"), disable it (set to 0). If enabled exactly once, keep it enabled [S1, unofficial].
5.  **Collect Valid Points:** Store the indices of all enabled edges in `validVertexIndices` [S1, unofficial].
6.  **Triangle Formation:**
    *   Initialize `usedVertices` as empty [S1, unofficial].
    *   Start with the lowest index point in `validVertexIndices` as the first vertex of a new triangle [S1, unofficial].
    *   Iterate through all other points in `validVertexIndices` to find the nearest point (based on Euclidean distance, assuming 0.5 grid spacing) [S1, unofficial].
    *   Add this nearest point as the second vertex [S1, unofficial].
    *   Find the nearest point to the second vertex that is not the first or second vertex; add it as the third vertex [S1, unofficial].
    *   Register the triangle in `usedTris` and reset `usedVertices` for the next triangle [S1, unofficial].
    *   Repeat until $n(n-1)/2$ triangles are formed, where $n$ is the number of enabled edge points [S1, unofficial].
7.  **Normal Reversal:** For every generated triangle, create a clone with reversed vertex order to ensure correct normal orientation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def brute_force_triangulation(vertex_mask: int) -> List[Tuple[int, int, int]]:
    # 1. Number corners 0-7 and edges 0-11
    # Define adjacency: which edges connect to which corners
    # Example: edge 0 connects corner 0 and 1, etc.
    corner_to_edges = {
        0: [0, 4, 8], 1: [0, 5, 9], 2: [1, 4, 10], 3: [1, 5, 11],
        4: [2, 6, 8], 5: [2, 7, 9], 6: [3, 6, 10], 7: [3, 7, 11]
    }
    
    # 2. Initialize edge array
    edge_enabled = [0] * 12
    
    # 3. Enable edges for active corners
    for corner in range(8):
        if (vertex_mask >> corner) & 1:
            for edge in corner_to_edges[corner]:
                edge_enabled[edge] += 1
                
    # 4. Resolve overlaps: disable if enabled > 1
    valid_edges = []
    for edge in range(12):
        if edge_enabled[edge] == 1:
            valid_edges.append(edge)
            
    n = len(valid_edges)
    triangles = []
    
    # 5. Form triangles
    # Note: The source describes a greedy nearest-neighbor approach.
    # This is a simplified implementation of the described logic.
    used_indices = set()
    
    # We need to form triangles until we have n(n-1)/2 triangles?
    # The source says "Repeat until n(n-1)/2 triangles are formed".
    # This implies a specific pairing strategy or full triangulation of the valid edge set.
    # However, standard MC produces 1-2 triangles per case. 
    # The source's logic seems to describe a general triangulation of the valid edge points.
    
    # Re-reading source: "Start with the lowest index point... find nearest... add as second... find nearest to second... add as third."
    # This forms one triangle. Then "reset vertex list for next triangle".
    # "Repeat until n(n-1)/2 triangles are formed".
    
    # This logic is ambiguous for general n. For n=3, 3(2)/2 = 3 triangles? 
    # But a triangle has 3 vertices. 
    # The source likely implies a specific traversal order or that n is small in typical cases.
    # Given the "brute force" nature and the specific rules about intersection, 
    # this method is primarily for generating the table, not necessarily efficient rendering.
    
    # Implementation of the specific greedy step:
    remaining = list(valid_edges)
    while len(remaining) >= 3:
        # Start with lowest index
        p1 = remaining.pop(0)
        
        # Find nearest to p1
        p2 = None
        min_dist = float('inf')
        for p in remaining:
            dist = distance(p1, p) # Assuming 0.5 grid
            if dist < min_dist:
                min_dist = dist
                p2 = p
        
        if p2 is None:
            break
            
        remaining.remove(p2)
        
        # Find nearest to p2, not p1 or p2
        p3 = None
        min_dist = float('inf')
        for p in remaining:
            if p == p1 or p == p2:
                continue
            dist = distance(p2, p)
            if dist < min_dist:
                min_dist = dist
                p3 = p
                
        if p3 is None:
            # Cannot form triangle, push back? Source doesn't specify fallback.
            # Assuming valid configuration always allows this.
            break
            
        remaining.remove(p3)
        
        triangles.append((p1, p2, p3))
        # Clone with reversed normal
        triangles.append((p3, p2, p1))
        
    return triangles

def distance(edge_idx_a: int, edge_idx_b: int) -> float:
    # Calculate distance between midpoints of two edges
    # Assuming standard cube edge numbering and 0.5 grid
    # This is a placeholder for the actual coordinate calculation
    # [G] Standard MC edge numbering: 
    # 0:(0,0,0)-(1,0,0), 1:(1,0,0)-(1,1,0), 2:(1,1,0)-(0,1,0), 3:(0,1,0)-(0,0,0)
    # 4:(0,0,0)-(0,0,1), 5:(1,0,0)-(1,0,1), 6:(1,1,0)-(1,1,1), 7:(0,1,0)-(0,1,1)
    # 8:(0,0,1)-(1,0,1), 9:(1,0,1)-(1,1,1), 10:(1,1,1)-(0,1,1), 11:(0,1,1)-(0,0,1)
    # Midpoints:
    midpoints = [
        (0.5, 0, 0), (1, 0.5, 0), (0.5, 1, 0), (0, 0.5, 0),
        (0, 0, 0.5), (1, 0, 0.5), (1, 1, 0.5), (0, 1, 0.5),
        (0.5, 0, 1), (1, 0.5, 1), (0.5, 1, 1), (0, 1, 1)
    ]
    p1 = midpoints[edge_idx_a]
    p2 = midpoints[edge_idx_b]
    return ((p1[0]-p2[0])**2 + (p1[1]-p2[1])**2 + (p1[2]-p2[2])**2)**0.5
```

**Edge Cases and Pitfalls:**
*   **Triangle Intersection:** Explicit rules must be applied to avoid intersecting triangles [S1, unofficial].
*   **Duplication:** Triangles must not be duplicated [S1, unofficial].
*   **Coverage:** Triangles must cover enabled corner points by checking normal alignment with adjacent corner distances [S1, unofficial].
*   **Precision:** The method assumes vertex positions are perfect multiples of 0.5 [S1, unofficial].

## Subtopics

### Vertex-Adjacent Edge Point Method
A procedural algorithm that generates the 256-case Marching Cubes triangle table by placing edge points adjacent to active vertices and resolving overlaps [S1, unofficial]. It initializes with cube vertices and their 12 adjacent edge points, places three edge points for each active vertex, removes edge points lying on edges connecting two active vertices, and uses the remaining points to construct the triangle array [S1, unofficial]. It iterates through all 256 vertex combinations to populate the full table [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The method involves iterating through 256 cases and performing nearest-neighbor searches, which is computationally expensive compared to direct lookup.
*   **Memory Complexity:** Not given in the sources. Requires storage for the 12-element edge array and lists for triangle generation.
*   **Strengths:** Does not require pre-computed or manually curated triangle tables; derives topology procedurally [S1, unofficial].
*   **Limitations:** Complex rules for avoiding intersection and duplication; potentially inefficient for real-time rendering due to the brute-force nature [S1, unofficial].
*   **When to use:** When generating a custom or non-standard Marching Cubes table is required, or for educational purposes to understand the geometric basis of the algorithm [S1, unofficial].
*   **When not to use:** For performance-critical applications where standard lookup tables are available [S1, unofficial].

## Background
The method was described by Kino Bacaltos in a 2019 Stack Overflow answer as a way to generate the Marching Cubes triangle table procedurally [S1, unofficial]. It is presented as an alternative to using pre-defined tables, leveraging geometric proximity to determine triangle connectivity [S1, unofficial].

## Sources
[S1] How is the marching cubes triangle table generated?, https://stackoverflow.com/questions/44755032/how-is-the-marching-cubes-triangle-table-generated, fetched 2026-10-27 (unofficial)
