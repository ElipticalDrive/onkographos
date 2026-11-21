# Level-Set Graph Traversal for IV Decomposition

Also called: IV decomposition order extraction, Level-Set Graph Traversal for IV Decomposition Order, Systematic traversal of level-set graph

## Overview
Level-Set Graph Traversal for IV Decomposition is a method to determine the inclusion relationships and decomposition order of Isosurface Volumes (IVs) by traversing a level-set graph from the exterior surface inward [S1, book]. It solves the problem of systematically peeling a volume into nested feature subvolumes, enabling applications such as optimal viewpoint selection for volume previewing [S1, book]. What sets it apart from sibling methods like [Connected Component Labeling via Union-Find](14247-connected-component-labeling-via-union-find.md) is its reliance on topological classification of critical points (specifically saddles) to distinguish between "solid" and "hollow" transitions, using dual queues to manage the traversal direction (upward vs. downward) relative to the scalar field [S1, book].

## How it works
**Inputs and Outputs**
*   **Input:** A level-set graph representing the topological skeleton of a volume. The graph consists of nodes (critical points) and links (feature subvolumes/IVs). Each link is associated with a scalar field range bounded by critical isosurfaces.
*   **Output:** A sequence of links in `Qorder` representing the decomposition order from outside to inside, along with the inclusion relationships between IVs [S1, book].

**Data Structures**
1.  **Level-Set Graph:** A directed graph where nodes $n$ represent critical points and links $L$ represent IVs.
    *   **Node Attributes:** Critical point type (C3, C2, C1, C0) and transition classification (e.g., (B)-C1).
    *   **Link Attributes:** Scalar value range, connection to parent/child nodes, and type (Solid or Hollow).
2.  **Queues:**
    *   `Qsolid`: FIFO queue for links to be traversed in the upward direction (exterior inward, lower scalar to higher scalar relative to critical point context, or specifically from exterior inward as per source description of "upward" tracing).
    *   `Qhollow`: FIFO queue for links to be traversed in the downward direction.
    *   `Qorder`: List/Queue to store the final decomposition order of links.

**Definitions**
*   **Isosurface Solid:** An isosurface that expands as the scalar field value decreases [S1, book].
*   **Isosurface Hollow:** An isosurface that shrinks as the scalar field value decreases [S1, book].
*   **Critical Point Index:** Classified by the number of negative eigenvalues of the Hessian matrix:
    *   C3: Maxima (3 negative eigenvalues)
    *   C2: Saddles (2 negative eigenvalues)
    *   C1: Saddles (1 negative eigenvalue)
    *   C0: Minima (0 negative eigenvalues) [S1, book]

**Algorithm Steps**
1.  **Initialization:**
    *   Initialize `Qsolid` as an empty FIFO queue.
    *   Initialize `Qhollow` as an empty FIFO queue.
    *   Initialize `Qorder` as an empty list.
    *   Identify the link $L_0$ connected to the exterior surface of the volume.
    *   Add $L_0$ to `Qsolid`. [S1, book]

2.  **Processing Solid Links (Upward Traversal):**
    *   While `Qsolid` is not empty:
        *   Dequeue a link $L$ from `Qsolid`.
        *   Append $L$ to `Qorder`.
        *   Identify the critical point node $n_{next}$ at the end of $L$ (in the direction of traversal).
        *   Classify the transition type at $n_{next}$ (e.g., (B)-C1, (B)-C2, etc.) based on the spatial configuration of isosurface transitions [S1, book].
        *   For each new link $L_{new}$ emerging from $n_{next}$:
            *   If the transition corresponds to a **Solid** isosurface (expands as scalar decreases), enqueue $L_{new}$ into `Qsolid`.
            *   If the transition corresponds to a **Hollow** isosurface (shrinks as scalar decreases), enqueue $L_{new}$ into `Qhollow`.
        *   *Note:* IV inclusions occur only in specific transition paths. Specifically, inclusions occur only in the two transition paths of row (B) in the classification diagram for both C2 and C1 critical points. Other paths (A, C, D) do not result in IV inclusions [S1, book].

3.  **Processing Hollow Links (Downward Traversal):**
    *   While `Qhollow` is not empty:
        *   Dequeue a link $L$ from `Qhollow`.
        *   Append $L$ to `Qorder`.
        *   Identify the critical point node $n_{next}$ at the end of $L$ (in the direction of traversal).
        *   Classify the transition type at $n_{next}$.
        *   For each new link $L_{new}$ emerging from $n_{next}$:
            *   If the transition corresponds to a **Solid** isosurface, enqueue $L_{new}$ into `Qsolid`.
            *   If the transition corresponds to a **Hollow** isosurface, enqueue $L_{new}$ into `Qhollow`.

