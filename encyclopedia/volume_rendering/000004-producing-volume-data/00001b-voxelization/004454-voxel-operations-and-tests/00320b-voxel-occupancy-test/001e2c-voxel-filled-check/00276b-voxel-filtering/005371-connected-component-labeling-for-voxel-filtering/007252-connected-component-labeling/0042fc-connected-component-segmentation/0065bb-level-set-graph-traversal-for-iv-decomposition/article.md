# Level-Set Graph Traversal for IV Decomposition

Also called: IV decomposition order extraction, Level-Set Graph Traversal for IV Decomposition Order, Systematic traversal of level-set graph

## Overview
This method determines the inclusion relationships and decomposition order of Isosurface Volumes (IVs) by traversing a level-set graph from the exterior surface inward [S1, book]. It solves the problem of identifying which nested feature subvolumes must be removed first to visualize interior structures without occlusion. It is distinguished from sibling methods like [Connected Component Labeling via Union-Find](14247-connected-component-labeling-via-union-find.md) by its explicit construction of a topological graph to manage inclusion relationships, relying on specific component algorithms for component counting, genus identification, and clustering [S1, book].

## How it works
**Inputs:** A level-set graph representing the topological skeleton of the volume, where nodes are critical points and links represent isosurface transitions. The graph is rooted at the exterior surface.
**Outputs:** A sequence of links in `Qorder` representing the IV decomposition order from exterior to interior.

**Data Structures:**
1.  **Level-Set Graph:** A directed graph where nodes $n$ represent critical points and edges (links) $L$ represent transitions between critical points. Each link has an associated scalar value range.
2.  **Queues:**
    *   `Qsolid`: FIFO queue for links corresponding to solid isosurface transitions.
    *   `Qhollow`: FIFO queue for links corresponding to hollow isosurface transitions.
    *   `Qorder`: List/Queue to store the final decomposition order.

**Definitions:**
*   **Isosurface Solid:** An isosurface that expands as the scalar field value decreases [S1, book].
*   **Isosurface Hollow:** An isosurface that shrinks as the scalar field value decreases [S1, book].
*   **Critical Point Index:** Classified by the number of negative eigenvalues of the Hessian matrix: $C3$ (maxima), $C2$ (saddles), $C1$ (saddles), $C0$ (minima) [S1, book].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize `Qsolid` and `Qhollow` as empty FIFO queues.
    *   Initialize `Qorder` as an empty list.
    *   Identify the link $L_0$ connected to the exterior surface node.
    *   Determine if $L_0$ is solid or hollow. Since it connects to the exterior, it is typically classified as solid [S1, book].
    *   Enqueue $L_0$ into `Qsolid`.

2.  **Process Solid Links (Upward Traversal):**
    *   While `Qsolid` is not empty:
        *   Dequeue a link $L$ from `Qsolid`.
        *   Append $L$ to `Qorder`.
        *   Let $n_{next}$ be the critical point node at the end of link $L$ (in the direction of decreasing scalar value, i.e., "upward" relative to the exterior root).
        *   Classify the transition type at node $n_{next}$ based on its critical point index ($C1$ or $C2$) and spatial configuration (referencing the classification in Fig. 3 of [S1, book]).
        *   Identify all outgoing links from $n_{next}$ to deeper critical points.
        *   For each outgoing link $L_{new}$:
            *   If the transition corresponds to a **solid** isosurface (expands as scalar decreases), enqueue $L_{new}$ into `Qsolid`.
            *   If the transition corresponds to a **hollow** isosurface (shrinks as scalar decreases), enqueue $L_{new}$ into `Qhollow`.
            *   *Note:* IV inclusions occur only in specific transition paths (row B in Fig. 3 of [S1, book]) for $C2$ and $C1$ critical points. Other paths (A, C, D) do not result in IV inclusions [S1, book].

