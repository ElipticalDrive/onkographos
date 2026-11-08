# Sequential Graph-cut Segmentation

Also called: Multi-material Graph-cut via Iterative Binary Segmentation

## Overview
Sequential Graph-cut Segmentation is a multi-material classification method that decomposes an $n$-material segmentation problem into $n-1$ binary Graph-cut iterations [S1]. It solves the NP-hardness of direct multi-way graph partitioning by classifying materials one by one, starting from the lowest CT value [S1]. This approach ensures that boundaries are consistent with isovalue thresholds and prevents the misclassification of lower-density materials at interfaces with higher-density materials [S1]. It builds on [Graph Construction for Multi-material Segmentation](8290-graph-construction-for-multi-material-segmentation.md) by defining the specific iterative strategy and terminal node assignments.

## How it works

### Inputs and Outputs
**Inputs:**
*   Volumetric data $V$ with CT values.
*   $n$: Total number of materials to classify (including background).
*   $M$: An ordered set of all $n$ materials $\{0, 1, \dots, n-1\}$, sorted by increasing CT value, where $m_0 < m_1 < \dots < m_{n-1}$ [S1]. $m_0$ is the background CT value.
*   $X$: The set of "doubt voxels" (unclassified voxels), initially containing all voxels not classified by a preceding restricted region growing step [S1].

**Outputs:**
*   A classification map assigning each voxel in $X$ to one of the materials $m_1, \dots, m_{n-1}$, leaving only background or unclassified voxels if any remain.

### Data Structures
*   **Graph $G=(V_{graph}, E)$**:
    *   $V_{graph}$: Nodes representing the voxels in the current set of doubt voxels $X$.
    *   $E$: Edges induced by voxel connectivity (6-, 18-, or 26-neighborhood) [S1].
    *   **Terminal Nodes**:
        *   $s$: Represents the current target material $m_i$ [S1].
        *   $t$: Represents the collection of all other materials (unclassified set $\bar{M}$) [S1].
    *   **Edge Weights**:
        *   $w_{cut}$: Weight for edges crossing the cut (assigned to edges connecting a node to the terminal it does *not* belong to, or based on boundary constraints).
        *   $w_{join}$: Weight for edges not crossing the cut (assigned to edges connecting a node to the terminal it *does* belong to, or based on region constraints).
        *   $\tau = w_{cut} / w_{join}$: Cut-ratio parameter, typically $0 < \tau < 1$ [S1].

### Numbered Steps

1.  **Initialization**:
    *   Set $M = \{0, 1, \dots, n-1\}$ sorted by CT value [S1].
    *   Identify initial doubt voxels $X$ (e.g., via restricted region growing) [S1].
    *   Initialize $\bar{M} = M \setminus \{0\}$ (exclude background) [S1].

2.  **Iteration Loop**:
    While $\bar{M}$ is not empty:
    1.  **Select Material**: Choose the next material $m_i$ from $\bar{M}$ starting with the lowest CT value (index $i=1$ first) [S1].
    2.  **Define Terminals**:
        *   Set source terminal $s$ to represent material $m_i$ [S1].
        *   Set sink terminal $t$ to represent $\bar{M} \setminus \{m_i\}$ (all other unclassified materials) [S1].
    3.  **Construct Graph**:
        *   Create nodes for all voxels in $X$ [S1].
        *   Add edges based on neighborhood connectivity (6, 18, or 26) [S1].
        *   Assign edge weights $w(p,q)$ proportional to $|I_p - I_q|$ (absolute CT difference) for internal edges [S1].
        *   Assign weights to edges connecting voxels to terminals $s$ and $t$ based on hard/soft constraints (user-defined object/background, boundary, and region information) [S1]. Specifically, edges favoring $s$ get weight related to $w_{join}$, and edges favoring $t$ get weight related to $w_{cut}$ [S1].
    4.  **Minimize Energy**:
        *   Compute the minimum cut to minimize the energy function $E$ [S1]:
            $$ E = \sum_{p \in S, q \in T} w(p, q) = N_{cut} \cdot w_{cut} + N_{join} \cdot w_{join} $$
            where $S$ is the set of nodes connected to $s$, $T$ is the set of nodes connected to $t$, $N_{cut}$ is the number of edges between $S$ and $T$, and $N_{join}$ is the number of edges within $S$ or $T$ [S1].
        *   This is equivalent to minimizing:
            $$ E = w_{join} (\tau \cdot N_{cut} + N_{join}) $$
            where $\tau = w_{cut} / w_{join}$ [S1].
    5.  **Classify**:
        *   All doubt voxels in subgraph $S$ (connected to $s$) are classified as material $m_i$ [S1]. Let this set be $M_i$ [S1].
    6.  **Update**:
        *   Subtract classified voxels from the doubt set: $X \leftarrow X \setminus (S \cap X)$ [S1].
        *   Remove $m_i$ from the unclassified set: $\bar{M} \leftarrow \bar{M} \setminus \{m_i\}$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sequential_graph_cut_segmentation(volume, initial_doubt_voxels, materials):
    """
    volume: 3D array of CT values
    initial_doubt_voxels: set of voxel indices to be classified
    materials: list of material IDs sorted by increasing CT value, 
               excluding background (index 0). 
               materials[0] is the lowest CT material.
    """
    doubt_voxels = set(initial_doubt_voxels)
    unclassified_materials = list(materials) # Copy of materials list
    classification_map = {} # voxel_index -> material_id
    
    # materials are already sorted by CT value: m0 < m1 < ...
    
    while unclassified_materials:
        # Step 1: Select material i (lowest CT among unclassified)
        material_i = unclassified_materials[0]
        
        # Step 2: Define terminals
        # s represents material_i
        # t represents all other unclassified materials (unclassified_materials[1:])
        
        # Step 3: Construct Graph for doubt_voxels
        # Nodes: doubt_voxels
        # Edges: 6/18/26 neighborhood connections
        # Weights:
        #   Internal edges w(p,q) proportional to |volume[p] - volume[q]|
        #   Terminal edges: 
        #     - To s: weight based on similarity to material_i (w_join)
        #     - To t: weight based on dissimilarity or boundary constraints (w_cut)
        graph = build_graph(doubt_voxels, volume)
        assign_terminal_weights(graph, doubt_voxels, volume, material_i, unclassified_materials)
        
        # Step 4: Minimize Energy (Min-Cut)
        # E = N_cut * w_cut + N_join * w_join
        # Minimize E to find partition (S, T)
        source_node = get_source_terminal(graph)
        sink_node = get_sink_terminal(graph)
        partition_S, partition_T = min_cut(graph, source_node, sink_node)
        
        # Step 5: Classify voxels in S as material_i
        classified_voxels = partition_S # Voxels connected to source s
        
        for voxel in classified_voxels:
            classification_map[voxel] = material_i
            
        # Step 6: Update sets
        doubt_voxels = doubt_voxels - classified_voxels
        unclassified_materials.pop(0) # Remove material_i from list
        
    return classification_map