4.  **Termination:**
    *   Repeat steps 2 and 3 until both `Qsolid` and `Qhollow` are empty.
    *   The final decomposition order is the sequence of links in `Qorder` [S1, book].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def level_set_graph_traversal(level_set_graph):
    """
    Determines the IV decomposition order by traversing the level-set graph.
    
    Args:
        level_set_graph: A graph object containing nodes (critical points) and links (IVs).
                         Must have an 'exterior_link' attribute pointing to the link L0 
                         connected to the exterior surface.
                         
    Returns:
        Qorder: A list of links in decomposition order.
    """
    Qsolid = []  # FIFO queue for solid links
    Qhollow = [] # FIFO queue for hollow links
    Qorder = []  # List for decomposition order
    
    # Step 1: Initialization
    L0 = level_set_graph.exterior_link
    Qsolid.append(L0)
    
    while Qsolid or Qhollow:
        # Step 2: Process Solid Links (Upward Traversal)
        while Qsolid:
            L = Qsolid.pop(0) # Dequeue from front
            Qorder.append(L)
            
            # Identify the critical point node at the end of L
            # The direction of traversal is "upward" (exterior inward)
            n_next = L.get_end_node_in_traversal_direction()
            
            # Classify the transition type at n_next
            # Returns a classification code, e.g., 'B_C1', 'B_C2', 'A_C1', etc.
            transition_type = classify_critical_point(n_next)
            
            # Get outgoing links from n_next
            outgoing_links = n_next.get_outgoing_links()
            
            for L_new in outgoing_links:
                # Determine if the link corresponds to a Solid or Hollow isosurface
                # based on the transition type and spatial configuration
                is_solid = is_solid_transition(transition_type, L_new)
                
                if is_solid:
                    Qsolid.append(L_new)
                else:
                    Qhollow.append(L_new)
        
        # Step 3: Process Hollow Links (Downward Traversal)
        while Qhollow:
            L = Qhollow.pop(0) # Dequeue from front
            Qorder.append(L)
            
            # Identify the critical point node at the end of L
            # The direction of traversal is "downward"
            n_next = L.get_end_node_in_traversal_direction()
            
            # Classify the transition type at n_next
            transition_type = classify_critical_point(n_next)
            
            # Get outgoing links from n_next
            outgoing_links = n_next.get_outgoing_links()
            
            for L_new in outgoing_links:
                # Determine if the link corresponds to a Solid or Hollow isosurface
                is_solid = is_solid_transition(transition_type, L_new)
                
                if is_solid:
                    Qsolid.append(L_new)
                else:
                    Qhollow.append(L_new)
                    
    return Qorder

def classify_critical_point(node):
    """
    Classifies the critical point node based on its type and spatial configuration.
    Returns a string code like 'B_C1', 'B_C2', 'A_C1', etc.
    """
    # Implementation depends on the specific topological volume skeletonization algorithm
    # Refer to Takahashi et al. [31] for practical implementation details.
    # This function identifies if the node is a C1 or C2 saddle and which 
    # transition path (A, B, C, D) it follows.
    pass

def is_solid_transition(transition_type, link):
    """
    Determines if a link is Solid or Hollow based on the transition type.
    Solid: Expands as scalar field value decreases.
    Hollow: Shrinks as scalar field value decreases.
    
    According to [S1, book], IV inclusions (and specific solid/hollow assignments)
    occur only in transition paths of row (B) for C2 and C1 critical points.
    """
    # This logic is derived from Fig. 3 in [S1, book].
    # The exact mapping of transition_type to solid/hollow is not explicitly 
    # detailed in the text beyond the visual classification in Fig. 3.
    # A generic implementation would check the link's scalar range and the 
    # node's critical point index.
    pass
```

**Parameters**
*   The algorithm relies on three component algorithms for its construction phase (not the traversal itself):
    1.  Carr et al. [6] for tracking the number of isosurface connected components.
    2.  Pascucci et al. [22] for identifying the genus of each isosurface component.
    3.  Takahashi et al. [31] for IV clustering and topological volume skeletonization [S1, book].

**Edge Cases and Pitfalls**
*   **IV Inclusions:** IV inclusions occur *only* in the two transition paths of row (B) for both C2 and C1 critical points. Other paths (A, C, D) do not result in IV inclusions [S1, book]. Failure to correctly classify these transitions will lead to incorrect inclusion relationships.
*   **Critical Point Classification:** Accurate classification of C1 and C2 saddles into their specific spatial transition types (A, B, C, D) is crucial. This requires robust topological volume skeletonization [S1, book].
*   **Traversal Direction:** The distinction between "upward" (solid) and "downward" (hollow) traversal must be strictly maintained. Mixing these directions will corrupt the decomposition order [S1, book].

## Subtopics

### Interval Volume Decomposer
An interface and algorithm that decomposes an entire volume into feature subvolumes (interval volumes) by tracking the topological transitions of isosurfaces with respect to the scalar field, constructing a level-set graph, and enabling systematic decomposition from outside to inside [S1, book]. It identifies each link of the level-set graph with a feature subvolume (IV) bounded by critical isosurfaces and retrieves inclusion relationships by systematically traversing the graph [S1, book]. It relies on the same three component algorithms as the traversal method: Carr et al. [6], Pascucci et al. [22], and Takahashi et al. [31] [S1, book].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The complexity depends on the size of the level-set graph and the number of critical points.
*   **Memory Complexity:** Not given in the sources. Requires storage for the level-set graph and two FIFO queues.
*   **Strengths:** Enables systematic decomposition of volumes from outside to inside, which is useful for viewpoint selection and previewing inner structures [S1, book].
*   **Limitations:** Requires accurate construction of the level-set graph, which depends on robust topological skeletonization algorithms [S1, book].
*   **When to use:** When the goal is to decompose a volume into nested feature subvolumes for visualization or analysis, particularly for viewpoint selection [S1, book].
*   **When not to use:** When only connected component labeling is needed without topological inclusion relationships [S1, book].

## Background
The method was developed by S. Takahashi, I. Fujishiro, Y. Takeshima, and C. Bi. It is described in the book chapter "Previewing Volume Decomposition Through Optimal Viewpoints" [S1, book]. The concept of Interval Volumes (IV) was formulated by Fujishiro et al. [10] and Guo [13] as a generalization of the isosurface [S1, book]. The level-set graph construction relies on topological volume skeletonization [31], which combines algorithms by Carr et al. [6], Pascucci et al. [22], and Takahashi et al. [31] [S1, book].

## Sources
[S1] Previewing Volume Decomposition Through Optimal Viewpoints, https://doi.org/10.4230/dfu.vol2.sciviz.2011.346, fetched 2026-10-26 (book)