3.  **Process Hollow Links (Downward Traversal):**
    *   While `Qhollow` is not empty:
        *   Dequeue a link $L$ from `Qhollow`.
        *   Append $L$ to `Qorder`.
        *   Let $n_{next}$ be the critical point node at the end of link $L$ (in the direction of decreasing scalar value, i.e., "downward" from the perspective of the hollow link's origin).
        *   Classify the transition type at node $n_{next}$.
        *   Identify all outgoing links from $n_{next}$.
        *   For each outgoing link $L_{new}$:
            *   If the transition corresponds to a **solid** isosurface, enqueue $L_{new}$ into `Qsolid`.
            *   If the transition corresponds to a **hollow** isosurface, enqueue $L_{new}$ into `Qhollow`.

4.  **Termination:**
    *   Repeat steps 2 and 3 until both `Qsolid` and `Qhollow` are empty.
    *   The sequence of links in `Qorder` constitutes the IV decomposition order.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompose_iv(level_set_graph, exterior_node):
    """
    Determines IV decomposition order by traversing the level-set graph.
    
    Args:
        level_set_graph: Graph structure with nodes (critical points) and links (transitions).
                         Each link has attributes: source_node, target_node, is_solid (bool).
                         Each node has attributes: critical_type (C0, C1, C2, C3), outgoing_links.
        exterior_node: The node representing the exterior surface.
        
    Returns:
        Qorder: List of links in decomposition order.
    """
    Qsolid = []  # FIFO queue for solid links
    Qhollow = [] # FIFO queue for hollow links
    Qorder = []  # List for decomposition order
    
    # Step 1: Initialization
    # Find the link connected to the exterior surface. 
    # In the level-set graph, this is typically the link starting from the exterior node 
    # or the link leading to the first critical point from the outside.
    # Assuming the graph is structured such that we start with the first link L0.
    # L0 is solid because it is connected to the exterior surface [S1, book].
    L0 = get_exterior_link(level_set_graph, exterior_node)
    
    if L0.is_solid:
        Qsolid.append(L0)
    else:
        Qhollow.append(L0)
        
    # Step 2 & 3: Process queues
    while Qsolid or Qhollow:
        # Process Solid Links (Upward Traversal)
        while Qsolid:
            L = Qsolid.pop(0) # Dequeue from front
            Qorder.append(L)
            
            # Get the next critical point node in the direction of traversal
            # For solid links, traversal is "upward" (exterior to interior, decreasing scalar)
            next_node = L.target_node
            
            # Classify transition and enqueue new links
            # The classification depends on the critical point type (C1, C2) and spatial config
            # See Fig 3 in [S1, book] for specific rules.
            # For simplicity, we assume the graph links are pre-classified as solid/hollow.
            for new_link in next_node.outgoing_links:
                if new_link.is_solid:
                    Qsolid.append(new_link)
                else:
                    Qhollow.append(new_link)
                    
        # Process Hollow Links (Downward Traversal)
        while Qhollow:
            L = Qhollow.pop(0) # Dequeue from front
            Qorder.append(L)
            
            # Get the next critical point node
            # For hollow links, traversal is "downward"
            next_node = L.target_node
            
            for new_link in next_node.outgoing_links:
                if new_link.is_solid:
                    Qsolid.append(new_link)
                else:
                    Qhollow.append(new_link)
                    
    return Qorder

def get_exterior_link(graph, exterior_node):
    """
    Helper to find the initial link L0 connected to the exterior.
    Implementation detail: [G]
    """
    # In a typical level-set graph, the exterior is represented by a specific node or boundary.
    # The link L0 is the one connecting the exterior to the first critical point.
    # This function would return the appropriate link object.
    return graph.links[0] # Placeholder for actual graph traversal logic
```

**Edge Cases and Pitfalls:**
*   **IV Inclusions:** IV inclusions occur only in specific transition paths (row B in Fig. 3 of [S1, book]) for both $C2$ and $C1$ critical points. Other paths (A, C, D) do not result in IV inclusions [S1, book].
*   **Critical Point Classification:** Correctly identifying the critical point index ($C3, C2, C1, C0$) is essential. The index is the number of negative eigenvalues of the Hessian matrix [S1, book].
*   **Queue Order:** The use of FIFO queues ensures that the traversal respects the topological order. Mixing up the order of processing `Qsolid` and `Qhollow` will result in an incorrect decomposition order.

## Subtopics

### [Interval Volume Decomposer](19455-interval-volume-decomposer.md)
The Interval Volume Decomposer is an algorithm that decomposes a scalar volume into feature subvolumes, specifically interval volumes (IVs), by tracking the topological transitions of isosurfaces. It constructs a level-set graph representing the topological skeleton of the volume and enables systematic decomposition from the exterior to the interior by traversing this graph. It is distinguished from sibling methods by its explicit construction of a topological graph to manage inclusion relationships, relying on specific component algorithms for component counting, genus identification, and clustering.

### [Automatic IV Decomposition with Optimal Viewpoint Preview](22857-automatic-iv-decomposition-with-optimal-viewpoint-preview.md)
This method generates an interactive animation scenario that peels off Interval Volumes (IVs) from the exterior to the interior of a dataset according to a predefined decomposition order, while simultaneously optimizing and smoothly interpolating the camera viewpoint to minimize occlusions between the remaining feature subvolumes. It distinguishes itself from static decomposition methods by coupling the topological removal of nested volumes with a dynamic, entropy-minimized camera path that tracks optimal viewing angles for each intermediate state of the decomposition.

## Operating characteristics
**Time Complexity:** Linear with respect to the number of links in the level-set graph, as each link is enqueued and dequeued exactly once.
**Memory Complexity:** Linear with respect to the number of links, as the queues store references to each link.
**Strengths:** Provides a systematic and topologically correct decomposition order. Explicitly handles nested structures (IV inclusions) through the level-set graph.
**Limitations:** Requires the prior construction of the level-set graph, which can be computationally expensive for large volumes. Relies on accurate critical point detection and classification.
**When to use:** When the goal is to visualize nested volume features by peeling them away from the exterior, and when the topological structure of the volume is important.
**When not to use:** When real-time performance is critical and the overhead of graph construction is prohibitive. When the volume does not have clear nested structures.

## Background
The method was introduced by S. Takahashi, I. Fujishiro, Y. Takeshima, and C. Bi in the chapter "Previewing Volume Decomposition Through Optimal Viewpoints" [S1, book]. It is part of a broader effort to improve the visualization of complex volume data by decomposing it into manageable feature subvolumes. The approach builds on the concept of level-set graphs and critical point theory in scalar fields.

## Sources
[S1] Previewing Volume Decomposition Through Optimal Viewpoints, https://doi.org/10.4230/dfu.vol2.sciviz.2011.346, fetched 2026-10-26 (book)