def assign_terminal_weights(graph, doubt_voxels, volume, material_i, unclassified_materials):
    """
    Assigns weights to edges connecting doubt_voxels to source (s) and sink (t).
    This is a placeholder for the specific hard/soft constraint logic described in [S1].
    The source represents material_i, sink represents others.
    """
    # [G] Implementation detail: 
    # Edges to s (material_i) should have high weight if voxel CT is close to material_i CT.
    # Edges to t (others) should have high weight if voxel CT is NOT close to material_i CT.
    # The ratio tau = w_cut / w_join controls the boundary smoothness vs. data fidelity.
    pass
```

### Parameters and Edge Cases
*   **Cut-ratio $\tau$**: The ratio $w_{cut}/w_{join}$ specifies the relative importance of cut vs. join weights. A reasonable cut requires $0 < \tau < 1$ [S1].
*   **Neighborhood**: 6-, 18-, or 26-neighborhood systems are used to define graph edges [S1].
*   **Ordering**: Materials must be processed in increasing order of CT value. Classifying lower CT materials first prevents misclassification at boundaries with higher CT materials (e.g., rubber before steel) [S1].
*   **Doubt Voxels**: The method relies on an initial classification (e.g., region growing) to reduce the graph size. Applying Graph-cut to the full volume is computationally expensive and may produce boundaries with incorrect topology [S1].

## Subtopics

### Voxel Classification via Graph-cut
A global segmentation method used within the sequential process to classify remaining unclassified (doubt) voxels at boundaries [S1]. It minimizes a cost function over binary variables defined by the graph structure [S1]. The cost function is based on CT value similarity and incorporates hard constraints (user-defined object/background) and soft constraints (boundary and region information) [S1]. Graph nodes are voxels of the target model, and edges are induced by 6-, 18-, or 26-neighborhood connectivity [S1].

## Operating characteristics
*   **Complexity**: The method avoids the NP-hard problem of multi-way graph cutting by repeating binary Graph-cut $n-1$ times [S1]. Each iteration is a standard binary min-cut, which is polynomial in time complexity relative to the number of doubt voxels [S1].
*   **Performance**: Constructing and cutting a full graph for large volumetric models is resource-intensive [S1]. Using a restricted region growing step to pre-classify the majority of voxels significantly reduces the number of doubt voxels and thus the graph size [S1].
*   **Strengths**: Ensures boundaries are consistent with isovalue thresholds and correctly handles material ordering (low to high CT) to prevent misclassification at interfaces [S1].
*   **Limitations**: The resulting boundaries are a global minimum of the defined cost function and may not always match the real parts' topological characteristics [S1].
*   **When to use**: Multi-material CT segmentation where material density ordering is known and boundaries need to respect isovalue thresholds.
*   **When not to use**: When real-time performance is critical and the graph size is too large, or when topological accuracy is more important than strict isovalue consistency.

## Background
The method was proposed in the context of extracting isosurfaces from multi-material CT volumetric data of mechanical parts [S1]. It addresses the limitations of direct Graph-cut application on large volumetric models, such as high computational cost and potential topological inaccuracies [S1]. The approach builds on foundational Graph-cut algorithms by Boykov and Funka-Lea [S1], Boykov and Jolly [S1], and Kolmogorov and Zabih [S1], adapting them for multi-material segmentation through iterative binary classification.

## Sources
[S1] Extraction of isosurfaces from multi-material CT volumetric data of mechanical parts, https://doi.org/10.1145/1364901.1364931, fetched 2026-11-01
